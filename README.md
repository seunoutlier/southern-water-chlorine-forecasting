# southern-water-chlorine-forecasting
Southern Water Chlorine Forecasting

Forecasting and anomaly detection on Southern Water's 2016–2026 open
drinking-water-quality dataset, at LSOA granularity.

A single global LightGBM model that predicts weekly mean free chlorine
residual across 1,566 LSOAs in southern England, beating a persistence
baseline by 28% on RMSE on the held-out 2025–26 year. Built on the
open dataset published by Southern Water on the
Stream platform in 2026.
TL;DR

Data: ~2.5M sample records, 433 determinands, 1,566 LSOAs, Jan 2016 – Apr 2026
Task: weekly free-chlorine residual forecast + residual-based anomaly detection
Model: global LightGBM with LSOA as a categorical embedding, lag/rolling features, co-determinand covariates
Headline: MAE 0.103 mg/L · RMSE 0.530 mg/L · 28.4% RMSE improvement over a lag-1 persistence baseline on 10,280 held-out predictions
Anomalies: 14 readings flagged at |z| > 3 in the test window
Most surprising finding: the LSOA categorical ranks above pH, turbidity, and temperature in feature importance — spatial heterogeneity carries more signal than co-determinand chemistry at weekly granularity

Background
In June 2026, Southern Water published the most extensive UK drinking-water-quality open dataset to date on Stream — ten years of domestic sample results across their regional network, including both DWI-regulated and non-regulated determinands. The release was driven by feedback from Stream's Collaborating on Lead project, which showed that access to non-regulated determinands meaningfully increases value for external research.
This project tests how predictable free chlorine residuals are at small-area (LSOA) granularity. Chlorine residual management is the operational workhorse of every water utility: too low risks microbial breach, too high triggers taste complaints and wastes chemical. A model that captures normal behaviour and flags deviations has direct operational value.
The project is also a methodological exercise: how well does modern panel-time-series ML handle a sparse, hierarchical, partially-censored real-world environmental dataset?
Data
PropertyValueSourceStream Water Data — Southern Water Domestic Drinking Water QualityDate range2016-01-04 to 2026-04-30Sample records2,467,050Unique determinands433Unique LSOAs1,566Files4 CSVs (~140 MB total)
The data is in long format: one row per measurement. Schema:
Sample_Date    | date the sample was collected
Determinand    | what was measured (e.g. "CHLORINE (FREE)", "LEAD")
DWI_Code       | regulator code (NaN for non-regulated determinands)
Units          | <, > or empty — actually a censoring flag, not a unit
Operator       | duplicate of Units
Result         | numeric measurement
LSOA           | Lower-Layer Super Output Area code (UK small geography)
ObjectId       | row identifier
Notable data quirks

The Units column is mislabelled. It contains only <, >, or NaN — these are censoring flags ("below detection limit" / "above detection limit"), not unit labels. The actual unit of measurement is implicit in the determinand name. The pipeline renames this to Censor_Flag.
Censoring is significant overall (13.3%), minor for free chlorine (1.2%). The model treats <LOD values as point estimates, which is a known bias — see Limitations.
The Operator column is empty in every row checked. Dropped.
The shapefile version distributed alongside the CSVs has empty geometries — only the LSOA code is available as a geographic key, not point coordinates (presumably for GDPR reasons since these are domestic samples).
Sampling density varies wildly per LSOA. Most LSOAs are sampled for free chlorine ~100 times across the decade; others fewer than 50. The pipeline filters to LSOAs with ≥ 50 chlorine samples.

Methodology
Modelling task
Target: weekly mean free chlorine residual per (LSOA, week).
Aggregation choice rationale: per-sample modelling would force the model to learn intra-week sampling noise; weekly aggregation focuses on the operational signal of interest (sustained chlorine levels in a small area).
Pipeline
Raw CSVs (4 files, ~2.5M rows)
    │
    ├── Parse dates, coerce results to numeric, normalise censoring flags
    │
    ▼
Filter to LSOAs with ≥ 50 free-chlorine samples
    │
    ▼
