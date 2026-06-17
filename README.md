# Southern Water Chlorine Forecasting

Weekly free chlorine residual forecasting across Southern Water's drinking water network, using LightGBM on the Stream open data platform. A ten-year, network-wide study of where chlorine variance actually lives in a regional UK water utility.

**Headline result:** At weekly granularity, the spatial identity of a sample location encodes more about chlorine behaviour than the co-determinand chemistry does. The Lower Layer Super Output Area (LSOA) categorical ranks first in feature importance, ahead of temperature, pH, and turbidity.

## The problem

Free chlorine residual is the operational lever utilities use to maintain microbial safety from treatment works to customer tap. Too little and the disinfection margin disappears; too much and it generates regulatory complaints and disinfection by-products. Operators want to predict where and when residual will drift, ideally early enough to act.

Most published chlorine decay work treats the problem as chemistry: temperature, organic load, pipe wall demand, residence time. That framing is correct at the bulk-fluid scale but it does not translate cleanly to a regional water network where 1,500+ distinct supply zones each have different treatment works, different pipe materials, different demand patterns, and different distances from the works.

This project asks a different question. Given a regional water utility's full ten-year sample record, what actually predicts next week's free chlorine residual in a specific neighbourhood: the chemistry, the recent history of that location, or the identity of the location itself?

## The data

Southern Water's drinking water quality dataset, released as open data on the Stream platform in June 2026.

| Metric | Value |
|---|---|
| Total sample records | 2,132,425 |
| Unique determinands | 422 |
| Unique LSOAs (sampling locations) | 1,566 |
| Date range | 2016-01-04 to 2026-04-30 |
| Free chlorine samples | 133,242 |
| Below detection limit (overall) | 13.4% |
| Below detection limit (free chlorine only) | 1.2% |

The platform also includes Severn Trent, Anglian, and Affinity Water datasets, which makes a cross-utility transfer learning extension natural follow-on work (see Next steps).

## Approach

A single global LightGBM model trained across all LSOAs simultaneously, with LSOA passed in as a categorical feature so the model can learn per-location structure without needing a separate model per location.

**Forecasting framing.** Predict next week's mean free chlorine residual at each LSOA, conditional on lagged values of that LSOA's free chlorine history and same-week co-determinand chemistry. Train on 2016-01 to 2024-12. Test on 2025-01 to 2026-04 (true forward holdout, no shuffling).

**Features.**

- `lsoa_id` (categorical): label-encoded LSOA identity
- `temp`, `turb`, `ph`: same-week LSOA mean of co-determinand chemistry
- `free_cl_lag1`, `free_cl_lag2`, `free_cl_lag4`, `free_cl_lag8`: lagged free chlorine at the same LSOA
- `free_cl_roll4_mean`, `free_cl_roll4_std`: 4-week rolling stats of free chlorine, shifted to avoid leakage
- `month`, `woy`, `year`: seasonality terms

**LSOA filter.** Only LSOAs with at least 50 historical free chlorine samples are kept. This drops sparse locations where the lagged features would be unreliable. Result: 1,332 LSOAs retained, 57,679 modelling rows after lag construction.

**Why LightGBM.** Gradient-boosted trees handle high-cardinality categoricals natively, are robust to missing covariates (turbidity and pH are missing in roughly half of free-chlorine sampling weeks), and produce interpretable feature importance that is meaningful for an operational audience.

## Results

Test set: 10,034 predictions over 2025-01-01 to 2026-04-30.

| Model | MAE (mg/L) | RMSE (mg/L) | RMSE improvement over persistence |
|---|---|---|---|
| Persistence baseline (next week = this week) | 0.147 | 0.748 | reference |
| LightGBM, honest forecaster | 0.145 | 0.549 | **26.6%** |

### Feature importance

The 13-feature honest model, sorted by LightGBM gain importance:

| Rank | Feature | Importance |
|---|---|---|
| 1 | `lsoa_id` | 224 |
| 2 | `temp` | 162 |
| 3 | `free_cl_lag1` | 129 |
| 4 | `free_cl_roll4_mean` | 117 |
| 5 | `free_cl_lag8` | 97 |
| 6 | `free_cl_lag4` | 82 |
| 7 | `turb` | 37 |
| 8 | `ph` | 35 |
| 9 | `free_cl_lag2` | 34 |
| 10 | `woy` | 29 |
| 11 | `month` | 21 |
| 12 | `year` | 14 |
| 13 | `free_cl_roll4_std` | 11 |

## Key findings

### 1. Spatial identity dominates over chemistry

LSOA ranks first, comfortably ahead of every chemistry covariate. At weekly granularity, the network identity of a sampling location (treatment works zone, pipe material, residence time, supply geometry) encodes more about chlorine behaviour than the same-week co-determinand chemistry does. This is the operational reading: where you sample matters more than what else is in the sample.

This is the finding the LinkedIn writeup of this project led with. It holds in the honest model.

### 2. Temperature ranks second, not low

