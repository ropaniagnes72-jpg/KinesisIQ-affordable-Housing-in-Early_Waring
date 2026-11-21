# KinesisIQ — Affordable Housing Early Warning
Toronto affordable housing early warning - KinesisIQ team project-pod3


pod3-kinesisiq/work flow
├── README.md
├── requirements.txt
├── notebook.ipynb                  
├── kinesis.db                      ← SQLite output
├── docs/
│   ├── MEASURES.md                 ← deliverable 3   
│   ├── REPORT.md                   ← deliverable 4   
│   └── VALIDATION.md               ← validation evidence 
├── figures/
│   ├── affordable_loss_by_year.png
│   ├── pipeline_vs_demolition_by_ward.png
│   ├── five_year_forecast.png
│   ├── build_lag_distribution.png
│   ├── feature_importance.png      ← to generate
│   └── ward_typology_map.png
└── slides/
    └── kinesisiq_pod3.pptx         ← deliverable 4   ← MISSING
# Pod 3 — KinesisIQ Affordable Stock Depletion & Developer Responsiveness

## Scope
Pod 3 covers Project 2 (affordable housing depletion) and Project 6 (developer
responsiveness) of the KinesisIQ programme.

## Deliverables included
1. ETL pipeline: notebook cells 0–C2, writes to `kinesis.db::events`
2. Feature engineering: documented in `docs/MEASURES.md`
3. ML models: HC3 regression, Random Forest, K-means, Monte Carlo scenario
4. Findings: `docs/REPORT.md` + `slides/kinesisiq_pod3.pptx`

## How to run
Open `notebook.ipynb` in Google Colab. Run all cells top-to-bottom.
The notebook downloads data from the City of Toronto CKAN API and writes
`kinesis.db` locally (or to Google Drive if mounted).

## Data sources
All from Toronto Open Data CKAN:
- Demolition and Replacement of Rental Housing Units
- Active Affordable and Social Housing Units
- Affordable Rental Housing Pipeline
- Apartment Building Evaluation (RentSafeTO)
- Development Applications
- Building Permits (Active + Cleared)
- City Wards (25-ward model)
- Neighbourhoods (158 model)

## Validation summary
See `docs/VALIDATION.md`.
## Validation Summary

**Pipeline Notebook:** `Pod3_KinesisIQ_finalRiipen_project.ipynb` | **Database:** `kinesis.db`  
**Overall Audit Status:** **27 PASS** | **1 WARNING** | **0 CRITICAL FAILURES**

### 1. Automated ETL & Schema Checks (Section D)

| Check Category | Verification Rule | Result | Status |
| :--- | :--- | :--- | :--- |
| **Row Counts & Deduplication** | Single authoritative resource loaded per CKAN package; exact duplicates removed (`FOLDERRSN` unique on Development Applications; latest `PERMIT_NUM` revision kept on Permits; `Project ID` unique on Pipeline; latest `RSN` kept on RentSafeTO). | `8,639` unique dated applications (from `26,455` raw rows), `57,292` cleared permit events, `277` pipeline projects, `3,611` RentSafeTO buildings. | **PASS** |
| **Primary Keys & Idempotency** | `event_id` unique across `kinesis_events`; rerunning replaces rows by `source_dataset` without duplicating records. | Zero duplicate `event_id` keys (`DEMO_`, `DEVAPP_`, `PERMIT_A`, `PERMIT_C`). | **PASS** |
| **Date Hygiene** | No invented default dates; out-of-range dates (`< 1990` or `> AS_OF + 1 yr`) and negative or `> 10-year` (`3,650`-day) lags set to `NaT`/`NaN`. | `1,048` invalid approval lags and `66,058` invalid build lags nulled; undated rows excluded from `kinesis_events`. | **PASS** |
| **Outlier Filtering** | Single permits exceeding `5,000` units or `>= 500` units at `< $10,000`/unit zeroed out and flagged `outlier_units`. | `2` implausible permits zeroed and flagged (e.g., `00 363746 CMB` with `637` units at `$0` cost). | **PASS** |
| **Ward Reconciliation** | Ward-level sums in `ward_depletion` equal dataset totals for mapped records. | Demolition (`4,311`), Pipeline (`21,876`), and RentSafeTO (`3,611`) reconcile with zero discrepancy. | **PASS** |

