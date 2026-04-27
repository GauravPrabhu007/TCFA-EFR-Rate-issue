# Code change document  
## `Z_LOG_TCFA_REPROCESS` — Subformat read key (`SCME_SUBFORMATID` / `LIPS-MFRGR`)

| Document control | |
|------------------|---|
| **Function module** | `Z_LOG_TCFA_REPROCESS` |
| **Change label** | TCFA reprocess: correct `ZLOG_EXEC_VAR` key for subformat (avoid common rate when vendor + subformat applies) |
| **Version** | 1.0 |
| **Companion FS** | `FS - Bug Fix - TCFA Reprocess considering common rate (not as per Subformat & Vendor).md` |
| **Reference extract (this repo)** | `FM - Z_LOG_TCFA_REPROCESS.txt` |

---

## 1. Summary

Correct the **`ZLOG_EXEC_VAR`** read for **`gc_subform_id`** (`SCME_SUBFORMATID`) by using **`LIPS-MFRGR`** (material freight group) as the **`mfrgr`** key instead of **`LIPS-MATKL`** (material group). Extend the local **`LIPS`** structure and **`SELECT`** so **`mfrgr`** is available on **`lw_lips`**.

**Do not** add a **`MARC`** read for this change (per FS).

**No other objects** are in scope for this transport item unless your team splits includes separately.

---

## 2. Objects touched

| Object type | Name | Change |
|-------------|------|--------|
| FU / FM | `Z_LOG_TCFA_REPROCESS` | Extend `lty_lips` + `SELECT` from `LIPS`; fix `READ TABLE lt_zlog_exec_var` key for `gc_subform_id` |

---

## 3. Location in source

**Reference file:** `FM - Z_LOG_TCFA_REPROCESS.txt`

| Change | Approx. lines (reference extract) |
|--------|----------------------------------|
| Type `lty_lips` | ~86–92 |
| `SELECT` from `LIPS` | ~671–678 |
| Erroneous `READ TABLE` | ~781–782 |

In SE37, locate the **`LOOP AT lt_trstlmnt`** block where **`lt_strlmt_fcpl`**, **`lt_vttk`**, **`lt_vttp`**, **`lt_lips`**, and **`lt_zlog_exec_var`** are used to build **`lw_sub_format`** / **`lw_contract_type`** before **`Z_SCE_SHIP_FREIGHT_CAL_QWIK`**.

---

## 4. Technical design

### 4.1 Root cause (one line)

`ZLOG_EXEC_VAR-MFRGR` is maintained for **material freight group**; the code passed **`LIPS-MATKL`**, causing wrong or failed lookup for **`SCME_SUBFORMATID`**.

### 4.2 Change steps

1. Add **`mfrgr`** to local type **`lty_lips`** using the DDIC type of **`LIPS-MFRGR`** (typically **`mfrgr`** from **`LIPS`**).
2. Add **`mfrgr`** to the **`SELECT`** list from **`LIPS`**, in the **same order** as the fields in **`lty_lips`** (see **§5 ABAP Rules** — structure / `SELECT` alignment).
3. Replace **`mfrgr = lw_lips-matkl`** with **`mfrgr = lw_lips-mfrgr`** on the **`READ TABLE lt_zlog_exec_var`** for **`name = gc_subform_id`** only.

**Do not** change other **`READ TABLE lt_zlog_exec_var`** calls that intentionally use **`zzpmatkl1`** / **`matkl`** for business rules.

---

## 5. ABAP rules compliance (`ABAP Rules - 02-04-2026`)

This change is **legacy function module maintenance** (allowed under core principles; no new `FORM`/`PERFORM` introduced).

