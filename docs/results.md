# Results

## 1. Experiment 03 Summary

Experiment 03 evaluates an MSLP-only multi-task CNN for:

1. **cyclone presence detection**, and
2. **cyclone center localization**.

The model operates on 121 × 121 ERA5 mean sea level pressure fields covering:

* 0°–30°N
* 75°–105°E

The dataset contains:

* 1,900 positive cyclone frames
* 1,900 negative non-cyclone frames

The final test set contains:

* 341 cyclone frames
* 341 non-cyclone frames
* 682 total samples

---

# 2. Best Validation Checkpoint

The best validation checkpoint occurred at:

**Epoch 11**

with validation score:

**0.8695**

### Presence Detection

| Metric    | Validation |
| --------- | ---------: |
| Accuracy  |     96.41% |
| Precision |     96.83% |
| Recall    |     95.96% |
| F1        |     96.40% |

### Center Localization

| Metric        | Validation |
| ------------- | ---------: |
| Mean error    |   37.41 km |
| Median error  |   15.93 km |
| Within 25 km  |      75.8% |
| Within 50 km  |      91.9% |
| Within 100 km |      96.0% |

Training continued after Epoch 11, but no later checkpoint achieved a better validation score.

Early stopping terminated training after 36 epochs.

The final test evaluation reloads the best validation checkpoint rather than using the last training epoch.

---

# 3. Final Test Results

## 3.1 Cyclone Presence Detection

The final test performance was:

| Metric    |       Test |
| --------- | ---------: |
| Accuracy  | **94.13%** |
| Precision | **94.13%** |
| Recall    | **94.13%** |
| F1        | **94.13%** |

The confusion counts were:

| Outcome        |   Count |
| -------------- | ------: |
| True positive  | **321** |
| True negative  | **321** |
| False positive |  **20** |
| False negative |  **20** |

The model therefore correctly classified:

```text
642? No.
321 TP + 321 TN = 642 correct samples
```

out of:

```text
682 total test samples
```

giving an accuracy of approximately 94.13%.

Among the 341 true cyclone frames, the model successfully detected:

**321 / 341**

or approximately:

**94.1%**

---

# 4. Validation-to-Test Generalization

Presence F1 decreased from:

```text
Validation F1 = 96.40%
```

to:

```text
Test F1 = 94.13%
```

This corresponds to a reduction of approximately:

**2.27 percentage points**

The decrease is noticeable but relatively small.

The test result therefore remains close to validation performance and does not show a severe generalization collapse.

---

# 5. Center Localization on All True Cyclone Frames

Localization was first evaluated across all:

**341 true cyclone frames**

This evaluation includes cyclone frames that were missed by the presence classifier.

The resulting metrics were:

| Metric        |         Test |
| ------------- | -----------: |
| Mean error    | **43.19 km** |
| Median error  | **19.03 km** |
| Within 25 km  |    **64.8%** |
| Within 50 km  |    **90.3%** |
| Within 100 km |    **95.3%** |

The large difference between:

```text
Mean error   = 43.19 km
Median error = 19.03 km
```

indicates that the localization-error distribution contains a relatively small number of large errors.

Most predictions are considerably closer to the true cyclone center than the mean alone suggests.

---

# 6. Validation-to-Test Localization

The best validation checkpoint achieved:

| Metric        | Validation |     Test |
| ------------- | ---------: | -------: |
| Mean error    |   37.41 km | 43.19 km |
| Median error  |   15.93 km | 19.03 km |
| Within 25 km  |      75.8% |    64.8% |
| Within 50 km  |      91.9% |    90.3% |
| Within 100 km |      96.0% |    95.3% |

The strongest degradation occurs at the strictest 25-km threshold.

However, the 50-km and 100-km localization rates remain close to validation.

This suggests that test localization remains generally stable, while very high-precision center localization is somewhat more difficult on unseen samples.

---

# 7. End-to-End Localization Among Detected Cyclones

A second localization evaluation was performed only on cyclone frames that were correctly detected by the presence head:

```text
actual presence = 1
AND
predicted presence = 1
```

There were:

**321 detected cyclone frames**

The corresponding localization metrics were:

| Metric        | Detected Cyclones |
| ------------- | ----------------: |
| Mean error    |      **25.22 km** |
| Median error  |      **18.57 km** |
| Within 25 km  |         **67.0%** |
| Within 50 km  |         **92.2%** |
| Within 100 km |         **97.5%** |

This is substantially better than the mean error calculated across all true cyclone frames.

The difference is:

```text
All true cyclone frames: 43.19 km
Detected cyclone frames: 25.22 km
```

This indicates that a meaningful portion of the largest localization errors occurs on cyclone frames that are also difficult for the presence classifier.

---

# 8. Interpretation of Localization Performance

The two localization evaluations answer different questions.

### All true cyclone frames

This asks:

> How well does the localization head identify cyclone centers across every true cyclone frame, including difficult cases missed by the detector?

Result:

**43.19 km mean error**

### Successfully detected cyclone frames

This asks:

> When the model successfully recognizes that a cyclone exists, how accurately does it place the center?

Result:

**25.22 km mean error**

The second metric is more representative of the full end-to-end detector when a positive detection has occurred.

The first metric is more useful for understanding the raw behavior and failure modes of the localization head.

Both should therefore be reported.

---

# 9. False-Negative Analysis

The test set contains:

**20 false-negative frames**

These frames belong to:

**10 unique cyclones**

Most affected cyclones were only partially missed.

However, one cyclone was completely missed across all of its evaluated test frames:

```text
SID 2023160N20092
```

Its predicted cyclone probabilities were approximately:

```text
0.013
0.050
0.049
0.037
```

These values are far below the decision threshold.

This indicates a systematic model failure rather than a borderline classification decision.

---

# 10. MSLP Pressure-Deficit Diagnostic

To investigate the false negatives, a local pressure-deficit measure was calculated:

```text
Local pressure deficit =
environmental mean MSLP − local minimum MSLP
```

The regions were defined as:

```text
Local region:
within 250 km of the audited cyclone center

Environmental region:
500–1000 km from the audited cyclone center
```

Across all 341 positive test frames:

| Statistic       | Pressure deficit |
| --------------- | ---------------: |
| Mean            |        10.19 hPa |
| Median          |         8.40 hPa |
| 25th percentile |         5.65 hPa |
| 75th percentile |        12.81 hPa |

A clear difference was observed between correctly detected and missed cyclone frames.

| Group    | Frames |  Mean deficit | Median deficit |
| -------- | -----: | ------------: | -------------: |
| Detected |    321 | **10.54 hPa** |   **8.65 hPa** |
| Missed   |     20 |  **4.61 hPa** |   **3.38 hPa** |

The missed frames therefore tend to have substantially weaker local pressure contrast.

---

# 11. Completely Missed Cyclone

For:

```text
SID 2023160N20092
```

the four analyzed frames had pressure deficits of:

```text
4.10 hPa
4.41 hPa
4.83 hPa
5.92 hPa
```

with mean:

**4.81 hPa**

For comparison:

| Statistic       | Detected frames |
| --------------- | --------------: |
| Median          |        8.65 hPa |
| 25th percentile |        5.89 hPa |
| 10th percentile |        4.41 hPa |

The completely missed cyclone therefore lies toward the weak-pressure-signature portion of the test distribution.

Visual inspection also showed a broad competing low-pressure structure elsewhere in the regional MSLP field.

This likely makes the cyclone harder to distinguish using pressure structure alone.

---

# 12. Main False-Negative Finding

The error analysis supports the following conclusion:

> Weak local MSLP contrast is strongly associated with missed cyclone detection.

However, the distributions of detected and missed samples overlap.

Therefore:

```text
weak pressure contrast
```

is an important explanatory factor, but it is not sufficient by itself to determine whether a cyclone will be detected.

Other aspects of atmospheric structure likely also contribute.

---

# 13. False-Positive Analysis

Experiment 03 produced:

**20 false positives**

Their predicted cyclone probabilities were:

| Statistic | Probability |
| --------- | ----------: |
| Mean      |       0.774 |
| Median    |       0.800 |
| Maximum   |       0.981 |

These are not primarily low-confidence threshold errors.

Many false positives were predicted with relatively high confidence.

---

# 14. Temporal Comparison with IBTrACS

False-positive timestamps were compared against nearby IBTrACS observations.