### 2. Geospatial & Cross-Source Agreement Checks

| Dataset / Bridge | Method | Agreement / Match Rate | Threshold | Status |
| :--- | :--- | :--- | :--- | :--- |
| **Affordable Housing Pipeline** | Point-in-polygon vs. source `Ward Number` | **100.0%** (`277 / 277` rows) | `>= 95.0%` | **PASS** |
| **RentSafeTO Evaluations** | Point-in-polygon vs. source `WARD` | **100.0%** (`3,442 / 3,442` rows) | `>= 95.0%` | **PASS** |
| **Address Points Geography Bridge** | Permit `GEO_ID` joined to `ADDRESS_POINT_ID` (`525,035` points) | **97.4%** `GEO_ID` match (`88.6%` coordinates resolved) | `>= 90.0%` | **PASS** |
| **WARD_GRID Out-of-Sample Lookup** | Grid lookup trained on coordinate-bearing permits | **98.9%** held-out accuracy (`>= 20` rows, `>= 90%` purity) | `>= 95.0%` | **PASS** |
| **Development Applications** | Point-in-polygon vs. source `WARD_NUMBER` | **91.5%** agreement | `>= 95.0%` | **WARNING** |

> **Note on the 1 Warning (`91.5%` Development Applications ward agreement):** Older application records carry pre-2018 ward boundaries or boundary-straddling parcels; spatial point-in-polygon against the post-2018 25-ward layer (`dim_ward`) takes precedence per notebook design rules.

### 3. Predictive Model Back-Test & Calibration Checks

* **Project 2 Rolling-Origin Back-Test (`2020–2025`, Section B6):**
  * **Pooled Mean (selected):** `1.63` units/ward-year `MAE` (`5.25` `RMSE`) vs. `"Zero loss (baseline)"` (`1.30` `MAE`, `5.16` `RMSE`) and `"Last year (baseline)"` (`2.26` `MAE`, `8.72` `RMSE`).
  * **Interval Calibration:** Simulated 10th–90th percentile intervals (nominal `80%` coverage) captured **89%** of actual ward-year outcomes across `2020–2025`.
* **Project 6 Time-Ordered Evaluation (`Sections C3–C4`):**
  * **Survival Analysis (Kaplan–Meier):** Right-censored unfinished permits retained (`908` days median ward build lag; `58` days median permit approval lag).
  * **Application Volume Forecast:** Outperforms seasonal-naive baseline on `MAE`; Poisson interval achieved **72%** empirical test coverage.

## Limitations
See `docs/REPORT.md` §7.

## Limitations

* **Sparse Historical Demolition Events:** Only **8%** of ward-years (`2017–2025`) recorded a non-zero net affordable rental loss (`92%` are zero). Because of this sparsity, no statistical point estimator beats a `"Zero loss"` baseline on average error (`1.63` vs. `1.30` units/ward-year `MAE`). Ward-level forecasts must be read as relative risk rankings with 10th–90th percentile uncertainty intervals rather than exact unit point predictions.
* **Snapshot Pipeline & Delivery Assumptions:** The *Affordable Housing Pipeline* (`277` projects; `21,876` non-completed units) is a point-in-time snapshot without historical status transitions. Stage delivery probabilities (`95%` under construction, `70%` approved, `40%` planning/other) and delivery windows (`1–3`, `2–5`, and `3–7` years) are explicit planning assumptions tested via `0.5×` and `1.25×` sensitivity scenarios rather than empirically calibrated transition rates.
* **Replacement Commitments vs. Completed Delivery:** In the *Demolition and Replacement* dataset (`4,311` demolished vs. `4,551` replaced), replacement figures reflect approved/permitted commitments rather than verified occupancy, and blank replacement counts are stored as `0`.
* **No Ward Denominator for City-Wide Active Stock:** *Active Affordable and Social Housing Units* (`13,163` units in Q2 2026) is published solely as a city-wide quarterly series with no ward or address fields, preventing normalization of ward-level deficit probabilities against total existing ward stock.
* **No Causal Link Between RentSafeTO Scores and Unit Loss:** While **430 of 3,611** evaluated RentSafeTO buildings (`11.9%`, `41,478` units) are flagged on the watch list (`score < 70` or `>= 5`-point drop), no public dataset links low evaluation scores to subsequent demolition or permanent unit removal. Additionally, `11,759` pre-2023 rows lacking compatible scores/dates were excluded to keep `2023–2026` evaluations on one rubric scale.
* **Address-Based Execution Matching & Text-Parsed Application Units:** Matching development applications to building permits via normalized street address (`addr_key`) captures **6.1%** of housing-related applications, serving as a lower bound because multi-address sites or phased parcels do not share a single street number. Similarly, dwelling unit counts extracted from *Development Applications* descriptions (`43%` parsed) are regex proxies rather than audited zoning counts.
* **Under-Dispersed Application Volume Intervals & Boundary Shifts:** The quarterly application volume forecast outperforms the seasonal-naive baseline on `MAE`, but its Poisson prediction interval achieved **72%** empirical test coverage (understating variance). *Development Applications* also showed **91.5%** agreement between spatial point-in-polygon ward assignment and the source `WARD_NUMBER` field due to pre-2018 ward boundary labels.



