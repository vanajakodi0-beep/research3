# An equal-width benchmark and early-warning diagnostic for post-intervention shifts

Google Colab notebook (`RESEARCH5.ipynb`) that reproduces every table and figure in Section 8 of the paper.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR_USERNAME/YOUR_REPO/blob/main/RESEARCH5.ipynb)

Causal-impact estimates (BSTS) can fail silently when business conditions change after an intervention. A negative-control series, which the intervention should not affect, can warn of such a shift and has been proposed as a basis for widening the interval. This project asks whether widening in proportion to the control's drift beats widening by a fixed multiplier **at the same mean width**.

## Main findings

- **Equal-width benchmark:** the break-aware interval (`1.96 + λ|z|`) covered 0.8 to 4.1 points less than fixed widening on Walmart and 1.0 to 18.2 points less on DMart. It is also blind to a shift confined to the treated unit (Scenario A).
- **Early-warning diagnostic:** a year-over-year growth z-score of a control series, with a conformal threshold calibrated to 5% false alarms, detects a 5% shift in 35.4% of Walmart cases with one control, and 97.5% with ten pooled controls (AUC 0.99).
- **Point correction does not help:** removing the estimated shift when the alarm fires raised mean absolute error by 36.4% in the common-shift scenario.

| Walmart (mean over 20 splits) | No shift | 5% common shift |
|---|---|---|
| Standard BSTS coverage | 75.2% | 68.5% |
| Break-aware coverage (width x6.0 under shift) | 96.0% | 95.8% |
| Fixed widening at break-aware width | 97.2% | 97.4% |

## Repository contents

| File | Purpose |
|---|---|
| `RESEARCH5.ipynb` | The full pipeline as 11 Colab cells |
| `Walmart_Sales.csv` | **Not included.** Download separately (see Data) |
| `DMart_sample_data.csv` | **Not included.** Download separately (see Data) |

## Data

- **Walmart:** 45 stores x 143 weeks, intervention after week 100 (43 post weeks). Kaggle: `mikhail1681/walmart-sales`.
- **DMart:** `DMart_sample_data.csv` has no store identifier, so 45 pseudo-units are built by randomly assigning the 400 customer names to 45 groups (fixed seed) and summing weekly `Total`. This gives 105 complete Monday-Sunday weeks, with the intervention after week 75 (30 post weeks).

> **Caution:** the DMart file has the statistical signature of generated data (uniform marginals, flat weekly totals, no trend or seasonality). It is used as a structure-free stress test, **not** as independent real-world evidence.

## How to run (Google Colab)

