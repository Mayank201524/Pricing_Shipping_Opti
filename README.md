# Dynamic Pricing & Elasticity-Based Profit Optimization

This repository contains a **single notebook** that cleans sales data, estimates
**price elasticity**, and runs a **guarded price simulation** to recommend profit-maximizing
price moves at the **category** (or SKU) level.

- Notebook: `pricing_shipping_optimization_Mayank_task.ipynb`
- Outputs (CSV + PNG) are written to the `artifacts/` folder.

---

## What the notebook does

1. **Configure & load data**
   - Centralized `CONFIG` block for file paths, column aliases, and cost assumptions.
   - Robust CSV loading for messy exports (multiple encodings/engines).
   - Header normalization to a canonical schema: `date, sku, category, qty, price`.
   - Derives `price` from `Amount / Qty` when missing.

2. **Attach costs**
   - Maps SKU unit costs from catalogs (min of `TP/TP1/TP2` per SKU).
   - Adds per-unit shipping and expected return costs.
   - Falls back to default unit cost if a SKU is missing in catalogs.

3. **Diagnostics**
   - Basic coverage checks (row counts, missing dates, unique SKUs/categories).
   - Price variation (CV) by SKU/category to see where elasticity is estimable.

4. **Elasticity estimation**
   - Per-group OLS: `ln(qty) ~ ln(price) + weekday + month + const`.
   - Pooled interaction model with category dummies; weekly fallback if sparse.
   - Guardrails: elasticities clipped to `[−5, 0]`.

5. **Guarded price simulation**
   - For each category, simulate a grid of price moves (default `−30% … +30%`).
   - For near-zero elasticity (β ≥ −0.02), restrict to **±5%** moves.
   - Demand response: `q1 = q0 * (p1/p0)^β`.
   - Profit: `(p1 − unit_cost − ship_cost − return_cost) * q1`.

6. **Recommendations & roll-up**
   - Choose **max-profit** scenario per category (best Δp, price, qty, revenue, profit).
   - Compare to baseline to compute absolute / % lifts.
   - Add **ALL_CATEGORIES** row to show portfolio-level impact.

7. **Artifacts**
   - CSVs: full simulation grid, best recommendations, memo table, global totals.
   - PNGs: elasticity histogram, profit-vs-price curves per top categories.

---

## Inputs (put these in the repo root)

The notebook expects the following files (as referenced in the CONFIG cell).  
You can adjust names/paths by editing the CONFIG block at the top of the notebook.

- Sales:
  - `Amazon Sale Report.csv`
  - `International sale Report.csv`
  - `Sale Report.csv`
- Catalogs (unit costs; requires `TP/TP1/TP2` columns):
  - `May-2022.csv`
  - `P  L March 2021.csv`

> If any file is missing, the notebook will skip it and continue where possible.  
> Column aliases (e.g., `["Date","DATE"]`, `["SKU","Sku","SKU Code"]`) are handled by CONFIG.

---

## Environment / Setup

- Python 3.9+ recommended

```bash
# (optional) create venv
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

pip install -r requirements.txt
````

**requirements.txt (minimal)**

```
pandas
numpy
pyyaml
statsmodels
matplotlib
```

---

## How to run

1. Open the notebook:

```bash
jupyter notebook pricing_shipping_optimization_Mayank_task.ipynb
```

2. Run cells **top to bottom**:

   * `CONFIG` → writes `config_notebook.yaml` and creates `artifacts/`
   * loaders & helpers (`safe_read_csv`, `_normalize_sales`, `load_all`, `load_catalogs`)
   * `derive_costs`
   * diagnostics
   * elasticity models (`estimate_group_elasticity`, `fit_pooled_elasticities`)
   * simulation + guardrails
   * memo + **ALL\_CATEGORIES** roll-up
   * plotting (histogram + profit surfaces)

3. Check outputs under `artifacts/`.

---

## Key outputs

* `artifacts/simulated_pricing_category_pooled.csv` – all simulated scenarios
* `artifacts/best_price_recos_category_pooled.csv` – best per category
* `artifacts/memo_category_recos_guarded.csv` – memo-ready table (guarded)
* `artifacts/memo_category_recos_guarded_FIXED.csv` – memo using **TOTAL qty** baseline
* `artifacts/memo_category_recos_with_total.csv` – includes **ALL\_CATEGORIES** roll-up
* `artifacts/elasticity_hist_category_guarded.png` – histogram of elasticities
* `artifacts/profit_surface_category_guarded_<Category>.png` – profit vs price change

---

## Interpreting results (quick guide)

* **Near-zero elasticities (β ≈ 0)** → demand is inelastic; small **+5%** increases often lift profit.
* **More negative β** (e.g., −0.1 to −0.3) → prices can be moved more; validate with tests.
* **Straight lines in plots** are expected when β ≈ 0 or the grid is limited to ±5%.

---

## Troubleshooting

* **NameError (loaders/functions)** → run earlier cells first (CONFIG → loaders → costs).
* **Empty outputs** → check file names/paths and column aliases in the CONFIG block.
* **Singular matrix / OLS warnings** → very low price variation; rely on pooled model or weekly fallback.

---

## Notes on methodology & safety

* Elasticities are clipped to `[−5, 0]` and “flat” categories use a conservative grid (**±5%**).
* Baseline **scale** uses **total quantity**, not mean, for realistic profit deltas.
* Recommendations are **decision support**; validate with controlled A/B tests.

---

## AI assistance

Some helper code and documentation were drafted with AI (OpenAI GPT).
All code was reviewed and validated by the author.