# Pod 3 — KinesisIQ Abstract Measure Definitions

**Author:** [YOUR NAME]
**Pod:** 3
**Projects:** 2 (Spatial Forecasting of Affordable Housing Depletion), 6 (Developer Responsiveness & Supply Forecasting)
**Last updated:** [DATE]

## Purpose of this document

KinesisIQ engineering will rebuild these measures inside the production repository.
This document defines every engineered feature used in the Pod 3 models
**independently of the authoring code**, so the measures can be re-implemented from
the source datasets alone.

Each measure follows the same structure:

- **Definition** — what the measure represents in plain language.
- **Formula** — the exact computation.
- **Unit** — the measurement unit.
- **Source** — the dataset(s) and field(s) required.
- **Population filter** — which records are eligible.
- **Null rules** — when the measure is undefined and must remain null (never zero).
- **Ward / neighbourhood assignment** — how geography is resolved.
- **KinesisIQ integration note** — guidance for production reuse.

A general principle applies to every measure below: **an unmeasured record is not
evidence of zero activity.** Where source coverage is incomplete, the measure is
left null rather than filled with zero. Wards or neighbourhoods with no mapped
records are reported as unknown, not as zero-valued.

## Shared geography rules (apply to all measures)

Two reference layers are used:

- **City of Toronto Wards, 25-ward model** (current since 2018-12-01).
  Enforced by asserting exactly 25 unique ward identifiers after loading the
  City Wards geometry.
- **City of Toronto Neighbourhoods, 158-model**.

Geography is assigned by the following priority, in order:

1. **Spatial join.** Where WGS84 latitude/longitude (or a GEO_ID mapped to
   coordinates via the Address Points Municipal Repository) is available and
   inside the Toronto bounding box (`lat 43.4–44.0`, `lon −79.8 to −78.9`),
   the record is point-in-polygon joined to ward and neighbourhood boundaries.
2. **Explicit source ward field.** Where no coordinates exist but the source
   carries a ward field, the field is decoded only if it is
   unambiguously a 25-ward value (integer 1–25, or an exact 25-ward name).
3. **Unmapped.** If neither applies, or if the record is dated before
   2018-12-01 and only a source ward field exists (pre-25-ward numbering
   cannot be reconciled safely), the record remains **ward-unmapped**.

Each record carries a `geo_method` label with one of:
`spatial_join`, `source_ward_field`, `ward_model_unverified`, `unmapped`.

Downstream, coverage is reported per dataset, and the **coverage gate**
(`MIN_GEO_COVERAGE = 0.80`) prevents any ward without a mapped record from
being zero-filled when the dataset's overall coverage falls below 80%.


# Project 2 — Spatial Forecasting of Affordable Housing Depletion

## M1 — `dem_aff`, `dem_tot`, `dem_mid`, `dem_high`

**Definition.** Counts of rental homes slated for demolition, split by affordability
band, as recorded in the Demolition and Replacement of Rental Housing Units dataset.

**Formula.**
- `dem_tot`  = `Total Rental Homes for Demolition`
- `dem_aff`  = `Affordable Rental Homes for Demolition`
- `dem_mid`  = `Mid-Range Rental Homes for Demolition`
- `dem_high` = `High-End Rental Homes for Demolition`

**Unit.** Integer count of dwelling units.

**Source.** City of Toronto Open Data — *Demolition and Replacement of Rental
Housing Units*. Fields: those named above.

