# Code change document  
## `Z_LOG_TCFA_REPROCESS` — Reuse NLP amount from `ZLOG_INT_DB` (success / max `SRNO`)

| Document control | |
|------------------|---|
| **Primary function module** | `Z_LOG_TCFA_REPROCESS` |
| **New function module** | `Z_LOG_TCFA_GET_NLP_INT_DB` |
| **Change label** | TCFA reprocess: short-circuit freight / liability / notional call when prior success exists in `ZLOG_INT_DB` |
| **Version** | 1.0 |
| **Reference extract (this repo)** | `FM - Z_LOG_TCFA_REPROCESS.txt` |

---

## 1. Summary

Add a **read-only** function module **`Z_LOG_TCFA_GET_NLP_INT_DB`** that reads **`ZLOG_INT_DB`** for a list of shipments and returns **`NLP_AMOUNT`** (and **`SRNO`**) from the **latest successful** notional-limit row per shipment: **`FLAG = 'S'`**, **`METHOD_NAME = UPDATE_NOTIONAL_LIMIT`**, **maximum `SRNO`**.

Extend **`Z_LOG_TCFA_REPROCESS`** to call this FM **after** **`lt_trstlmnt`** is finalized and **`lw_amount_param`** is loaded, and **before** the FCPL / freight preparation block (before **`CLEAR lt_strlmt_fcpl`** / **`LOOP AT lt_trstlmnt ASSIGNING`**, ~line 553 in the reference extract). For each hit that passes the **same ceiling rule** as the existing notional loop, perform **`UPDATE ZTRSTLMNT`** (unchanged scope: **`WHERE mandt`**, **`WHERE tknum`**, set **`nlp_amt`**, **`nlp_acks = '02'`** using existing **`lc_02`**), **`COMMIT` / `ROLLBACK`** and **`MODIFY lt_trstlmnt`** in line with current patterns, then **`DELETE`** all **`lt_trstlmnt`** rows for that **`tknum`** so freight simulation, liability build, and **`update_notional_limit` / `Z_RNM_VENDOR_NLPUPDATE_WS`** **do not run** for that shipment.

If there is **no** qualifying log row, the amount **fails** the ceiling check, **`ENQUEUE_EZZTRSTLMNT`** fails, or **`UPDATE`** fails, behaviour for that shipment follows the **existing** path (no change).

---

## 2. Business rules

| # | Rule |
|---|------|
| 1 | **`ZLOG_INT_DB`**: **`SHIPMENT_NO`** = shipment, **`FLAG = 'S'`**, **`METHOD_NAME = 'UPDATE_NOTIONAL_LIMIT'`** (same value as **`lc_method`** in `Z_LOG_TCFA_REPROCESS`). |
| 2 | Several success rows: take the row with **maximum `SRNO`** per **`SHIPMENT_NO`**. |
| 3 | **Ceiling:** Reuse only if **`NLP_AMOUNT`** is **not** above the limit using the **same** test as the notional loop: **do not reuse** when **`lw_nlp_hit-nlp_amount GT lw_amount_param-amount`**. There is **no** `IS NOT INITIAL` guard on **`lw_amount_param-amount`** in existing code; when the customizing row is missing, **`lw_amount_param`** is **`CLEAR`**, so **`amount`** is **initial** and **`GT`** compares as **0** (standard behaviour for the currency component). That matches **`lw_attr-amount GT lw_amount_param-amount`** / **`CONTINUE`** at ~1191. |
| 4 | On successful reuse: **`nlp_acks = '02'`**; **no** **`Z_SCE_SHIP_FREIGHT_CAL_QWIK`**, no **`lt_libility_new`** / **`lt_libility`** path, no notional RFC/WS for that **`tknum`**. |
| 5 | **`UPDATE ZTRSTLMNT`**, enqueue, dequeue, commit, rollback, **`MODIFY lt_trstlmnt`**: **only** in **`Z_LOG_TCFA_REPROCESS`**. The new FM performs **SELECT** only. |
| 6 | **`ENQUEUE_EZZTRSTLMNT`** fails: **do not** **`DELETE`** from **`lt_trstlmnt`**; do not update; **standard** flow for that **`tknum`**. |
| 7 | **`UPDATE`** fails: set **`e_error`**, **`ROLLBACK`**, **do not** **`DELETE`** from **`lt_trstlmnt`** (shipment remains for standard processing or retry, per same philosophy as failed update in the notional block). |
| 8 | New FM **`Z_LOG_TCFA_GET_NLP_INT_DB`** in the **same function group** as **`Z_LOG_TCFA_REPROCESS`**. |