Temperature ranks second behind only the LSOA categorical. This is the chemistry-consistent result: chlorine decay is temperature-driven, and at weekly granularity that signal survives operational dosing adjustments.

(An earlier version of this analysis included same-sample total chlorine as a feature, which dominated the ranking and pushed temperature to 7th. That version was reported in the initial LinkedIn post. See Methodological notes below.)

### 3. The model beats persistence on large errors, not on typical weeks

MAE is essentially tied with persistence (0.145 vs 0.147). RMSE is 26.6% better. The gap means the model adds value specifically on the cases where chlorine deviates substantially from last week's value, which is operationally the cases that matter. Predicting that next week looks like this week is easy. Predicting where it will not is where forecasting earns its keep.

## Methodological notes

### Same-sample total chlorine: a leakage caveat

An earlier version of this model included `total_cl` (same-week LSOA mean total chlorine) as a feature. Because free chlorine plus combined chlorine equals total chlorine, and total is measured from the same sample as free, this feature was effectively nowcasting rather than forecasting: at prediction time in a real operational setting, the utility would not have access to next week's total chlorine.

Removing the leaky feature:

- MAE moved from 0.103 to 0.145 (degraded, as expected)
- RMSE moved from 0.530 to 0.549 (almost unchanged)
- LSOA stayed at rank 1
- Temperature moved from rank 7 to rank 2 (the chemistry-consistent ranking)

The small RMSE delta is the operationally interesting part: the project's headline forecasting value did not depend on the leaky feature. The spatial-identity finding survived. The temperature ranking became more honest.

Current notebook contains only the honest 13-feature model.

### Anomaly detection (preliminary)

Test-set residuals were z-scored against the residual standard deviation, with |z| > 3 flagged as candidate anomalies. The honest model flagged 5 anomalies across 10,034 test predictions (0.05%). The threshold is z-score based and therefore moves with the model's overall error; an absolute-residual threshold would give a more directly comparable count across model versions. Validation against Drinking Water Inspectorate (DWI) incident records is the obvious next step (see below).

### Censoring

13.4% of all sample readings sit below the laboratory limit of detection across the full dataset, but only 1.2% of free chlorine readings do. The model uses point values; below-LOD imputation strategies (Tobit regression, multiple imputation) are not currently implemented, since free chlorine censoring is rare enough not to drive the result.

### What the model does not do

- It does not predict at sub-weekly resolution.
- It does not predict at the individual sampling-point level inside an LSOA.
- It does not condition on planned operational events (mains flushing, treatment works diversions).
- It does not handle the full censoring structure rigorously.

These are not flaws in the present scope but boundaries of what this version was built to do.

## Repo structure

The notebook is the single source of truth. Cells run top to bottom in Google Colab in under five minutes from a cold start on the Stream CSV files.

## How to run

1. Download the four Southern Water drinking water quality CSV files from the Stream platform.
2. Upload them to your Colab session at `/content/sample_data/`, or update the paths in the file-list cell.
3. Run all cells.

Required packages: `lightgbm`, `pandas`, `numpy`, `scikit-learn`, `matplotlib`, `seaborn`. The first cell installs them.

## Next steps

In rough priority order:

**Cross-utility transfer learning.** The Stream platform hosts equivalent open datasets from Severn Trent, Anglian, and Affinity Water. The natural test of generalisation is: does an LSOA-first model trained on Southern Water transfer to a different utility's network, and does fine-tuning on a smaller target-utility sample close any gap? This is the strongest follow-on extension scientifically and the one that turns a single-utility result into a pan-industry methodology.

**DWI anomaly validation.** Match the residual anomalies against publicly available Drinking Water Inspectorate incident records and compliance event reports. If even a small fraction of flagged residuals correspond to documented operational events, the project becomes evidence of operational utility, not just a forecasting exercise. Use absolute-residual thresholds rather than z-scores for this, so the count does not move with the model's overall error.

**Feature importance robustness.** Multi-seed runs to confirm the LSOA-above-chemistry ranking is stable across random initialisation, plus permutation importance as a cardinality-bias check on the categorical.

**Sub-LSOA resolution.** A finer-grained version that operates at supply-zone or DMA (district metering area) level, conditional on the data being available.

**Operational deployment scaffolding.** A simple inference API and a weekly batch job that ingests new Stream data and produces a per-LSOA forecast and anomaly flag.

## Credits

This work uses Southern Water's drinking water quality data, released as open data on the Stream platform. Thanks to Daniel Slidel and the Stream team for making this kind of analysis possible. The LSOA-above-chemistry finding could not have been formulated, let alone tested, on a smaller or differently aggregated dataset.

## Author

Oluwaseun Franklin Olabode, PhD. Hydrogeologist (Aberdeen). Working on machine learning for water resources, with a focus on data-scarce regions and operational decision support.

- GitHub: seunoutlier
- LinkedIn: oluwaseun-franklin-olabode-phd-6baaaa1a6
- Google Scholar: https://scholar.google.com/citations?hl=en&user=LslmGiUAAAAJ&view_op=list_works&sortby=pubdate

## License

MIT.