**Population filter.** All rows in the dataset. The dataset records demolition
applications, not completed demolitions.

**Null rules.** A blank numeric field is treated as **zero homes reported**
(the source uses blanks to mean "none reported in this band"), which is the
dataset's own convention. Non-numeric cells are coerced to null and then
treated as zero under the same rule.

**Ward / neighbourhood assignment.** The dataset contains no coordinates. Ward
comes from the source ward field; pre-2018 values remain ward-unmapped.

**KinesisIQ integration note.** These values are input measures, not derived
ones. Re-extract whenever the City refreshes the dataset. No transformation
is applied to affordability-band counts.

## M2 — `rep_aff`, `rep_tot`, `rep_mid`, `rep_high`

**Definition.** Counts of rental homes **reported as replaced**, split by
affordability band.

**Formula.** Analogous to M1, using the *Replaced* fields.

**Unit.** Integer count of dwelling units.

**Source.** Same dataset as M1.

**Population filter.** Same as M1.

**Null rules.** Blanks mean "no replacement reported" → zero, matching the
source convention.

**Ward / neighbourhood assignment.** Inherited from the parent record.

**KinesisIQ integration note.** These are **reported** replacements, not
verified completed homes. Downstream models must not treat them as completed
construction. If the source later adds a status field, replacements should
be re-filtered to only those with a confirmed completion.

## M3 — `net_affordable_change`

**Definition.** Net change in affordable units on a demolition record:
affordable units reported replaced minus affordable units slated for demolition.

**Formula.** `net_affordable_change = rep_aff − dem_aff`

**Unit.** Integer units. Negative values indicate net loss.

**Source.** M1 and M2 fields.

**Population filter.** Any demolition record where both M1 and M2 are known
(which is always, under the null rules).

**Null rules.** Null if either component is null; never coerced to zero.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** This is a per-record measure; ward-level
aggregation happens downstream (see M5, M6).

---

## M4 — `affordable_replacement_ratio`

**Definition.** Ratio of affordable units reported replaced to affordable units
slated for demolition, for a single demolition record.

**Formula.**
`affordable_replacement_ratio = rep_aff / dem_aff`, defined only when
`dem_aff > 0`.

**Unit.** Unitless ratio. Ratio = 1.0 means parity; < 1 indicates loss; > 1
indicates gain.

**Source.** M1 and M2 fields.

**Population filter.** Records where `dem_aff > 0`.

**Null rules.** Null when `dem_aff = 0` (division undefined). Null when
`rep_aff` is null.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** Do not aggregate this ratio by ward using
mean or median directly — that mixes records with different denominators.
The ward-level affordable position is expressed via M5 and M6 instead.

## M5 — `aff_demolished`, `aff_replaced`, `demolition_cases` (per ward)

**Definition.** Ward-level sums of the above per-record measures.

**Formula.**
- `aff_demolished` = Σ `dem_aff` over records assigned to the ward
- `aff_replaced`   = Σ `rep_aff` over records assigned to the ward
- `demolition_cases` = count of demolition records assigned to the ward

**Unit.** Integer units / integer case count.

**Source.** M1, M2, aggregated by `ward_id`.

**Population filter.** Records with a verified ward (`geo_method ∈ {spatial_join,
source_ward_field}`). Unmapped and ward-model-unverified records are excluded.

**Null rules.** If the dataset's overall ward coverage ≥ 80%, a ward with
no demolition records receives **zero** for these measures (genuinely no
activity in the dataset). If overall coverage < 80%, unmapped wards remain
**null** because absence of records may reflect missing mapping rather than
absence of demolitions.

**Ward / neighbourhood assignment.** Ward from the parent record.

**KinesisIQ integration note.** Recompute whenever the demolition dataset is
refreshed. Report the ward coverage percentage alongside these measures so
readers know whether absence-of-record is informative.

## M6 — `net_affordable_change` (per ward), `pipeline_cover`

**Definition.** Ward-level net affordable change and the ratio of mapped
pipeline units to affordable demolitions.

**Formula.**
- `net_affordable_change` (per ward) = `aff_replaced − aff_demolished`
- `pipeline_cover` = `pipeline_units / aff_demolished`, defined only when
  `aff_demolished > 0`

**Unit.** Integer units (net change); unitless ratio (cover).