---

## 3. Objects touched

| Object type | Name | Change |
|-------------|------|--------|
| FU / FM | **`Z_LOG_TCFA_GET_NLP_INT_DB`** | **New** — read `ZLOG_INT_DB`, export at most one line per shipment. |
| FU / FM | **`Z_LOG_TCFA_REPROCESS`** | Call new FM; apply updates; **`DELETE`** from **`lt_trstlmnt`** when reuse completes successfully. |
| FG | *(existing FG for TCFA reprocess)* | New FM + include changes. |
| DDIC (optional) | **`ZST_TCFA_NLP_INT_DB`**, **`ZTT_TCFA_NLP_INT_DB`** | Line type + table type for **`ET_RESULT`**; alternatively define equivalent **`TYPES`** in the function group top include only. |

---

## 4. Call site in `Z_LOG_TCFA_REPROCESS`

**File:** `FM - Z_LOG_TCFA_REPROCESS.txt` (adjust line numbers in SE37 if your copy differs).

| Step | Approx. location | Description |
|------|------------------|-------------|
| 1 | After **`SELECT SINGLE ... zlog_stlmntvr`** into **`lw_amount_param`** (~520–528) and after **`lt_trstlmnt`** is confirmed **not** initial (~489–492). | **`lw_amount_param`** must be loaded before ceiling evaluation. |
| 2 | **Immediately before** **`CLEAR: lt_strlmt_fcpl.`** and **`LOOP AT lt_trstlmnt ASSIGNING`** (~552–555). | Ensures shipments resolved from the log **do not** enter FCPL build, **`Z_SCE_TCFA_R1_RATE`**, VTTK/VTTP/LIPS preparation, or the **`LOOP AT lt_trstlmnt`** (~718) that fills **`lt_libility_new`**. |

---

## 5. New function module `Z_LOG_TCFA_GET_NLP_INT_DB`

### 5.1 Interface (SE37)

```text
FUNCTION z_log_tcfa_get_nlp_int_db.
*"----------------------------------------------------------------------
*"*"Local Interface:
*"  IMPORTING
*"     VALUE(IT_TKNUM) TYPE  ROIO_T_TKNUM
*"  EXPORTING
*"     VALUE(ET_RESULT) TYPE  ZTT_TCFA_NLP_INT_DB
*"----------------------------------------------------------------------
```

**`ET_RESULT`** line type (DDIC **`ZST_TCFA_NLP_INT_DB`** or FG-local equivalent):

| Component | Type reference |
|-----------|----------------|
| `TKNUM` | `TKNUM` |
| `NLP_AMOUNT` | `ZLOG_INT_DB-NLP_AMOUNT` / `ZLIBTY_AMT` |
| `SRNO` | `ZLOG_INT_DB-SRNO` |

**Contract:** At most **one** row per **`TKNUM`**. Omit shipments with no qualifying data.

### 5.2 Constants

```abap
CONSTANTS: lc_method TYPE zmethodname VALUE 'UPDATE_NOTIONAL_LIMIT',
           lc_flag_s TYPE char1       VALUE 'S'.
```

### 5.3 Implementation (NetWeaver 7.31)

- Declare all **`DATA`** / **`FIELD-SYMBOLS`** at the start of the FM; no inline declarations.
- **`IT_TKNUM`**: caller supplies **sorted** values, **no initial** `TABLE_LINE`, **adjacent duplicates removed**.
- **`SELECT`** explicit fields only; **`CLIENT SPECIFIED`** on **`ZLOG_INT_DB`** consistent with **`Z_LOG_TCFA_REPROCESS`**.
- **`FOR ALL ENTRIES`** on **`IT_TKNUM`**; **`WHERE mandt = sy-mandt`**, **`shipment_no = it_tknum-table_line`**, **`flag = lc_flag_s`**, **`method_name = lc_method`**.
- If **`sy-subrc <> 0`**, **`RETURN`** with **`ET_RESULT`** initial.
- **`SORT`** result by **`shipment_no`** descending **`srno`**; keep first row per **`shipment_no`** (loop with **`lw_prev_tknum`** or **`DELETE ADJACENT DUPLICATES`** **`COMPARING shipment_no`** after sort).
- No **`UPDATE`**, no **`ENQUEUE`**, no **`COMMIT`**.

