# AI Technical Specification

## 1. What the AI currently does

The repository contains two distinct analytics paths:

1. **Cognitive analytics / anomaly detection** (`ml_core.py` + `cognitive_engine.py`)
2. **Next-dose risk heuristic** (`prediction_engine.py`)

These should not be described as one trained predictive model. The first path is an unsupervised/statistical anomaly pipeline plus hand-written rules. The second is a weighted heuristic based on counts and time buckets.

No trained neural-network artifact, model checkpoint, or serialized estimator was present in the supplied ZIP. No training dataset directory was present either.

## 2. Feature extraction

`FeatureExtractor` processes `opened` events, sorted by timestamp. It builds one five-dimensional row per accepted event:

| Feature | Meaning | Source |
|---|---|---|
| `delay_minutes` | Opening time minus the configured schedule time for that compartment, adjusted around midnight | Event timestamp + schedule |
| `hour_of_day` | Hour of opening (0–23) | Event timestamp |
| `day_of_week` | ISO weekday (1–7) | Event timestamp |
| `is_weekend` | 1 for Saturday/Sunday, otherwise 0 | Event timestamp |
| `minutes_since_prev` | Minutes since the previous accepted opening event | Consecutive timestamps |

The extractor returns the numeric matrix, feature names, event labels, event count, and delay statistics.

### Delay statistics

The extractor calculates:
- Mean delay
- Standard deviation of delay
- Minimum and maximum delay
- Median delay
- `adherence_pct`, defined in code as the percentage of opening events whose absolute delay is at most 15 minutes

This adherence statistic depends on the schedule mapping supplied to the extractor. The current mapping stores the first schedule encountered for each compartment, so multiple schedules for one compartment need a more explicit schedule-matching implementation.

## 3. Anomaly detection pipeline

### Stage A: standardization

For each feature:

`X_std = (X - mean(X)) / (std(X) + 1e-9)`

This prevents features with larger numeric scales from dominating solely due to their units.

### Stage B: custom Isolation Forest

The project implements its own lightweight Isolation Forest:
- 100 random isolation trees.
- Random feature selected at each split.
- Random split value between the selected feature's minimum and maximum.
- Tree depth limited approximately by `ceil(log2(sample_count))`.
- Each sample receives an anomaly score based on average path length.

This is implemented directly in `ml_core.py`; it is not imported from scikit-learn.

### Stage C: Mahalanobis distance

The second detector:
- Calculates the feature mean and covariance matrix.
- Adds a small diagonal value (`1e-6`) for numerical stability.
- Attempts to invert the covariance matrix.
- Calculates each row's Mahalanobis distance from the multivariate center.

### Stage D: rank consensus

Both score arrays are converted to percentile ranks in `[0, 1]`. The final score is:

`consensus = 0.6 * isolation_forest_rank + 0.4 * mahalanobis_rank`

### Stage E: adaptive threshold

The configured threshold percentile is 88. Events with a consensus score **strictly greater than** the 88th percentile are flagged.

The implementation requires at least five events. With fewer than five, it returns `method="none"` and no anomalies.

## 4. Cognitive behavior classification

`PatternClassifier` returns one of:
- `Regular`
- `Delayed`
- `Irregular`
- `Weekend-Skipper`
- `Insufficient Data`

It requires at least five events. Its rules use:
- On-time percentage (`abs(delay) <= 15`)
- Mean delay
- Delay standard deviation
- Difference between weekday and weekend on-time percentages

Current rule order:
1. Adherence below 50% → `Irregular`
2. Weekday/weekend adherence difference above 40 percentage points → `Weekend-Skipper`
3. Delay standard deviation above 30 minutes → `Irregular`
4. Mean delay above 30 minutes → `Delayed`
5. Otherwise → `Regular`

The returned `confidence` is a formula based on event count and delay standard deviation. It is not a statistically calibrated probability.

## 5. Composite risk score

`RiskScorer` combines four hand-coded components:

| Component | Rule contribution |
|---|---|
| Unacknowledged `missed_dose` alerts | 0, 15, 25, or 30 points based on count |
| Absolute mean delay | 0, 10, 20, or 25 points |
| Delay standard deviation | 0, 10, 20, or 25 points |
| ML anomalies | 5 points per anomaly, capped at 20 |

The total is categorized:
- 0–25: Low
- 26–50: Medium
- 51–75: High
- 76–100: Critical

These weights and cutoffs are engineering rules in the source code. They are not derived from clinical studies or validated against patient outcomes.

## 6. Explainability

`Explainer` combines:
- Pattern classifier reason
- Number of detected anomalies and detector names
- Composite risk score and level
- A non-medical reminder/schedule suggestion for the `Delayed` category

This is a generated text explanation of the rules that fired, not a model-generated clinical explanation.

## 7. Next-dose risk heuristic

`NextDoseRiskPredictor` uses the last 30 days of `opened` events and requires at least five events. It combines:

`risk = 0.50*(1 - base) + 0.20*(1 - day_factor) + 0.15*(1 - time_factor) + 0.15*(1 - trend_factor)`

The result is clamped to `[0, 1]` and displayed as a percentage with Low / Moderate / High / Critical bands.

Important implementation detail: the code comments describe `base` as an adherence rate, but `_base_adherence()` actually estimates event frequency per day and divides by two. It does not match openings to individual scheduled doses or determine whether a dose was swallowed.

Similarly:
- `day_factor` is based on how many events occurred on the target weekday relative to average counts over represented weekdays.
- `time_factor` compares event counts in a four-part time-of-day bucket.
- `trend_factor` compares event counts in the most recent seven days with the previous seven days; if no prior-week events exist, it defaults to 0.5.
- `confidence` is `min(1, number_of_events / 30)`, a data-volume indicator rather than measured predictive confidence.

The prediction code currently accepts `patient_id` but does not use it to filter the data; it records predictions with patient ID 1.

## 8. What is not implemented

The supplied source does not show:
- A supervised training pipeline with labeled real-world patient data.
- A train/validation/test split.
- Cross-validation.
- Hyperparameter search.
- A persisted model artifact (`.pt`, `.pth`, `.onnx`, `.tflite`, `.h5`, `.pkl`, `.joblib`).
- Training-loss or validation-loss history.
- Camera-based medication recognition.
- A model that identifies a pill or verifies ingestion.

Do not add claims for these capabilities unless they are implemented and tested in a later version.