**Source.** M5, M7.

**Population filter.** Records with verified ward.

**Null rules.** `pipeline_cover` is null when `aff_demolished = 0`.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** `pipeline_cover` is a ratio of "planned
replacement" to "observed demolition". Values above 1 indicate that more
pipeline units exist than demolitions this period, which may include supply
unrelated to demolition replacement. Do not interpret as a guarantee of
replacement.

## M7 — `pipeline_units` (per ward), `pipeline_records` (per ward)

**Definition.** Total affordable pipeline units recorded for a ward, and
the number of pipeline records, after excluding completed/cancelled projects.

**Formula.**
- `counts_as_pipeline` = TRUE where `status` does not match
  `complet|cancel|withdraw|occup|closed|reject|refus` (case-insensitive)
- `pipeline_units` = Σ `units` over records with `counts_as_pipeline = TRUE`
- `pipeline_records` = count of records with `counts_as_pipeline = TRUE`

**Unit.** Integer units; integer record count.

**Source.** City of Toronto Open Data — *Affordable Rental Housing Pipeline*.

**Population filter.** Records with `counts_as_pipeline = TRUE` **and** a
verified ward.

**Null rules.** Ward without a mapped pipeline record receives **zero** only
if the pipeline dataset has ≥ 80% ward coverage. Otherwise null.

**Ward / neighbourhood assignment.** Spatial join where coordinates exist;
source ward field otherwise; unmapped preserved.

**KinesisIQ integration note.** The pipeline dataset does not distinguish
affordable from social projects. The combined total is reported. Where
KinesisIQ has a richer status field, re-run the `counts_as_pipeline` rule
against the new field and report how many units move in and out of the live
pipeline definition.

## M8 — `score`, `score_observed`, `at_risk` (per building)

**Definition.** RentSafeTO evaluation score per building, whether the score
was observed, and whether the building is flagged at risk.

**Formula.**
- `score` = `CURRENT_EVAL_SCORE` (numeric)
- `score_observed` = TRUE where `score` is not null
- `at_risk` = `score_observed` AND `score ≤ LOW_SCORE_THRESHOLD` (50)

**Unit.** Score out of 100; boolean.

**Source.** City of Toronto Open Data — *Apartment Building Evaluation
(RentSafeTO)*.

**Population filter.** Latest evaluation per building where the RentSafeTO
identifier (RSN) is available; exact duplicate source rows removed.

**Null rules.** Buildings with no observed score are **not** treated as
passing or failing; they remain `at_risk = FALSE, score_observed = FALSE`
and are excluded from at-risk share denominators.

**Ward / neighbourhood assignment.** Spatial join where coordinates exist;
source ward field otherwise; unmapped preserved.

**KinesisIQ integration note.** `LOW_SCORE_THRESHOLD = 50` is an analyst
assumption. If KinesisIQ has an official threshold, substitute and recompute
at-risk shares. `at_risk` is a building-condition indicator, **not** a
demolition-risk indicator.

## M9 — `pct_units_at_risk` (per ward)

**Definition.** Share of units in the ward that are inside at-risk buildings,
denominated by units inside scored buildings only.

**Formula.**
`pct_units_at_risk = at_risk_units / scored_units`, defined only when
`scored_units > 0`.

**Unit.** Unitless proportion (reported as %).

**Source.** M8, aggregated by `ward_id`.

**Population filter.** Building records with observed score and observed
unit count, assigned to a verified ward.

**Null rules.**
- Null when `scored_units = 0` (no scored buildings in the ward — coverage unknown).
- Zero when the ward has scored buildings but none are at-risk (genuine zero).

**Ward / neighbourhood assignment.** Inherited from building records.

**KinesisIQ integration note.** The denominator intentionally excludes
unscored buildings because including them would treat missing assessment
as if it were a pass. Any change to the denominator rule must be documented
and versioned.

## M10 — `net_flow_p10 / net_flow_p50 / net_flow_p90` (per ward, 5-year horizon)

**Definition.** Simulated 5-year net flow of affordable units per ward,
reported at the 10th, 50th, and 90th percentiles.

**Formula.**
- `pool = flat array of all ward-year net losses` (M3 aggregated by
  ward and year, from the analysis window 2017–present)
