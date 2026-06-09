# Material Classification Tool — Database Package

Everything needed to build and sync the tool's database (Firebase Firestore),
in one folder.

## Folder layout

```
material_tool_db/
├── README.md                  ← this file
├── SCHEMA.md                  ← full database structure reference
├── data/                      ← the source data (rebuilt from Supabase, corrected)
│   ├── materials_v2.json
│   ├── vm_classification_v2.json
│   ├── consumption_2025_v2.json
│   └── vendors_v2.json
└── scripts/
    ├── rebuild_database.py    ← (re)builds collections 1-4, seeds 5-6
    ├── vm_stock_fetch.py      ← VM stock → CSV only (verification run)
    ├── vm_stock_sync.py       ← VM stock → Firestore live_stock + CSV
    └── vm_config.json         ← your VM login (fill this in; stays local)
```

You also need (NOT included, add yourself, keep private):
- Your **Firebase service-account .json** → place in the `scripts/` folder.

## The database — 6 collections, all linked by SKU

| # | Collection | What it holds | Filled by |
|---|------------|---------------|-----------|
| 1 | materials | 37-col item master (no stock/price) | rebuild_database.py |
| 2 | vm_classification | consumption, min/max, lead time | rebuild_database.py |
| 3 | consumption_2025 | regional 3-month consumption | rebuild_database.py |
| 4 | vendors | vendor master | rebuild_database.py |
| 5 | live_stock | current closing stock from VM + adjustment | vm_stock_sync.py + the tool |
| 6 | stock_adjustments | audit log (sku, amount, reason) | the tool |

See `SCHEMA.md` for every field.

## One-time setup

```
pip install firebase-admin requests beautifulsoup4
```

1. Put your Firebase service-account .json in `scripts/`.
2. Open `scripts/vm_config.json`, fill in your VM login + password.

## Usage

**Build / rebuild the database (collections 1-4):**
```
cd scripts
python rebuild_database.py
```

**Sync live stock from the VM software (run a few times a day):**
```
cd scripts
python vm_stock_sync.py
```
Each run logs into vm.printo.in, reads the Closing Stock report, writes
`closing_stock` + `synced_at` to `live_stock`, and saves a CSV backup.
It refuses to write if it parses too few rows (safety gate).

**Just verify VM stock without touching the database:**
```
python vm_stock_fetch.py
```
Saves a CSV only.

## Key design rules

- **Stock is read-only from VM.** `closing_stock` is never edited by hand.
- **Adjustments are separate.** `adjusted_count = closing_stock + adjustment`,
  stored in its own field; `closing_stock` stays untouched.
- **Adjustments persist** across syncs and stack on the latest stock, until
  cleared in the tool. Every change is logged in `stock_adjustments`.
- **BLR vendor** field is combined `Name-LeadTime` (e.g. `Canvera-3`); the tool
  parses the lead time from the suffix. NCR/HYD/CHN keep vendor + lead time
  in separate fields.
- **Editable in the tool:** vendor, lead time, unit price, MOQ.

## Future migration (AWS / local server)

The data is plain SKU-keyed JSON — portable to SQL or other NoSQL stores.
When moving off Firebase, the main work is putting the database behind a
small API and pointing the tool's data layer at it. Build the tool with all
DB calls in one place to make that swap easy.
