# Equal-width benchmark and early-warning diagnostic for CausalImpact intervals

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/vanajakodi0-beep/Research--1/blob/main/unified_pipeline_colab.ipynb)

Code for the paper *An Equal-Width Benchmark and Early-Warning Diagnostic for Post-Intervention Business Shifts in
CausalImpact Intervals* (RESEARCH_5.docx).

## Data (not redistributed)
| File | What it is |
|---|---|
| `Walmart_Sales.csv` | Kaggle `mikhail1681/walmart-sales`: 45 stores, first 143 weeks. Real data. Intervention after week 100 (43 post weeks). |
| `DMart_sample_data.csv` | 1,000,000 transactions, no store column. 45 pseudo-units are built by randomly grouping the 400 customer names (seed 2024); 105 weeks, intervention after week 75 (30 post weeks). **Synthetic stress test, not independent real-world evidence**: the file has uniform prices and quantities, equal category shares and white-noise weekly series (paper, Section 7.1.2). |

SHA-256 checksums of both files are written to `results/<panel>/run_config.json`.

## One pipeline, one engine
`unified_pipeline.py` loads each panel, builds seven scenarios, fits the BSTS, calibrates lambda / k / the alarm threshold on 22
calibration stores, evaluates on 23 test stores over 20 random splits, bootstraps confidence intervals and writes every table
and figure. **Every number in the paper comes from the default engine, `kalman`** (local-level BSTS, maximum likelihood,
numpy/scipy only). Both panels, 20 splits: about 25 seconds.

`--engine tfci` (tfcausalimpact) exists for cross-checks but **has not been run against the real package**, and no number in
the paper uses it. The tfcausalimpact tables of the earlier draft (old Sections 8.1-8.11) are not reproduced by any file in
this repository; the notebook that made them is not part of it.

```
pip install -r requirements.txt
python unified_pipeline.py --walmart Walmart_Sales.csv --dmart DMart_sample_data.csv --out results   # both panels
python unified_pipeline.py --walmart Walmart_Sales.csv --only walmart --out results                   # one panel
python make_paper_tables.py paper_tables.md                                                           # Tables 8.1-8.19 as markdown
```
`--smoke-test` uses synthetic data to test the code only; its output goes to `*_SMOKE_SYNTHETIC` and must not be used.

**Colab:** click the badge, then Runtime > Run all and upload the two CSV files when asked. `unified_pipeline_colab.ipynb` is
the same code as the script, cell by cell (26 function definitions, identical to `unified_pipeline.py`).

## DMart panel
`DMart_sample_data.csv` has one row per transaction (date, customer name, product category, customer type, unit price, quantity,
total) and **no store column**, so the pipeline builds a panel from it (`load_dmart` in `unified_pipeline.py`):

1. Parse dates and assign each row to its Monday-Sunday week; keep complete weeks only (105 weeks, 2 Jan 2023 - 5 Jan 2025).
2. Assign the 400 distinct customer names at random (seed 2024) to 45 groups of 8-9 names.
3. A unit's weekly sales = the weekly sum of `Total` over its group. This gives 45 units x 105 weeks.
4. Intervention after week 75 (week of 3 June 2024), 30 post weeks; 23 weeks of year-over-year history before it
   (Walmart: 100 / 43 / 48). Placebo origins for DiD and synthetic control start at 30 weeks (Walmart: 40).

Everything else (7 scenarios, 22/23 store splits, 20 splits, grids, threshold rule, bootstrap) is identical to Walmart.
Run it with `--only dmart`, or with both panels in one command (see above). Outputs are in `results/dmart/`.

**How to read the DMart results.** The file's unit prices and quantities are uniform, category and customer-type shares are
equal, and the weekly series show no trend, no seasonality and about zero autocorrelation (lag-1 -0.02, lag-52 -0.03; Walmart 0.45 and
0.39). The panel is therefore used as a structure-free stress test, not as independent evidence of real-world retail dynamics.
It checks whether the Walmart conclusions depend on Walmart's particular dynamics; a real second dataset (for example
Favorita) would be the next replication.

