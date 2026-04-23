# Functional Specification  
## EFR product category alignment in `Z_SCM_FREIGHT_NLP_NEW` (TCFA MGX)

| Document control | |
|--------------------|---|
| **Functional module** | `Z_SCM_FREIGHT_NLP_NEW` |
| **Title** | Align TCFA NLP freight contract / rate path with EFR-aware product category (CCF parity) |
| **Version** | 1.0 |
| **Related change** | TCFA EFR vs PMT rate discrepancy (QWIK / `YS01`) |

---

## 1. Background and problem statement

### 1.1 Current behaviour

- **CCF** (`Z_SCM_SHIP_FREIGHT_CHK_QWIK`) calls **`Z_EFR_PROD_CAT`**, which evaluates **`ZSCM_EFRCONTYP`** (departure zone, destination zone, material freight group, vendor, vehicle type where applicable, and validity dates). When a match exists, it returns a **product category** (e.g. `PACKSOLE`) that **overrides** the product category otherwise taken from **`ZLOG_EXEC_VAR`**. That value feeds **`Z_SCE_CONTRACT_TYPE`**, which drives **contract type** (`add03`, e.g. EFR vs PMT) and therefore which **condition record** is selected for freight.

- **TCFA MGX** (`Z_SCM_FREIGHT_NLP_NEW`) resolves **product category** only from **`ZLOG_EXEC_VAR`** and **does not** call **`Z_EFR_PROD_CAT`**. As a result, **`Z_SCE_CONTRACT_TYPE`** can resolve to a **different contract type** than CCF / EFS for the same physical shipment when **`ZSCM_EFRCONTYP`** applies.

### 1.2 Business impact

- Notional / NLP freight amounts and condition selection at **MGX / TCFA trigger** can use **PMT** (or other) rates while **CCF** and **EFS shipment cost** paths use **EFR** for the same corridor and vendor, causing reconciliation and audit differences.

### 1.3 Objective

For **in-scope** material freight groups only, extend **`Z_SCM_FREIGHT_NLP_NEW`** so that **product category** is determined **the same way as CCF** with respect to **EFR** (i.e. include **`Z_EFR_PROD_CAT`** and apply override when returned), before **`Z_SCE_CONTRACT_TYPE`**, so that **contract type** and downstream **QWIK rate** selection align with CCF where master data intends an EFR path.

---

## 2. Scope

### 2.1 In scope

| Item | Detail |
|------|--------|
| **Object** | Function module `Z_SCM_FREIGHT_NLP_NEW` only |
| **Scenario** | **`SOLIDS_OB`** branch where shipment header, route zones, `LIPS`, `MARC`, and existing `ZLOG_EXEC_VAR` derivation for sub-business / sub-format / product category / service product are already executed |
| **Material freight groups** | **RUBR, RUMB, POYO, POLM, POLMJ, POYJ, POYL, POYY** only |
| **EFR logic** | Reuse **`Z_EFR_PROD_CAT`** (same interface pattern as **`Z_SCM_SHIP_FREIGHT_CHK_QWIK`**) |
| **Date for `ZSCM_EFRCONTYP` validity** | **Option A:** `cn_date` passed to **`Z_EFR_PROD_CAT`** = **`I_LR_DATE`** when **not initial**; otherwise **`sy-datum`** |

### 2.2 Out of scope (this release)

| Item | Detail |
|------|--------|
| **`Z_LOG_TCFA_REPROCESS`** | Explicitly deferred to a later change |
| **`Z_LOG_VEND_NLP_POL`** | POL / liquid-specific path; not in scope |
| **`Z_LOG_VEND_NLP_UPD`**, **`Z_SCE_SHIP_FREIGHT_CAL_QWIK`**, **`Z_SCM_SHIP_FREIGHT_CHK_QWIK`** | No change in this specification |
| **Master data** | No change to **`ZSCM_EFRCONTYP`**, **`ZSCM_CONTYPE`**, or pricing conditions; only application logic |
| **RFC `I_CONTRACT_TYPE` on `Z_SCE_GET_ROAD_RATES_QWIK`** | Optional future hardening; not part of this FS |

---

## 3. Stakeholders and references

| Role | Interest |
|------|----------|
| SCM / Logistics | Correct NLP and TCFA amounts vs CCF |
| Pricing / PNC | Consistent `YS01` / truck-load dimension (`EFR` vs `PMT`) |

**Reference implementation (parity target for EFR block):**  
`Z_SCM_SHIP_FREIGHT_CHK_QWIK` → build `it_shipment_det` → `Z_EFR_PROD_CAT` → optional override of product category from `et_shipmentoprd_cat`.

**Underlying tables / FMs:**  
`ZSCM_EFRCONTYP`, `Z_EFR_PROD_CAT`, `Z_SCE_CONTRACT_TYPE`, `Z_SCE_GET_ROAD_RATES_QWIK` (existing chain in `Z_SCM_FREIGHT_NLP_NEW`).

---

## 4. Functional rules

### 4.1 Eligibility (MFRGR gate)