| Rule source | Application to this change |
|-------------|------------------------------|
| **`00-main.mdc`** (NetWeaver **7.31** / `abap_731`) | No inline declarations, no constructor operators, no string templates. Use **existing** `DATA` / `TYPES` style of the FM. |
| **`03-database.mdc`** | Keep **explicit field lists** on `SELECT`; when extending `lty_lips`, **align `SELECT` field order with the structure** so populated fields are unambiguous. Do **not** introduce `SELECT *` on `LIPS`. |
| **`02-naming.mdc`** | Reuse existing prefixes (`lw_lips`, `lt_lips`, `lty_lips`, `gc_subform_id`). No new global types required if `lty_lips` remains local. |
| **`20-code-generation-checklist.mdc`** | After `SELECT`, `lt_lips` is already **`SORT BY vbeln`** before **`READ TABLE ... WITH KEY vbeln`** — **leave intact**. No `SELECT` inside loops added. |

**Code Inspector:** Run **SCI / Code Inspector** (and your transport quality gate) on the function group after the change; resolve any new findings per team policy.

---

## 6. Code fragments (implementation guide)

> **Note:** Adjust includes / client clauses to match your system copy of the FM if they differ from the reference extract.

### 6.1 Extend `lty_lips`

**Before:**

```abap
  TYPES: BEGIN OF lty_lips,
           vbeln TYPE lips-vbeln,
           posnr TYPE lips-posnr,
           matnr TYPE lips-matnr,
           matkl TYPE lips-matkl,
           spart TYPE lips-spart,
   END OF lty_lips.
```

**After (example — keep field order identical to `SELECT`):**

```abap
  TYPES: BEGIN OF lty_lips,
           vbeln TYPE lips-vbeln,
           posnr TYPE lips-posnr,
           matnr TYPE lips-matnr,
           matkl TYPE lips-matkl,
           mfrgr TYPE lips-mfrgr,
           spart TYPE lips-spart,
   END OF lty_lips.
```

### 6.2 Extend `SELECT` from `LIPS`

**Before:**

```abap
    SELECT vbeln
           posnr
           matnr
           matkl
           spart FROM lips
      INTO TABLE lt_lips
```

**After:**

```abap
    SELECT vbeln
           posnr
           matnr
           matkl
           mfrgr
           spart FROM lips
      INTO TABLE lt_lips
```

If your system standard requires **`CLIENT SPECIFIED`** on `LIPS`, follow the **same pattern** as elsewhere in this FM (do not mix standards within one `SELECT`).

### 6.3 Fix `READ TABLE` key

**Before:**

```abap
              READ TABLE lt_zlog_exec_var INTO lw_zlog_exec_var WITH KEY name = gc_subform_id
                                                                          mfrgr = lw_lips-matkl.
```

**After:**

```abap
              READ TABLE lt_zlog_exec_var INTO lw_zlog_exec_var WITH KEY name = gc_subform_id
                                                                          mfrgr = lw_lips-mfrgr.
```

### 6.4 Optional comment (recommended, one line)

Above the corrected `READ TABLE`, a short comment helps auditors link to the FS, for example:

```abap
              " CD nnn: SCME_SUBFORMATID key must use LIPS-MFRGR (ZLOG_EXEC_VAR-MFRGR), not MATKL
```

---

## 7. Regression and test hints (for ABAP / QA)

1. **Shipment with vendor + subformat rate:** Reprocess should pick up **subformat-specific** conditions when `ZLOG_EXEC_VAR` is maintained for `LIPS-MFRGR`.
2. **`LIPS-MFRGR` initial:** No dump; existing fallback after failed `READ TABLE` remains.
3. **Batch / `LOG_BATCH`:** Same logic path as interactive reprocess for the touched block.
4. Compare **`lw_sub_format`** / **`lw_contract_type`** (debugger) before vs after on a known failing shipment.

---

## 8. Transport

| Item | Detail |
|------|--------|
| **Workbench request** | Single TR containing **`Z_LOG_TCFA_REPROCESS`** (or FG include actually containing the code) |
| **Rollback** | Restore previous FM / include version |

---

## 9. Sign-off (optional)

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Functional | | | |
| Technical (ABAP) | | | |
| QA | | | |

---

*End of document*