- For each ward `w` and each simulation draw `s`:
  - Draw 5 annual losses from `H[w]` with probability `wgt`, or from
    `pool` with probability `1 − wgt`, where
    `wgt = n_years / (n_years + POOL_K)` and `POOL_K = 3`
  - Cumulative loss after `h` years = sum of the first `h` draws
  - Pipeline gain after `h` years = `PIPELINE_REALISATION × pipeline_units_w ×
    min(h, PIPELINE_DELIVERY_YEARS) / PIPELINE_DELIVERY_YEARS`
  - Net flow = pipeline gain − cumulative loss
- Report percentiles across the 2,000 draws.

**Unit.** Units (net).

**Source.** M5, M7, plus simulation constants.

**Population filter.** Wards with mapped demolition history and mapped
pipeline units.

**Null rules.** Null if the ward has no mapped demolition records (only
possible when demolition coverage < 80%). Wards with historical data but
no pipeline receive zero pipeline gain, which is legitimate.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** This is a **scenario**, not a forecast.
The three assumed parameters that materially affect the result are:

- `PIPELINE_REALISATION = 0.50` (baseline pipeline delivery share)
- `PIPELINE_DELIVERY_YEARS = 5` (linear delivery ramp)
- `AT_RISK_ANNUAL_LOSS_RATE = 0.01` (stress-scenario annual loss from
  scored at-risk units)

KinesisIQ should re-run the simulation with organisationally agreed
assumptions and version the scenario with its parameter set. **Do not
integrate a single number without also integrating the parameter record
that produced it.**

## M11 — `net_flow_stress_p50` (per ward, 5-year horizon)

**Definition.** Same as M10, with an additional annual loss from
scored at-risk units at rate `AT_RISK_ANNUAL_LOSS_RATE`.

**Formula.**
`net_flow_stress_p50 = percentile_50(net_flow_base − at_risk_units_w ×
AT_RISK_ANNUAL_LOSS_RATE × h)`

**Unit.** Units (net).

**Source.** M9, M10.

**Null rules.** As M10.

**KinesisIQ integration note.** This is the **stress-scenario** output,
intended for early-warning reporting. Do not publish without labelling it as
a scenario with named parameters.

# Project 6 — Developer Responsiveness & Supply Forecasting

## M12 — `approval_lag_days` (execution lag, City side)

**Definition.** Calendar days between the date a permit application was
submitted and the date the permit was issued.

**Formula.** `approval_lag_days = issued_date − application_date`

**Unit.** Integer days.

**Source.** City of Toronto Open Data — *Building Permits (Active)* and
*Building Permits (Cleared)*. Fields: `ISSUED_DATE`, `APPLICATION_DATE`.

**Population filter.** Housing permits only (`DWELLING_UNITS_CREATED > 0`
or `DWELLING_UNITS_LOST > 0`); `STATUS` must not contain
`under review|cancelled|withdrawn|refused|revoked`; the record must have a
confirmed issue date; `unit_anomaly = FALSE`.

**Null rules.** Null if either date missing. Null if the computed value
is negative (data-entry error). Never coerced to zero.

**Ward / neighbourhood assignment.** Spatial join where permit has usable
coordinates (via GEO_ID join to Address Points). Unmapped preserved.

**KinesisIQ integration note.** This is the "execution lag" component
contributed by the City. It reflects the planning and issuance process,
not the developer's construction speed.

## M13 — `build_lag_days` (execution lag, developer side)

**Definition.** Calendar days between the date a permit was issued and the
date the permit was recorded as completed.

**Formula.** `build_lag_days = completed_date − issued_date`

**Unit.** Integer days.

**Source.** Building Permits (Active) and (Cleared). Fields:
`COMPLETED_DATE`, `ISSUED_DATE`.

**Population filter.** Housing permits only (as M12); **mature permits
only** (`issued_date ≤ TODAY − 3 years`), so that recent permits not yet
completed do not silently bias the lag downwards.

**Null rules.** Null if either date is missing. Null if the computed
value is negative. Null for permits that have not yet completed.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** Censoring is fundamental: a permit issued
6 months ago has not had time to complete, so it cannot contribute a
build-lag observation. Reporting build lag without this filter produces a
misleadingly fast distribution. The maturity rule must be preserved in any
re-implementation.

## M14 — `total_execution_lag_days`

