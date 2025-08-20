# Uber Eats A/B Simulator — Free Delivery vs $5 Off (MOV + CUPED)

[![python](https://img.shields.io/badge/python-3.9%2B-blue.svg)](#)
[![license: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![deps](https://img.shields.io/badge/deps-numpy%20%7C%20pandas-lightgrey.svg)](#)

A tiny, reproducible **A/B testing simulator** for Uber-style promos: **Free Delivery** vs **$5 Off**. It estimates **Net Incremental Revenue per 1,000 exposures (NIR/1k)**, models **Minimum Order Value (MOV)** rules (friction + flooring), and applies **CUPED** for variance reduction. Built with NumPy/Pandas; no heavy deps.

**Last updated:** 2025-08-20

---

## TL;DR
- Randomizes users 50/50 into **Free Delivery** or **$5 Off**.  
- Computes **NIR per exposure** and scales to **NIR/1k** with **bootstrap 95% CIs** and a Welch test.  
- Includes **MOV** (conversion friction + basket flooring) and optional **CUPED** (pre-period covariate) for tighter CIs.

---

## Features
- Per-user **price sensitivity** heterogeneity (Beta distribution)
- Platform economics: **take-rate (commission)**, **delivery fee margin**, **courier cost**
- **MOV**: reduces conversion if expected basket < MOV (*friction*) and tops up converting baskets to ≥ MOV (*flooring*)
- **CUPED**: pre-period baseline net revenue as covariate (variance reduction)
- Reproducible seeds; clean NumPy/Pandas implementation

---

## Install

```bash
# in a fresh virtualenv
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
# if you have pyproject/setup, otherwise add src to PYTHONPATH
pip install -e . || export PYTHONPATH=$PYTHONPATH:$(pwd)/src
```

---

## Quickstart (Python)

```python
from ubereats_ab import BizParams, simulate, summarize_variant, cuped_adjusted_metric, diff_in_means_test

# 1) Configure assumptions
P = BizParams(
    n_exposures=100_000, seed=42,
    base_conv=0.10, take_rate=0.12, df_margin_frac=0.20,
    mov_value=12.0, mov_friction=0.15,
    uplift_fd_pp=0.025, uplift_5off_pp=0.030,
    aov_mult_fd=1.00, aov_mult_5off=1.05,
    five_off_value=5.00, five_off_threshold=15.00,
    cuped=True
)

# 2) Simulate baseline + A/B
res = simulate(P)
A, B = res["A_FREE_DELIVERY"], res["B_FIVE_OFF"]

# 3) Summaries (RAW)
summ_A = summarize_variant(A, P.bootstraps)
summ_B = summarize_variant(B, P.bootstraps)

# 4) CUPED summaries
cup_A = cuped_adjusted_metric(A, P.bootstraps)
cup_B = cuped_adjusted_metric(B, P.bootstraps)

# 5) Differences
d_raw, p_raw = diff_in_means_test(A["nir_per_exposure"].values, B["nir_per_exposure"].values)
print("Δ NIR per 1k (B–A):", d_raw * 1000, "p=", p_raw)
```

Save artifacts:

```python
res["all"].to_csv("runs/per_exposure.csv", index=False)

import json
json.dump(
  {"A_raw": summ_A, "B_raw": summ_B, "A_cuped": cup_A, "B_cuped": cup_B,
   "diff_raw_per_1k": float(d_raw*1000)},
  open("runs/summary.json","w"), indent=2
)
```

---

## Example output (simulated)

```
=== Uber Eats A/B Test Simulation (with MOV & CUPED) ===
Exposures: A=50,025  B=49,975
Conv Rate: A=11.620%  B=11.670%
Avg AOV (baseline ref): A=$28.79  B=$28.61

--- RAW Net Incremental Revenue per 1,000 exposures ---
A Free Delivery: $-508.25  (95% CI: $-521.35, $-493.77)
B $5 Off      : $-457.39  (95% CI: $-470.09, $-445.12)
Difference (B - A): $50.85   Welch p-value: 4.704e-08

--- CUPED-ADJUSTED Net Incremental Revenue per 1,000 exposures ---
A Free Delivery (CUPED): $-508.25  (95% CI: $-521.74, $-494.82)
B $5 Off       (CUPED): $-457.39  (95% CI: $-470.57, $-445.87)
Pooled theta (CUPED): -0.0002
Difference (B - A), CUPED: $50.86   Welch p-value: 4.699e-08

Winner on expected NIR/1k: B ($5 Off)
```

**Interpretation:** both promos are **negative NIR** vs. no promo (under these assumptions), but **$5 Off** is **less negative** than Free Delivery by ≈ **$51 per 1k**. CUPED did little here (θ≈0).

---

## Key knobs (`BizParams`)
- **Demand/AOV**: `base_conv`, `aov_lognorm_*`
- **Fees & economics**: `deliv_fee_*`, `take_rate`, `df_margin_frac`
- **Promos**: `uplift_fd_pp`, `uplift_5off_pp`, `aov_mult_*`, `five_off_*`
- **MOV**: `mov_value`, `mov_friction`
- **CUPED**: `cuped`, `cuped_noise_sd`
- **Scale**: `n_exposures`, `seed`

---

## How it computes NIR
Per converting order:
```
net = take_rate * subtotal
    + (charged_delivery ? delivery_fee : 0)
    - courier_cost(delivery_fee)
    - promo_discount
```
- **Free Delivery**: no fee revenue; courier cost still incurred.  
- **$5 Off**: fee charged; discount applies only if subtotal ≥ threshold.  
**NIR per exposure** = `net_variant − net_baseline`, then scale to **NIR/1k**.

---

## Project layout

```
src/ubereats_ab/
  __init__.py
  version.py
  sim.py                  # BizParams, simulate(), summarize_variant(), CUPED, tests
tests/
```

---

## Contributing
PRs welcome—tests, docs, or new features (power calc, targeting rules, control-only CUPED).

## License
MIT. **Disclaimer:** This is a simulation; real-world results depend on marketplace dynamics, supply pricing, fraud, and seasonality.