### 5.4 SE37 documentation

Short text: e.g. **TCFA: read NLP amount from integration log (success).**  
Long text: filters, max-**`SRNO`**, and that **ceiling and database updates are performed by the caller**.

---

## 6. Changes in `Z_LOG_TCFA_REPROCESS`

### 6.1 Additional data declarations

Near existing **`DATA`** for **`lt_trstlmnt`** / processing, add (use exact types from your system for temp copy of **`lt_trstlmnt`**):

```abap
DATA: lt_tknum_nlp TYPE roio_t_tknum,
      lw_tknum_nlp TYPE LINE OF roio_t_tknum,
      lt_trst_temp TYPE ... ,           " same line type as lt_trstlmnt
      lw_trst_temp TYPE ... ,
      lt_nlp_hit   TYPE ztt_tcfa_nlp_int_db,
      lw_nlp_hit   TYPE zst_tcfa_nlp_int_db,
      lv_update_ok TYPE c.
```

### 6.2 Build `IT_TKNUM` and call new FM

Insert at call site §4:

```abap
REFRESH lt_tknum_nlp.
lt_trst_temp[] = lt_trstlmnt[].
SORT lt_trst_temp BY tknum.
DELETE lt_trst_temp WHERE tknum IS INITIAL.
DELETE ADJACENT DUPLICATES FROM lt_trst_temp COMPARING tknum.

LOOP AT lt_trst_temp INTO lw_trst_temp.
  CLEAR lw_tknum_nlp.
  lw_tknum_nlp-table_line = lw_trst_temp-tknum.
  APPEND lw_tknum_nlp TO lt_tknum_nlp.
ENDLOOP.

IF lt_tknum_nlp IS NOT INITIAL.
  CALL FUNCTION 'Z_LOG_TCFA_GET_NLP_INT_DB'
    EXPORTING
      it_tknum = lt_tknum_nlp
    IMPORTING
      et_result = lt_nlp_hit.
  IF sy-subrc <> 0.
    CLEAR lt_nlp_hit[].
  ENDIF.
ENDIF.
```

### 6.3 Apply reuse (ceiling, enqueue, update, memory, delete)

```abap
LOOP AT lt_nlp_hit INTO lw_nlp_hit.

  IF lw_nlp_hit-nlp_amount GT lw_amount_param-amount.
    CONTINUE.
  ENDIF.

  CLEAR lv_update_ok.

  CALL FUNCTION 'ENQUEUE_EZZTRSTLMNT'
    EXPORTING
      mandt = sy-mandt
      tknum = lw_nlp_hit-tknum
    EXCEPTIONS
      foreign_lock   = 1
      system_failure = 2
      OTHERS         = 3.

  IF sy-subrc <> 0.
    CONTINUE.
  ENDIF.

  UPDATE ztrstlmnt CLIENT SPECIFIED
    SET nlp_amt  = lw_nlp_hit-nlp_amount
        nlp_acks = lc_02
    WHERE mandt = sy-mandt
      AND tknum = lw_nlp_hit-tknum.

  IF sy-subrc = 0.
    COMMIT WORK AND WAIT.
    lv_update_ok = 'X'.
  ELSE.
    e_error = abap_true.
    ROLLBACK WORK.
  ENDIF.

  CALL FUNCTION 'DEQUEUE_EZZTRSTLMNT'
    EXPORTING
      mandt = sy-mandt
      tknum = lw_nlp_hit-tknum.

  IF lv_update_ok = 'X'.
    SORT lt_trstlmnt BY tknum.
    READ TABLE lt_trstlmnt INTO lw_trstlmnt WITH KEY tknum = lw_nlp_hit-tknum BINARY SEARCH.
    IF sy-subrc = 0.
      lw_trstlmnt-nlp_amt  = lw_nlp_hit-nlp_amount.
      lw_trstlmnt-nlp_acks = lc_02.
      MODIFY lt_trstlmnt FROM lw_trstlmnt TRANSPORTING nlp_amt nlp_acks WHERE tknum = lw_nlp_hit-tknum.
    ENDIF.
    DELETE lt_trstlmnt WHERE tknum = lw_nlp_hit-tknum.
  ENDIF.

ENDLOOP.
```