**Definition.** Total calendar days from the development application date
to the building permit completion date — the full end-to-end project
timeline.

**Formula.** `total_execution_lag_days = completed_date − application_date`

**Unit.** Integer days.

**Source.** Building Permits (Active) and (Cleared). Fields:
`COMPLETED_DATE`, `APPLICATION_DATE`.

**Population filter.** Housing permits that have completed (both dates
present and result non-negative) and are mature (issued at least 3 years
ago). Same `STATUS` and `unit_anomaly` filters as M12.

**Null rules.** Null if either date missing, or if the computed value is
negative.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** Where both intermediate lags are known,
`total_execution_lag_days ≈ approval_lag_days + build_lag_days` (equal
up to missing-day rounding). This measure corresponds most directly to the
brief's "execution lag" phrasing.


## M15 — `net_units_row`

**Definition.** Net dwelling units on a single permit record.

**Formula.** `net_units_row = units_created − units_lost`

**Unit.** Integer units.

**Source.** Building Permits. Fields: `DWELLING_UNITS_CREATED`,
`DWELLING_UNITS_LOST`.

**Population filter.** Any permit; aggregations applied downstream.

**Null rules.** Both source fields coerced to numeric, then blank means
zero (the source does not distinguish "no units" from "blank", and the
magnitude of the field is small enough that this is the correct
interpretation). Never null at the row level once coercion succeeds.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** Negative values indicate net loss of
dwelling units (demolition or conversion permits). Ward-level totals
therefore reflect both construction and removals.

## M16 — `net_units` (per ward, analysis window)

**Definition.** Sum of `net_units_row` across eligible permits issued in
the analysis window (2017-01-01 onward), per ward.

**Formula.** `net_units = Σ net_units_row` over eligible permits.

**Unit.** Integer units.

**Source.** M15.

**Population filter.** M15 records with `include_in_unit_analysis = TRUE`
(see M18) and `issued_date ≥ 2017-01-01`.

**Null rules.** Ward with no mapped eligible permits remains **null**
(not zero) because absence may reflect missing geography, not absence of
construction. This is a deliberate departure from ward-zero-filling —
permits have weaker geography coverage than demolition records.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** Any downstream aggregation that reports
per-ward net units must also report the wards where the measure is null
and the reason.

## M17 — `dev_app_count` (per ward, analysis window)

**Definition.** Number of development applications submitted in the
analysis window, per ward.

**Formula.** `dev_app_count = count(applications with verified ward and
event_date ≥ 2017-01-01)`

**Unit.** Integer count.

**Source.** City of Toronto Open Data — *Development Applications*.
Field: application submission date.

**Population filter.** Applications with a verified ward and a valid
date ≥ 2017-01-01.

**Null rules.** Ward with no mapped applications receives **zero only
when overall application coverage ≥ 80%**; otherwise null.

**Ward / neighbourhood assignment.** Spatial join where coordinates
exist; source ward field otherwise; pre-2018 ward fields unmapped.

**KinesisIQ integration note.** The application dataset is used as a
leading indicator of future supply, not as a measure of permitted units.
An application does not guarantee a permit.

---

## M18 — `units_parsed` (indicative only)

**Definition.** The largest integer before a word matching
`units?|suites?` in the free-text application description, used as a
**fallback** estimate of the number of dwelling units in the proposal.

**Formula.** `units_parsed = max(integer matches in description text
matching the pattern `\d[\d,]*\s*(residential|dwelling|rental|affordable|
condominium|apartment|new|replacement)?\s*(units?|suites?)`)`

**Unit.** Integer units (indicative).

**Source.** Development Applications. Field: description text.

**Population filter.** All application records with a description.

**Null rules.** Null when no pattern matches.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** This measure is **indicative only** and
must **never** be used as a training target or as a final unit-count
figure. It exists as a proxy for scale ranking. If KinesisIQ can obtain
structured unit counts (e.g. through an address-join to the pipeline
dataset), replace this measure entirely and mark it deprecated.

## M19 — `mature_for_build_lag`

**Definition.** Boolean flag indicating a permit was issued sufficiently
long ago that non-completion cannot be assumed a data gap — i.e. it has
had time to plausibly complete.

**Formula.** `mature_for_build_lag = (issued_date ≤ TODAY − 3 years)`

**Unit.** Boolean.

**Source.** Derived from permit `ISSUED_DATE`.

