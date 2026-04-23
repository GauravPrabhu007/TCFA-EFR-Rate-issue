# Code Change Document  
## `Z_SCM_FREIGHT_NLP_NEW` — EFR product category (`Z_EFR_PROD_CAT`)

| Document control | |
|--------------------|---|
| **Function module** | `Z_SCM_FREIGHT_NLP_NEW` |
| **Change label** | TCFA EFR parity (NLP / MGX) |
| **Version** | 1.0 |
| **Companion** | `Functional_Specification_Z_SCM_FREIGHT_NLP_NEW_EFR.md` |

---

## 1. Summary

Insert an **MFRGR-gated** call to **`Z_EFR_PROD_CAT`** after **`lw_prod_category`** is populated from **`ZLOG_EXEC_VAR`** and **before** the **“Get the service product”** step, then optionally **override `lw_prod_category`** from **`et_shipmentoprd_cat`**. Use **`cn_date = I_LR_DATE`** when **`I_LR_DATE` IS NOT INITIAL**, else **`sy-datum`**.

No other objects are changed in this document.

---

## 2. Objects touched

| Object type | Name | Change |
|-------------|------|--------|
| FU / FM | `Z_SCM_FREIGHT_NLP_NEW` | Insert DATA (if not present), constants, logic block |

---

## 3. Location in source

**File (reference extract):** `FM - Z_SCM_FREIGHT_NLP_NEW.txt`  
**Scenario:** `CASE lw_scenario.` → **`WHEN 'SOLIDS_OB'.`**

**Insertion point:** Immediately **after** the existing block that sets **`lw_prod_category`** (comments `"Get the product category"` / `READ TABLE lt_zlog_exec_var` … **`lw_prod_category`**) and **before** the block `"Get the service product"` (`READ TABLE` … **`lc_servcat`**).

Approximate line anchor in reference file: **after line ~704**, **before line ~706**.

---

## 4. Data definitions

Add in the **DATA** section of the FM (or nearest local block for `SOLIDS_OB`), if not already declared:

```abap
" EFR / TCFA parity — Z_EFR_PROD_CAT
DATA: lt_shipment_det     TYPE zscm_efr_shp_tt,
      lw_shipment_det     TYPE zscm_efr_shp,
      lt_shipmentoprd_cat TYPE zscm_efr_shp_tt,
      lw_shipmentoprd_cat TYPE zscm_efr_shp,
      lv_efr_cn_date      TYPE sy-datum.
```

**Note:** Use the **exact** DDIC types exported by **`Z_EFR_PROD_CAT`** in your system (`ZSCM_EFR_SHP_TT` / `ZSCM_EFR_SHP` are used in `Z_SCM_SHIP_FREIGHT_CHK_QWIK`; confirm in SE37 if names differ).

---

## 5. Constants — EFR-eligible MFRGR

```abap
CONSTANTS:
  lc_efr_mfrgr_rubr TYPE mfrgr VALUE 'RUBR',
  lc_efr_mfrgr_rumb TYPE mfrgr VALUE 'RUMB',
  lc_efr_mfrgr_poyo TYPE mfrgr VALUE 'POYO',
  lc_efr_mfrgr_polm TYPE mfrgr VALUE 'POLM',
  lc_efr_mfrgr_polmj TYPE mfrgr VALUE 'POLMJ',
  lc_efr_mfrgr_poyj TYPE mfrgr VALUE 'POYJ',
  lc_efr_mfrgr_poyl TYPE mfrgr VALUE 'POYL',
  lc_efr_mfrgr_poyy TYPE mfrgr VALUE 'POYY'.
```

**Alternative (recommended for maintainability):** single internal table **`lt_efr_mfrgr`** filled once in initialization or static fill, then **`READ TABLE … WITH KEY table_line = lw_mfrgr`**, to avoid long **`IF lw_mfrgr = … OR …`** chains.

---

## 6. Logic fragment (pseudocode / paste guide)

Place **inside** the existing nest where **`lw_mfrgr`**, **`lw_prod_category`**, **`lw_vttk`**, **`lw_vttp`**, **`lt_lips`** are available — **same depth** as current **`CALL FUNCTION 'Z_SCE_CONTRACT_TYPE'`** (i.e. after **`READ TABLE lt_lips`** success and after **`lw_prod_category`** assignment).

