# Walk-Run Classification

Classify walking vs. running using wrist-worn accelerometer and gyroscope data. The project emphasizes reliable evaluation through **day-based forward-chaining validation**, a sealed final test day, and window-level features.

**Headline result:** 451/451 windows correctly classified on the sealed test day, with **0.993 balanced mean recall** in cross-validation. Zero test errors correspond to an approximate 0.7% upper error-rate bound at 95% confidence under the rule of three, assuming independent representative windows. This is not proof of zero real-world errors.

**Scope:** One participant, one device, one recording period.

## Dataset

| Property | Details |
|---|---|
| Rows | 88,588 |
| Sensors | 3-axis accelerometer and 3-axis gyroscope |
| Participant | One user (`viktor`) |
| Period | June 30–July 17, 2017 |
| Recording days | 12 |
| Task | Walking (0) vs. running (1) |

Investigation found 12 date-contiguous blocks, with rows scrambled within blocks and 9,334 backward timestamp jumps in the original file order. All time-based operations therefore use parsed, sorted timestamps.

## Key Challenges

1. **Temporal leakage:** 99.98% of adjacent time-sorted rows share the same label. Random row splitting could overestimate generalization, so validation uses future recording days rather than randomly shuffled rows.
2. **Wrist and time confounding:** The first eight recording days use wrist label 0 and the final four use wrist label 1. Wrist was excluded from model features because it could act as a proxy for date.
3. **Undocumented labels:** Sensor patterns suggested the mapping `0 = walking` and `1 = running`. Walking showed a strong gravity component, while running showed greater variation and acceleration peaks. This mapping was inferred from the data rather than independently verified.
4. **Recording gaps and activity boundaries:** Windows are built only within continuous recording runs, with a label-purity assertion to prevent windows from mixing activities.
5. **Uneven class distribution across days:** Some validation days contain only one activity. Class recall is averaged only over folds where that class is present.

## Methodology

- Parse and sort timestamps before time-based operations.
- Use forward-chaining validation across recording days.
- Keep July 17 as a sealed test day, evaluated once after model selection.
- Split recordings at gaps greater than one second.
- Build non-overlapping windows of 25 rows, approximately five seconds at the typical sampling rate.
- Extract 24 features per window: mean, standard deviation, minimum, and maximum for each of the six sensor channels.
- Compare a dummy baseline, Logistic Regression, Random Forest, and an MLP.
- Select models using balanced mean recall and a predefined promotion threshold.

Timestamps are used for ordering and splitting, not as model features.

## Model Results

| Model | Prediction unit | Balanced mean recall | Walk recall | Run recall |
|---|---|---:|---:|---:|
| Dummy | Row | 0.479 | 0.125 | 0.833 |
| Logistic Regression | Row | 0.820 | 0.838 | 0.803 |
| Random Forest | Row | 0.924 | 0.865 | 0.983 |
| MLP | Row | 0.924 | 0.872 | 0.976 |
| Logistic Regression | Window | 0.993 | 1.000 | 0.986 |
| Random Forest | Window | 0.999 | 0.999 | 0.999 |

## Approach in Five Steps

1. **Dummy baseline:** 0.479 balanced mean recall.
2. **Logistic Regression on individual readings:** 0.820.
3. **Random Forest and MLP on individual readings:** Both reached 0.924 but failed the predefined requirement of at least 0.95 recall for each class.
4. **Window-based features:** Summarizing 25 readings into 24 statistical features raised Logistic Regression to 0.993 balanced mean recall, passing the quality threshold. This was the decisive improvement.
5. **Window-based Random Forest:** Reached 0.999, an improvement of only 0.006. Because this was below the predefined 0.010 promotion margin, the simpler Logistic Regression model remained the champion.

**Key lesson:** Feature representation mattered more than model complexity. The same Logistic Regression approach improved from 0.820 to 0.993 when the features described five-second activity windows instead of individual sensor readings.

## Final Test Results

The selected Logistic Regression model correctly classified **451 of 451 windows** on the sealed July 17 test day.

| Metric | Result |
|---|---:|
| Test windows | 451 |
| Correct predictions | 451 |
| Window-level accuracy | 1.000 |
| Balanced recall | 1.000 |
| Walking windows | 180 |
| Running windows | 271 |

The corresponding row-level score covers 11,275 rows. The remaining 90 rows were excluded because they did not form a complete 25-row window. This row score is derived from the window predictions and is not independent evidence.

## Repository Structure

```text
walk-run-classification/
├── data/
│   └── original.csv                 # Original dataset, untouched
├── notebooks/
│   └── ...                          # EDA, modeling, sealed test, reports
├── models/
│   └── walk_run_champion.joblib     # Saved model and metadata bundle
├── requirements.txt
└── README.md
```

## Reproduce

Run these commands from the repository root using Python:

```bash
python -m venv venv
```

On Windows PowerShell:

```powershell
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
jupyter notebook
```

Open the project notebook inside `notebooks/` and run the cells from top to bottom.

The project uses `random_state=42` where applicable. Reproducibility also depends on package versions, data integrity, and the notebook's execution order. The sealed test should be run only after model selection is complete.

## Limitations

- One participant, one device, and one held-out test day.
- Activity labels were inferred from sensor physics rather than documented ground truth.
- Strong test performance does not establish generalization to new users or recording conditions.
- The approximate 0.7% upper error-rate bound assumes independent, representative windows; correlations between windows may make it overly optimistic.

This is not a cross-person or general-purpose activity-recognition system. See the **Model Comparison Report** in the notebook for the detailed evaluation and limitations.