- After the FM has determined **`lw_mfrgr`** (same logic already used for sub-format: **`MARC-MFRGR`** when **`SCME_CHK_MFG_MARC_SUBFORMATID`** applies for `LIPS-MFRGR`, else **`LIPS-MFRGR`**), evaluate:

  **`lw_mfrgr` ∈ { RUBR, RUMB, POYO, POLM, POLMJ, POYJ, POYL, POYY }`**

- **If not in the set:** do **not** call **`Z_EFR_PROD_CAT`**; do **not** override product category; behaviour remains **identical** to current production.

- **If in the set:** execute the EFR block (section 4.2).

### 4.2 EFR block (when eligible)

1. Populate **one** row of **`it_shipment_det`** (type per **`Z_EFR_PROD_CAT`** signature, same as CCF):
   - **`tknum`** = shipment number from current processing (`lw_vttk-tknum`).
   - **`tdlnr`** = service agent on shipment (`lw_vttk-tdlnr`).
   - **`cn_date`** = **`I_LR_DATE`** if **not initial**, else **`sy-datum`**.

2. Call **`Z_EFR_PROD_CAT`** with **`it_shipment_det`** and receive **`et_shipmentoprd_cat`**.

3. If return table is **not initial**, **sort by `tknum`** (binary search consistency, same as CCF).

4. **Read** `et_shipmentoprd_cat` with key **`tknum`** = current shipment.

5. **If** read successful **and** **`product_cat`** is **not initial**: set **`lw_prod_category`** = returned **`product_cat`** (override value from **`ZLOG_EXEC_VAR`** for this execution path).

6. If **`Z_EFR_PROD_CAT`** returns no row or blank **`product_cat`**: leave **`lw_prod_category`** unchanged (still from **`ZLOG_EXEC_VAR`**).

### 4.3 Order of operations (relative to existing code)

Mandatory order to mirror CCF intent:

1. Existing: sub-business, sub-format (including POLM override), **product category from `ZLOG_EXEC_VAR`** (unchanged).  
2. **New:** EFR block (sections 4.1–4.2) when MFRGR eligible.  
3. Existing: **service product** from `ZLOG_EXEC_VAR`.  
4. Existing: **`Z_SCE_CONTRACT_TYPE`** → existing rate fetch and NLP calculation.

*Rationale:* CCF applies param-based product category first, then EFR override, then service product, then contract type.

### 4.4 Error handling

- **`Z_EFR_PROD_CAT`** is a **read / derive** FM in current usage; if it raises or returns empty, **do not** fail the whole TCFA flow unless existing project standards require otherwise. **Default:** no override, continue with **`lw_prod_category`** from parameters.

- No new user-facing message is **required** for FS v1.0; optional diagnostic log may be agreed separately.

---

## 5. Functional acceptance criteria

| # | Criterion |
|---|-----------|
| AC1 | For shipment with **`lw_mfrgr`** **not** in the eight listed values, NLP output and **`lw_contract_type`** / freight are **byte-for-byte identical** to pre-change behaviour (regression). |
| AC2 | For shipment with **`lw_mfrgr`** in the list **and** a valid **`ZSCM_EFRCONTYP`** row for **`cn_date`** (per 4.2), **`lw_prod_category`** after the new block equals **`Z_EFR_PROD_CAT`** output (e.g. `PACKSOLE` when FM sets it). |
| AC3 | For same shipment as AC2, **`Z_SCE_CONTRACT_TYPE`** receives the **post-override** **`lw_prod_category`** and resulting **`e_contract_type`** / selected rate align with **CCF** for the same **`cn_date`** rule (Option A) and same master data. |
| AC4 | When **`I_LR_DATE`** is supplied, **`ZSCM_EFRCONTYP`** effective range is evaluated against **`I_LR_DATE`**; when **`I_LR_DATE`** is initial, **`sy-datum`** is used. |
| AC5 | **`SOLIDS_OB`** paths with no `LIPS` / no `Vttp` continue to behave as today (no EFR call without valid context). |

---

## 6. Assumptions and dependencies

- **`Z_EFR_PROD_CAT`** and types **`ZSCM_EFR_SHP`** / **`ZSCM_EFR_SHP_TT`** (or equivalent used by CCF) exist and are callable from the NLP package include.
- **`lw_mfrgr`** is already populated before product category read in all **`SOLIDS_OB`** paths that reach **`Z_SCE_CONTRACT_TYPE`**; if not, implementation must define **`lw_mfrgr`** consistently before the MFRGR gate (see Code Change document).

---

## 7. Test scenarios (functional)

| ID | Scenario | Expected |
|----|----------|----------|
| T1 | MFRGR = **POLMJ**, `ZSCM_EFRCONTYP` valid on **`I_LR_DATE`**, CCF returns EFR | TCFA NLP uses same contract type / rate dimension as CCF after change |
| T2 | MFRGR outside list (e.g. generic solids) | No call to **`Z_EFR_PROD_CAT`**; same as before |
| T3 | MFRGR in list, no **`ZSCM_EFRCONTYP`** hit | No override; param-only **`lw_prod_category`** |
| T4 | **`I_LR_DATE`** blank | **`cn_date = sy-datum`**; EFR evaluated on posting date |
| T5 | MFRGR in list, **`I_LR_DATE`** on boundary of **`effective_from` / `effective_to`** | Matches **`Z_EFR_PROD_CAT`** rules (inclusive bounds per FM) |

---

## 8. Sign-off

| Name | Role | Date | Signature |
|------|------|------|-----------|
| | Functional owner | | |
| | Technical owner | | |

---

*End of document*