1. Click the **Open in Colab** badge above (or upload `RESEARCH5.ipynb` at <https://colab.research.google.com>).
2. Run the cells in order (**Runtime > Run all** works).
3. When Cell 9 asks, upload `Walmart_Sales.csv` and `DMart_sample_data.csv`. They are read from `/content/`.
4. Cell 10 shows a figure and downloads `results_zip.zip` with all outputs.

With the default Kalman engine the full run (both datasets, 20 splits) took about 15 seconds per dataset in Colab.

### Notebook cells

| Cell | Content |
|---|---|
| 0 | Imports |
| 1 | Constants, grids, dataset geometry, method labels |
| 2 | Data loaders (Walmart, DMart, synthetic smoke-test panel) |
| 3 | Diagnostics (year-over-year z-score, Newey-West) and scenario generator |
| 4 | BSTS engines: Kalman local-level (default) and tfcausalimpact |
| 5 | Helpers, calibration grid selection, splits, DiD and synthetic control |
| 6 | Main experiment `run()` |
| 7 | Coverage confidence intervals (cluster bootstrap) |
| 8 | Tables and figures |
| 9 | Upload data, settings and run |
| 10 | Show figure and download results |

### Settings (top of Cell 9)

| Setting | Default | Description |
|---|---|---|
| `ONLY` | `None` | `None` runs both datasets; or `"walmart"` / `"dmart"` |
| `ENGINE` | `"kalman"` | `"kalman"` (maximum-likelihood local level, deterministic, fast) or `"tfci"` (tfcausalimpact, slow; run `!pip install tfcausalimpact` first) |
| `SCENARIOS` | all seven | Must include `"none"` and `"common5"` |
| `SPLITS` | 20 | Random calibration/test splits (22 / 23 stores). Use 2 for a quick test |
| `SEED` | 0 | `tfci` only |
| `TFCI_PAIRS` | `"primary"` | `tfci` only: fit control-store models for the primary split or all splits (very slow) |
| `SMOKE` | `False` | `True` uses synthetic data, for code testing only. Never use its output in the paper |

> The loaders in Cell 2 read the CSVs from fixed `/content/...` paths. To run outside Colab, change those paths.

## Method summary

- **Scenarios:** known 3% treatment effect injected into post-period sales, plus seven shift scenarios: no shift, 5% common shift, A (treated only), B (size per store), C (slow gradual), D (control leakage), D2 (leakage + 5% shift).
- **Diagnostic:** change in mean year-over-year log growth of the control after the intervention, divided by a Newey-West standard error.
- **Alarm threshold:** conformal `(n+1)` rule on the calibration control stores under no shift.
- **Calibration:** `λ` (break-aware) and `k` (fixed multiplier) are the smallest grid values reaching at least 95% coverage in both the no-shift and 5% common-shift calibration scenarios.
- **Comparators:** standard BSTS, BSTS with random or best control, fixed widening, fixed widening at break-aware width (oracle), conformal (scaled and absolute), DiD, and synthetic control.
- **Uncertainty:** store-cluster bootstrap intervals over the splits.

## Outputs

Written to `results/<dataset>/` (zipped by Cell 10):

| File | Content |
|---|---|
| `all_results.csv` | Per-split, per-scenario, per-method coverage, width, MAE and bias |
| `table_coverage_pct_*.csv`, `table_width_vs_standard_*.csv`, `table_mae_*.csv`, `table_bias_*.csv` | Method x scenario tables (primary split and mean over splits) |
| `coverage_cis.csv` | Coverage with 95% bootstrap intervals, including break-aware minus fixed differences |
| `table_8_7_detector_comparison.csv` | Diagnostic false-alarm rate, detection and AUC by threshold rule |
| `table_8_7b_pooled_controls_detection.csv` | Detection with 1, 3, 5, 10 pooled controls |
| `table_detection_power_by_shift_size.csv` | Detection by shift size |
| `table_early_warning_summary.csv` | Compact early-warning summary |
| `table_8_8_point_correction.csv` | Point-correction results |
| `standard_bsts_diagnostics.csv`, `worked_example.csv` | Standard-interval diagnostics and the worked example |
| `frontier.csv`, `calibration_curves.csv`, `split_params.csv` | Coverage-width frontier, calibration curves, selected `k` and `λ` per split |
| `fig8_7_calibration.png`, `fig8_8_frontier.png`, `fig8_9_scenarios.png` | Figures |
| `run_config.json` | Engine, grids, data SHA-256, library versions |

## Reproducibility

- Every table comes from the same data loader, scenario generator, diagnostic, store splits, calibration grids, threshold rule and evaluation code. Only the BSTS engine can differ between runs.
- The Kalman engine is deterministic; splits use seed = split index. `run_config.json` records the SHA-256 of the input data and package versions.
- The notebook stops with an error if a real CSV is missing; it never silently substitutes synthetic data.

## Limitations

- The DMart panel is constructed and appears synthetic, so it does not count as a second real intervention. A real second dataset (for example Favorita) remains the main outstanding replication.
- The 20 splits overlap, so they are not 20 independent experiments.
- Bootstrap intervals do not include variability of the calibrated `λ` and `k` across splits; this is reported separately.

## Requirements

Python 3.9+, `numpy`, `pandas`, `scipy`, `matplotlib` (all preinstalled in Colab). Optional: `tfcausalimpact` for `ENGINE = "tfci"`.

## Citation

```bibtex
@misc{yourname_equalwidth,
  title  = {An equal-width benchmark and early-warning diagnostic for post-intervention shifts},
  author = {Your Name},
  year   = {2026}
}
```

## License

Add a license (for example MIT) before publishing.