```abap
" EFR product category (TCFA parity with CCF) — gated by MFRGR
IF lw_mfrgr = lc_efr_mfrgr_rubr OR lw_mfrgr = lc_efr_mfrgr_rumb OR
   lw_mfrgr = lc_efr_mfrgr_poyo OR lw_mfrgr = lc_efr_mfrgr_polm OR
   lw_mfrgr = lc_efr_mfrgr_polmj OR lw_mfrgr = lc_efr_mfrgr_poyj OR
   lw_mfrgr = lc_efr_mfrgr_poyl OR lw_mfrgr = lc_efr_mfrgr_poyy.

  CLEAR: lt_shipment_det[], lt_shipmentoprd_cat[],
         lw_shipment_det, lw_shipmentoprd_cat, lv_efr_cn_date.

  IF i_lr_date IS NOT INITIAL.
    lv_efr_cn_date = i_lr_date.
  ELSE.
    lv_efr_cn_date = sy-datum.
  ENDIF.

  lw_shipment_det-tknum   = lw_vttk-tknum.
  lw_shipment_det-cn_date = lv_efr_cn_date.
  lw_shipment_det-tdlnr   = lw_vttk-tdlnr.
  APPEND lw_shipment_det TO lt_shipment_det.

  IF lt_shipment_det IS NOT INITIAL.
    CALL FUNCTION 'Z_EFR_PROD_CAT'
      EXPORTING
        it_shipment_det     = lt_shipment_det
      IMPORTING
        et_shipmentoprd_cat = lt_shipmentoprd_cat.

    IF lt_shipmentoprd_cat IS NOT INITIAL.
      SORT lt_shipmentoprd_cat BY tknum.
      READ TABLE lt_shipmentoprd_cat INTO lw_shipmentoprd_cat
        WITH KEY tknum = lw_vttk-tknum
        BINARY SEARCH.
      IF sy-subrc = 0 AND lw_shipmentoprd_cat-product_cat IS NOT INITIAL.
        lw_prod_category = lw_shipmentoprd_cat-product_cat.
      ENDIF.
    ENDIF.
  ENDIF.
ENDIF.
```

### 6.1 Adjustments for production code

1. **Replace** the **`IF … OR …`** block with **`READ TABLE lt_efr_mfrgr`** or **`lw_mfrgr IN lt_efr_mfrgr_range`** per team standard.

2. **`BINARY SEARCH`:** only valid if **`SORT lt_shipmentoprd_cat BY tknum`** is executed (as above).

3. **Authority / performance:** **`Z_EFR_PROD_CAT`** issues selects on **`VTTK`**, **`VTTP`**, **`LIPS`**, **`MARC`**, **`ZSCM_EFRCONTYP`**, etc. One call per shipment in this path is acceptable; avoid calling inside tight loops if the FM is ever moved.

4. **`#EC` pragmas:** Add only if static check requires (e.g. CI_SUBRC after CALL).

---

## 7. Code **not** moved or duplicated

- Do **not** change **`Z_SCE_CONTRACT_TYPE`** signature or call parameters other than **`lw_prod_category`** being possibly updated beforehand.
- Do **not** change **`Z_SCE_GET_ROAD_RATES_QWIK`** export list in this change document (optional separate CR).
- Do **not** alter **`LIQUIDS_*`** branches in this FM.

---

## 8. Regression matrix (technical)

| Area | Check |
|------|--------|
| Non-EFR MFRGR | No new CALL; `lw_prod_category` unchanged vs backup |
| EFR MFRGR, no `ZSCM_EFRCONTYP` | Override skipped |
| EFR MFRGR, hit | `lw_prod_category` = FM output; `lw_contract_type` may change vs old |
| `I_LR_DATE` initial | `cn_date = sy-datum` |
| `lt_marc` / `lw_mfrgr` | Gate uses same **`lw_mfrgr`** as sub-format block (already set lines ~656–666 in reference) |

---

## 9. Transport and dependencies

- **Transport:** workbench request containing only `Z_SCM_FREIGHT_NLP_NEW` (or its include if split).
- **Dependencies:** `Z_EFR_PROD_CAT` must be active and remote-enabled if cross-client (same as CCF usage).

---

## 10. Rollback

Revert FM to previous version; no database migration.

---

*End of document*