**Population filter.** Any housing permit.

**Null rules.** Null when `issued_date` is null.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** `BUILD_LAG_MATURITY_YEARS = 3` is an
assumption based on the observed distribution of build lags in the
dataset. If the median build lag changes materially, re-derive the
threshold. Any model using build lag as a training target must filter to
`mature_for_build_lag = TRUE`; otherwise the target is biased towards
fast completions.

## M20 — `unit_anomaly`, `include_in_unit_analysis`

**Definition.** Flag for extreme unit-count values that likely represent
data-entry errors, plus the composite eligibility flag for unit-based
analysis.

**Formula.**
- `unit_anomaly = (units_created > 5000) OR (units_lost > 5000)`
- `status_excluded = STATUS matches under review|cancelled|canceled|withdrawn|
  refused|revoked`
- `confirmed_issued_record = issued_date not null AND NOT status_excluded`
- `include_in_unit_analysis = confirmed_issued_record AND NOT unit_anomaly`

**Unit.** Boolean.

**Source.** Building Permits. Fields: `DWELLING_UNITS_CREATED`,
`DWELLING_UNITS_LOST`, `STATUS`, `ISSUED_DATE`.

**Population filter.** All permits.

**Null rules.** Flags are never null; missing inputs default to FALSE for
the flag.

**Ward / neighbourhood assignment.** Inherited.

**KinesisIQ integration note.** The 5,000-unit threshold is analyst-set
after inspecting the upper tail; the flagged records are retained in the
audit table and excluded from unit-based modelling but **not deleted**.
If KinesisIQ prefers a different threshold, the anomaly list should be
regenerated. Flagged records must never be silently dropped.

---

## M21 — Cluster features (ward typology)

**Definition.** Features used in unsupervised k-means clustering of wards.
These are not intended for supervised modelling.

**Features and preprocessing.**

| Feature | Source | Preprocessing |
|---|---|---|
| `net_affordable_change` | M6 | signed-log: `sign(x) × log1p(|x|)` |
| `pct_units_at_risk` | M9 | raw |
| `pipeline_cover` | M6 | raw, clipped at 10 |
| `approval_lag_median` | M12 aggregated | raw |
| `build_lag_median` | M13 aggregated | raw |
| `dev_applications` | M17 | signed-log |
| `units_permitted` | M16 | signed-log |

**Unit.** Mixed (see table).

**Source.** Per-feature as above.

**Population filter.** Wards where ≤ 50% of features are missing.
Remaining nulls filled with the feature median (and then 0 if the median
is also missing). Features standardised with z-scores before clustering.

**Null rules.** Wards excluded from clustering if > 50% of features are
missing; this exclusion is reported.

**Ward / neighbourhood assignment.** Ward-level only.

**KinesisIQ integration note.** Cluster number is an **exploratory
descriptor**, not a fixed label. If ward data updates, k-means cluster
assignments may shift and cluster numbers may re-index. Report the
cluster profile (mean of each feature per cluster) alongside the number
so the interpretation survives re-runs.

## Documented analyst-set constants

For clarity, the following values are analyst assumptions, not sourced facts:

| Constant | Value | Reason |
|---|---|---|
| `LOW_SCORE_THRESHOLD` | 50 | RentSafeTO at-risk threshold |
| `MIN_GEO_COVERAGE` | 0.80 | Coverage gate below which ward zero-filling is suppressed |
| `PANEL_START_YEAR` | 2017 | First year of demolition data |
| `BUILD_LAG_MATURITY_YEARS` | 3 | Censoring threshold for build lag |
| `MAX_UNITS_PER_PERMIT_REVIEW` | 5000 | Upper-tail anomaly flag |
| `PIPELINE_REALISATION` | 0.50 | Baseline pipeline delivery share (scenario) |
| `PIPELINE_DELIVERY_YEARS` | 5 | Scenario delivery ramp |
| `AT_RISK_ANNUAL_LOSS_RATE` | 0.01 | Stress-scenario annual loss rate |
| `POOL_K` | 3 | Partial-pooling shrinkage constant |
| `FORECAST_YEARS` | 5 | Forecast horizon |
| `SIMS` | 2000 | Monte Carlo draws |

Any KinesisIQ re-implementation must record which of these were changed
and re-derive the affected measures.
## End of document