**Notes**

- **Ceiling:** **`IF lw_nlp_hit-nlp_amount GT lw_amount_param-amount. CONTINUE.`** mirrors **`lw_attr-amount GT lw_amount_param-amount`** / **`CONTINUE`** in the notional loop (~1191), including behaviour when **`lw_amount_param-amount`** is initial (see §2 rule 3).
- **`lc_02`**: reuse existing constant for **`'02'`**.
- **`DELETE lt_trstlmnt`** only after **successful** **`UPDATE`** and **`COMMIT`** (**`lv_update_ok`**), so failed updates keep the shipment in the working set.
- **`DEQUEUE`** after the update attempt, consistent with the notional block (~1348–1351).

### 6.4 Optional application log

If required for audit, call **`Z_APPLICATION_LOG_CREATE`** with **`objectkey`** including **`tknum`**, **`srno`**, **`nlp_amount`** when **`lv_update_ok`** is set.

### 6.5 Public interface

No change to **`Z_LOG_TCFA_REPROCESS`** parameters unless a separate functional request adds a switch.

---

## 7. Technical constraints (target NetWeaver 7.31)

- No inline **`DATA(...)`**; no **`VALUE` / `NEW` / string templates`**.
- Explicit **`SELECT`** field list; local structure field order **matches** **`SELECT`** order.
- No **`SELECT`** inside **`LOOP`** over shipments for this feature.
- Check **`sy-subrc`** after **`SELECT`**, **`CALL FUNCTION`**, **`READ TABLE`**, **`UPDATE`**, **`ENQUEUE`**.
- Run **Code Inspector** on the function group after implementation.

---

## 8. Test scenarios

| # | Scenario | Expected |
|---|----------|----------|
| 1 | No row in **`ZLOG_INT_DB`** matching filters | Standard freight + liability + notional path. |
| 2 | Success row, **`NLP_AMOUNT`** not **`GT`** **`lw_amount_param-amount`** | **`ZTRSTLMNT`** updated; **`lt_trstlmnt`** rows for **`tknum`** removed; no notional call for that shipment. |
| 3 | Success row, amount **`GT`** limit (same as notional **`CONTINUE`**) | No reuse; standard path. |
| 4 | Two success rows, different **`SRNO`** | Amount from **higher `SRNO`**. |
| 5 | **`ENQUEUE`** fails | No update, no **`DELETE`**; standard path. |
| 6 | **`UPDATE`** fails | **`e_error`**, rollback, no **`DELETE`**; standard path. |
| 7 | **`ZSCE_TCFA_NLP_MAX_AMOUNT`** not maintained (**`lw_amount_param`** cleared) | Ceiling compares like existing FM (initial **`amount`**); reuse logic stays **consistent** with notional **`CONTINUE`**. |
| 8 | Regression | Shipments without log hit unchanged vs current release. |

---

## 9. Transport

- Recommended single transport: **`Z_LOG_TCFA_GET_NLP_INT_DB`**, FG include updates for **`Z_LOG_TCFA_REPROCESS`**, optional DDIC **`ZST_*` / `ZTT_*`**.
- Activate DDIC before / with FM per STMS practice.

---

## 10. Implementation checklist

- [ ] Create **`Z_LOG_TCFA_GET_NLP_INT_DB`** in the same function group as **`Z_LOG_TCFA_REPROCESS`**.
- [ ] Add **`ZST_TCFA_NLP_INT_DB` / `ZTT_TCFA_NLP_INT_DB`** or FG-local types; activate.
- [ ] Implement **`SELECT`** + max-**`SRNO`** per shipment (§5.3).
- [ ] Insert caller §4 / §6.2–6.3 before FCPL loop (~553).
- [ ] Code Inspector; run tests §8.