Pivot to (LSOA, week) panel
    Target: weekly mean free chlorine
    Covariates joined on same key:
        • Temperature
        • Turbidity
        • pH
        • Total chlorine
    │
    ▼
Feature engineering (per LSOA)
    • Lags 1, 2, 4, 8 weeks
    • 4-week rolling mean and std (lagged)
    • Calendar features: month, week-of-year, year
    • LSOA as label-encoded integer (categorical feature for LightGBM)
    │
    ▼
Time-based train/test split at 2025-01-01
    │
    ▼
LightGBM regressor with categorical_feature=['lsoa_id']
    │
    ▼
Persistence baseline: predict y_t = y_{t-1}
    │
    ▼
Evaluation: MAE, RMSE, feature importance
    │
    ▼
Anomaly layer: flag residuals where |z| > 3
Model choices and rationale
Why global LightGBM rather than per-series SARIMA/Prophet?
With 1,566 LSOAs and many sampled sparsely, per-series classical models would be underpowered. A single global model with LSOA as a categorical feature pools information across series while still learning per-LSOA effects — the standard approach in modern panel forecasting.
Why LSOA as a categorical instead of pre-defining supply zones?
Southern Water's water supply zone (WSZ) boundary shapefile is not bundled with the open data. Rather than guessing zones, LightGBM's native categorical splits let the model learn whatever spatial groupings actually matter for chlorine behaviour. The feature-importance ranking suggests this paid off (LSOA ranked 4th overall, above all co-determinand covariates).
Why a time-based split?
Random splits would leak future information through the LSOA categorical and through neighbouring weeks' lag features. Train pre-2025, test 2025–26 honestly evaluates forecast performance.
Results
Headline metrics
Held-out window: 2025-01-01 to 2026-04-30 · 10,280 predictions across the filtered LSOA set.
ModelMAE (mg/L)RMSE (mg/L)Lag-1 persistence baseline0.1470.739LightGBM (full feature set)0.1030.530Improvement29.8%28.4%
Feature importance
RankFeatureImportance1total_cl3822free_cl_roll4_mean1683free_cl_lag11104lsoa_id775free_cl_lag4626year627temp498ph479turb3510free_cl_lag83211free_cl_lag22812free_cl_roll4_roll4_std2713woy2214month15
Interpretation

total_cl dominates — but this is mechanically expected. Free chlorine + combined chlorine = total chlorine, measured in the same sample. Most of the model's lift on the held-out set is therefore nowcasting (imputing free Cl from a co-measured sister determinand) rather than forecasting in the strict sense. See Limitations.
free_cl_roll4_mean beats any single lag. Recent history is more informative than any one point in that history — a clean argument for the rolling-statistic feature.
lsoa_id ranks above all co-determinand covariates. Spatial heterogeneity in chlorine behaviour carries more signal than the chemistry covariates included here. The spatial fingerprint of a network — treatment-works zone, pipe material, residence time — apparently encodes more than pH/turbidity/temperature can.
Temperature ranks 7th, despite chemistry suggesting it should dominate chlorine decay rates. A plausible reading: operators already adjust dosing seasonally, so the temperature signal has been absorbed by operational practice before it reaches network sample points. If true, that means climate impacts will manifest as changes in operational cost and intervention frequency, not as visible signals in the sample data itself.

Anomaly detection
Flagged 14 readings (0.14% of test predictions) at |z| > 3 — distributed across LSOAs, with both extreme high and extreme low residuals. Validation against publicly reported DWI incident records is the obvious next step but is not yet implemented in this project.
Limitations
This project is a baseline, not a deployment. Known weaknesses:

The total_cl feature confounds forecasting vs. nowcasting. A fair forecasting evaluation would exclude this and any other same-sample co-determinand. With those features removed, expected performance is meaningfully lower than the headline 28% improvement — the rolling-mean and lag features alone are doing more modest work.
Censored values are treated naively. 13.3% of the broader dataset (only 1.2% for free Cl specifically) is left-censored at the detection limit. The model treats these as point values, which biases predictions downward in censored ranges. A Tobit-style loss or interval-censored objective would be the proper handling.
No prediction intervals. Point forecasts are operationally less useful than quantile forecasts (P10/P90), since the low tail is what actually drives intervention. Quantile LightGBM is a one-line extension and a known next step.
No hierarchical reconciliation. Forecasts at LSOA level don't sum coherently to higher aggregations. For operational use, hierarchical reconciliation (hierarchicalforecast, MinT) would matter.
Anomaly threshold is arbitrary. |z| > 3 was chosen for simplicity. Without ground-truth labels (DWI incidents cross-referenced), the false-positive rate is unknown.
No causal interpretation. Feature importance shows associations, not causes. The "operators already adjust dosing seasonally" interpretation of temperature ranking is a hypothesis, not a finding.

Reproducing
Prerequisites

Python 3.10+
Google Colab (recommended) or a local Jupyter environment with ~4 GB RAM available
The four CSV files from Stream — see Data access below

Data access
Download the four CSV files from Southern Water's Stream open data platform:

Southern_Water_Domestic_Drinking_Water_Quality_2016_2018_part1.csv
Southern_Water_Drinking_Water_Quality_2018_2022_part1.csv
Southern_Water_Drinking_Water_Quality_2022_2026_part1.csv
Southern_Water_Domestic__Drinking_Water_Quality_2022_2026_part2.csv

Note the double underscore in file 4 — that's the original filename, not a typo.
Running on Google Colab

Open southern_water_chlorine_forecasting.ipynb in Colab via File → Upload notebook.
Upload the four CSVs to either your Google Drive (recommended for persistence) or to /content/sample_data/ (wiped on each restart).
Adjust the file paths in Cell 3 if needed.
Run cells 1 through 11 in order.

Runtime
End-to-end run on Colab's free tier (no GPU needed) is roughly:
StageTimeData load (4 CSVs)~1–2 minFeature engineering~30 sLightGBM training (early stopping)~30–60 sEvaluation + plots~10 s
Dependencies
All come pre-installed in Colab. For a local environment:
bashpip install lightgbm pandas numpy scikit-learn matplotlib seaborn
Repository structure
.
├── README.md
├── southern_water_chlorine_forecasting.ipynb   ← main pipeline (11 cells)
└── data/                                       ← (gitignored) put CSVs here
    ├── Southern_Water_Domestic_Drinking_Water_Quality_2016_2018_part1.csv
    ├── Southern_Water_Drinking_Water_Quality_2018_2022_part1.csv
    ├── Southern_Water_Drinking_Water_Quality_2022_2026_part1.csv
    └── Southern_Water_Domestic__Drinking_Water_Quality_2022_2026_part2.csv
The raw CSVs are not included in the repository — download them from Stream.
Next steps
In rough order of value:

Fair forecasting evaluation. Re-run without total_cl (and any other same-sample co-determinand features) to report honest forecasting performance separately from the current nowcasting numbers.
Quantile forecasts. Retrain with objective='quantile', alpha=0.1 and alpha=0.9 to get P10/P90 prediction intervals. Operationally far more useful than point forecasts.
Censored-value handling. Implement a Tobit-style loss or compare against interval-censored treatments.
Anomaly validation. Cross-reference the 14 flagged anomaly dates/LSOAs against publicly reported DWI incidents.
Derived supply zones. Cluster LSOAs on their chlorine fingerprints (HDBSCAN on per-LSOA seasonal features) to recover data-driven zones, then compare against Southern Water's published WSZ boundaries if those become available.
Hierarchical reconciliation. Apply MinT or similar so LSOA-level forecasts are coherent with zone-level and network-level forecasts.
Extend to other determinands. The same pipeline should generalise to turbidity, pH, and any other continuous determinand with reasonable sample density.

Acknowledgments

Southern Water for publishing the dataset openly and at this scale.
Stream for hosting the open-data platform and the Collaborating on Lead project that drove the inclusion of non-regulated determinands.
Daniel Slidel for the dataset release announcement that prompted this project.

License
This project's code is released under the MIT License — see LICENSE for details.
The underlying data is published by Southern Water under their own open-data terms via the Stream platform. Refer to Stream's licensing notes for data reuse conditions.
Contact
[Add your name, GitHub handle, LinkedIn URL here]