The fractions occurring near an IBTrACS observation were:

| Time window  | Fraction |
| ------------ | -------: |
| Within 48 h  |      30% |
| Within 72 h  |      40% |
| Within 120 h |      75% |

This initially suggests that some false positives could be temporally close to tropical-cyclone activity.

However, time alone is not enough.

---

# 15. Spatial Comparison with IBTrACS

Spatial comparison showed:

| Distance      | Fraction |
| ------------- | -------: |
| Within 100 km |       0% |
| Within 200 km |      15% |
| Within 300 km |      25% |
| Within 500 km |      50% |

Only:

**3 / 20 false positives**

or:

**15%**

satisfied both:

```text
within ±120 h
AND
within 300 km
```

Therefore, most false positives cannot simply be explained as slightly misaligned known cyclone observations.

---

# 16. Pre-IBTrACS Disturbance Example

One particularly high-confidence false positive:

```text
negative_000113
```

had:

```text
P(cyclone) ≈ 0.981
```

Visual and temporal analysis showed an organized low-pressure system that later corresponded to an IBTrACS tropical-cyclone entry.

This suggests that at least some samples labeled as negative may contain physically meaningful developing disturbances before formal best-track representation.

This is an important distinction:

```text
Negative according to the dataset label
```

does not always imply:

```text
Meteorologically featureless atmosphere
```

The model may therefore occasionally respond to genuine cyclone-like precursor structure.

---

# 17. Implications for Dataset Labels

The false-positive analysis highlights a limitation of binary labels derived from cyclone-observation periods.

Atmospheric development is continuous, while best-track labels represent discrete observational records.

A developing disturbance may already exhibit organized pressure structure before its official best-track representation.

This means that some apparent false positives may not be straightforward model mistakes.

Future dataset versions could potentially distinguish between:

```text
non-cyclone background
developing disturbance
established tropical cyclone
```

rather than using only a binary cyclone/non-cyclone label.

---

# 18. U850/V850 Diagnostic Motivation

U850 and V850 were not used as Experiment 03 model inputs.

However, low-level wind fields were inspected after identifying the completely missed cyclone.

The diagnostic showed additional low-level flow structure around the system.

The circulation was not necessarily a simple compact vortex, but the wind field contained dynamical information that was not directly available from MSLP alone.

This motivated the hypothesis:

> Low-level wind information may help identify weak or structurally complex systems that are difficult to recognize using pressure structure alone.

---

# 19. Experiment 03 Main Findings

Experiment 03 demonstrates that an MSLP-only multi-task CNN can achieve strong performance on the test set.

Headline results are:

```text
Presence F1                         94.13%
Cyclone detection rate             94.1%
Mean error among detected cyclones 25.22 km
Median error among detected        18.57 km
Detected centers within 50 km      92.2%
Detected centers within 100 km     97.5%
```

The model therefore provides a strong MSLP-only baseline.

---

# 20. Main Limitation

The most important identified limitation is:

> weak or ambiguous pressure structure.

Missed cyclone frames have substantially lower average pressure contrast than correctly detected frames.

This limitation is physically meaningful because an MSLP-only model depends entirely on pressure morphology to distinguish cyclone structure from the surrounding atmospheric field.

---

# 21. Research Hypothesis Generated by Experiment 03

Experiment 03 leads naturally to the following next hypothesis:

> Combining MSLP with low-level zonal and meridional wind may provide complementary circulation information that improves detection of weak or structurally complex cyclones.

This motivates testing:

```text
MSLP
+
U850
+
V850
```

while keeping the dataset splits, targets, model-training protocol, and evaluation procedure as consistent as possible.

This makes the subsequent experiment a controlled test of whether additional physically meaningful atmospheric information improves upon the MSLP-only baseline.

---

# 22. Result Files

Per-sample test predictions are stored in:

```text
results/test_predictions.csv
```

The final model is stored in:

```text
models/center_detection_experiment_03_final.pt
```

The training-derived MSLP normalization statistics are stored in:

```text
models/mslp_mean.npy
models/mslp_std.npy
```

The main experiment notebook is:

```text
notebooks/center_detection_experiment_03.ipynb
```

and supplementary error-analysis work is stored in:

```text
notebooks/center_detection_error_analysis.ipynb
```