### Headline results, both panels (mean over 20 splits)
| Quantity (mean over 20 splits unless stated) | Walmart | DMart panel |
|-----------------------------------------|----------:|----------:|
| Standard BSTS, no shift | 75.2 | 91.5 |
| Standard BSTS, 5% common shift | 68.5 | 25.2 |
| Break-aware, 5% common shift | 95.8 | 96.2 |
| Fixed at break-aware width, 5% common shift | 97.4 | 100.0 |
| Break-aware, treated-only shift (A) | 90.3 | 74.4 |
| Fixed at break-aware width, treated-only shift (A) | 94.3 | 92.6 |
| DiD, best control, treated-only shift (A) | 78.9 | 72.2 |
| Synthetic control, treated-only shift (A) | 85.7 | 58.9 |
| Break-aware minus fixed at equal width, range over 7 scenarios (pp) | −4.1 to −0.8 | −18.2 to −1.0 |
| Scenarios with 95% CI of that difference below zero | 6 of 7 | 7 of 7 |
| 5% shift detected, one control (finite-sample rule) | 35.4% | 13.7% |
| 5% shift detected, ten pooled controls | 97.5% | 96.1% |
| False-alarm rate, ten pooled controls | 4.5% | 2.4% |

DMart-specific notes (paper Section 8.11):
* With no shift the standard interval is close to calibrated on DMart (91.5% coverage) and collapses under a 5% shift (25.2%), so the shift alone explains the failure there. On Walmart the standard interval under-covers even with no shift (75.2%).
* At equal width the break-aware interval covers less than fixed widening in all 7 scenarios (95% interval below zero in all 7).
* The alert detects a 5% shift in 13.7% of cases with one control (the finite-sample threshold is conservative with 22 calibration stores) and 96.1% with ten pooled controls.

## Which file makes which table
All files are in `results/<panel>/` (`walmart` for Tables 8.1-8.14, `dmart` for 8.15-8.18). `make_paper_tables.py` turns them into the paper's tables.

| Paper table / figure | Source file(s) | Function in `make_paper_tables.py` |
|---|---|---|
| Worked example (Section 8.2) | `worked_example.csv` | (read directly) |
| 8.1 Standard BSTS diagnostics | `standard_bsts_diagnostics.csv` | `t_std` |
| 8.2, 8.3, 8.5 Coverage (CI, all methods, primary split) | `coverage_cis.csv` | `t_cov_ci`, `t_cov_all` |
| 8.4 Width relative to standard | `table_width_vs_standard_mean_over_splits.csv` | `t_width_all` |
| 8.6, 8.16 Break-aware minus fixed at equal width | `coverage_cis.csv` (rows "... minus ...") | `t_equal_width` |
| 8.7 One common width | `extras_bootstrap_and_pooled_width.csv` | `t_pooled_width` |
| 8.8, 8.17 Calibrated parameters, implied lambda | `split_params.csv`, `extras_bootstrap_and_pooled_width.csv` | `t_selected` |
| 8.9 Detector comparison | `table_8_7_detector_comparison.csv` | `t_detect` |
| 8.10 Detection by shift size | `table_detection_power_by_shift_size.csv` | `t_power` |
| 8.11, 8.18 Pooled controls | `table_8_7b_pooled_controls_detection.csv` | `t_pooled` |
| 8.12 Alarm and interval by scenario | `all_results.csv`, `coverage_cis.csv` | `t_diag_scen` |
| 8.13 Point-estimate correction | `table_8_8_point_correction.csv` | `t_pc` |
| 8.14 Bias / MAE by method | `table_bias_mean_over_splits.csv`, `table_mae_mean_over_splits.csv` | `t_err` |
| 8.15 DMart coverage | `coverage_cis.csv` | `t_cov_ci('D')` |
| 8.19 Summary | several | `cross_table` |
| Figure 8.1 / 8.4 | `fig8_8_frontier.png` | |
| Figure 8.2 | `fig8_7_calibration.png` | |
| Figure 8.3 / 8.5 | `fig8_9_scenarios.png` | |

The pipeline's output file names keep the table numbers of the old draft (`table_8_7_...`, `table_8_7b_...`, `table_8_8_...`);
use the table above for the numbers in the paper.

## Points worth checking
* **Pooled controls (paper Table 8.11).** The code for this table was not in the old repository; the rule is reconstructed
  and stated in paper Section 4.8: flag when |mean z of K controls - calibration mean| > t(.975, n-1) * s * sqrt(1/K + 1/n);
  six draws per test store. It gives 63.6 / 83.8 / 97.5% detection of a 5% shift for 3 / 5 / 10 controls and 4.8% false alarms
  for 3 controls. The old draft's 62.9 / 83.4 / 96.8% and 5.72% came from code that is not in the repository.
* **Point-estimate correction (paper Table 8.13, Section 8.9).** Walmart, 5% common shift: mean absolute error 2.15 -> 2.93
  million (+36.4%). It is applied in 35.4% of those cases; on Walmart it raises the error in every scenario.
* Primary split: split 0 (23 test stores). Seeds: splits 0-19, bootstrap 12345, DMart grouping 2024.
