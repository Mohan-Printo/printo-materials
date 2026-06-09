# Database Schema — Material Classification Tool

6 collections in Firestore (database id: `default`). All item collections are
keyed by **SKU** (in the VM software this is called **Material Code**).

---

## 1. materials  (key: SKU)
The item master — your 37-column spec. No stock, no unit price (those live in
`live_stock`).

| Field | Source | Notes |
|-------|--------|-------|
| blr, chn, hyd, pun, ncr | Input in tab | City presence flags (B/C/H/P/D or "Discontinued") |
| combined | Computed | Join of the 5 city flags |
| classification | Input | |
| type | Input | |
| sku | VM Software | = Material Code; document id |
| material_category_desc | VM Software | |
| category | VM Software | Category (manually done / vmmstr) |
| major_category | VM Software | |
| material_description | VM Software | |
| uom | VM Software | |
| supplier_moq | Input | **editable in tool** |
| season | Input | |
| launch_dt | Input | |
| criticality | Input | V.High / High / Med / Low |
| str_reorder_type | Input | BLR reorder type |
| blr_v1 | Ven Mstr | **combined "Name-LeadTime"** e.g. "Canvera-3"; tool parses lead time from suffix. **editable** |
| blr_v2 | Ven Mstr | |
| cat_mgr | Input | Category manager |
| ncr_lead_time | Ven Master | separate column (NCR) |
| ncr_str_reorder | Input | |
| ncr_ordering_frequency | Computed | |
| ncr_v1, ncr_v2 | Input | NCR vendors |
| hyd_lead_time | Ven Master | separate column (HYD) |
| hyd_str_reorder | Input | |
| hyd_ordering_frequency | Computed | |
| hyd_v1 | Input | |
| chn_lead_time | Ven Master | separate column (CHN) |
| chn_str_reorder | Input | |
| chn_ordering_frequency | Computed | |
| chn_v1 | Input | |
| product_sku | FIN Costing | |
| ratelist_name | FIN Costing | |

Lead-time rule: **BLR** lead time is parsed from the `blr_v1` suffix
(`Canvera-3` → 3). **NCR/HYD/CHN** use their own `*_lead_time` fields.

---

## 2. vm_classification  (key: SKU)
Consumption and reorder-level calculations.

`sku, cnsump_value, buffer_lead_time, new_buffer_lead_time, ord_freq,
min_qty, max_qty, min_val, max_val, old_min_val, old_max_val, helper,
avg_unit_price, consumption_per_month, lead_time, uncertainty_min,
uncertainty_max, vendor, matrl_categorization`

Notes: both consumption fields kept (`consumption_per_month` here and
`vm_consumption` in consumption_2025) — tool decides which to show.
Negative values are kept as-is (raw from source).

---

## 3. consumption_2025  (key: SKU)
Regional 3-month consumption.

`sku, material_desc, blr_3, chn_3, hyd_3, ncr_3, pun_3, grand_total,
vm_consumption, region_vm, material_desc_sys, combined`

---

## 4. vendors  (auto id)
Vendor master.

`region, vendor_name_sys, vendor_name_short, vendor_plus_ltime,
act_lead_time, l_time, vital_status, status, payment_days, mtrl_val,
ordering_frequency, type`

---

## 5. live_stock  (key: SKU)   ← stock + adjustments
Current stock from the VM software, plus the adjustment layer.

| Field | Written by | Notes |
|-------|-----------|-------|
| closing_stock | vm_stock_sync.py | **protected** — from VM, never hand-edited |
| synced_at | vm_stock_sync.py | timestamp of last VM sync |
| adjustment | the tool | running net +/- (persists across syncs) |
| adjusted_count | the tool | = closing_stock + adjustment (separate field) |

Rules:
- Pressing +25 sets/updates `adjustment` and recomputes `adjusted_count`;
  `closing_stock` is untouched.
- Adjustment **persists** and stacks on the latest synced stock.
- "Clear adjustment" in the tool resets `adjustment` to 0.

---

## 6. stock_adjustments  (auto id)   ← audit log
One row per adjustment action.

| Field | Notes |
|-------|-------|
| sku | which item |
| amount | the +/- applied (e.g. +25, -10) |
| reason | reason / note for the adjustment |

(Plus an automatic timestamp when each entry is written.)

---

## Relationships

```
materials.sku ─┬─ vm_classification.sku
               ├─ consumption_2025.sku
               ├─ live_stock.sku  ──(+ adjustment)──► adjusted_count
               └─ stock_adjustments.sku  (history)

materials.blr_v1 / *_v1  ──► vendors  (vendor reference)
```
