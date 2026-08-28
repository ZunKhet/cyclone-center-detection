# Methodology

## 1. Problem Formulation

Experiment 03 formulates cyclone center detection as a **multi-task learning problem**.

Given a regional ERA5 mean sea level pressure (MSLP) field, the model simultaneously predicts:

1. whether a tropical cyclone is present in the regional frame; and
2. the spatial location of the cyclone center.

For an input atmospheric field \(X\), the model can be represented conceptually as:

```text
X → Shared CNN Encoder → {Presence Prediction, Center Heatmap}
```

The two tasks share spatial features learned from the same MSLP field.

---

# 2. Input

The Experiment 03 model uses one atmospheric variable:

**ERA5 mean sea level pressure (MSLP)**

Each sample covers:

* Latitude: 0°–30°N
* Longitude: 75°–105°E
* Grid size: 121 × 121
* Approximate grid spacing: 0.25°

The model input therefore has the form:

```text
[1, 121, 121]
```

where the single channel represents normalized MSLP.

Experiment 03 intentionally uses MSLP alone to establish a simple and meteorologically interpretable baseline.

---

# 3. Input Normalization

MSLP is standardized before being passed to the neural network.

Normalization statistics are calculated using the **training split only**.

Conceptually:

```text
normalized MSLP = (MSLP - training mean) / training standard deviation
```

The resulting normalization statistics are stored as:

```text
models/
├── mslp_mean.npy
└── mslp_std.npy
```

The same values must be used during validation, testing, and later inference.

Calculating normalization statistics only from training data prevents information from the validation and test sets from leaking into preprocessing.

---

# 4. Multi-Task Architecture

The model contains three conceptual components:

1. shared spatial feature extraction;
2. cyclone-presence classification;
3. cyclone-center localization.

The overall structure is:

```text
                         MSLP
                    [1 × 121 × 121]
                           │
                           ▼
                  Shared CNN Encoder
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
          Presence Head        Localization Head
                │                     │
                ▼                     ▼
         Presence Logit       Center Heatmap
                │
                ▼
           P(cyclone)
```

The shared encoder allows both tasks to learn from common atmospheric pressure structures.

---

# 5. Shared Spatial Encoder

The shared convolutional encoder extracts increasingly abstract spatial features from the regional MSLP field.

Early convolutional features can respond to relatively local pressure patterns, while deeper representations can encode broader structures across the regional domain.

These shared features are then supplied to both prediction tasks.

This design is useful because cyclone presence and cyclone-center location are related problems:

* identifying whether a cyclone exists requires recognizing cyclone-like pressure organization;
* locating its center requires retaining information about where that organization occurs.

A completely separate model for each task would not exploit this shared structure.

---

# 6. Cyclone Presence Head

The presence head performs binary classification.

Its output represents:

```text
P(cyclone)
```

or the estimated probability that the regional ERA5 frame contains a cyclone.

Ground-truth labels are:

```text
1 → cyclone frame
0 → non-cyclone frame
```

Presence classification is trained using both positive and negative samples.

During evaluation, the probability is converted into a binary cyclone/non-cyclone prediction using the classification threshold defined in the experiment.

Performance is reported using:

* accuracy;
* precision;
* recall;
* F1 score;
* confusion counts.

---

# 7. Cyclone Center Localization

Experiment 03 does **not** directly regress cyclone latitude and longitude.

Instead, center localization is formulated as a spatial heatmap-prediction problem.

For each positive cyclone sample, the audited cyclone center is mapped onto the 121 × 121 spatial grid and represented by a target heatmap.

Conceptually:

```text
Audited cyclone center
        │
        ▼
Grid coordinate
        │
        ▼
Target heatmap
```

The localization head predicts a heatmap over the same spatial domain:

```text
Predicted heatmap
        │
        ▼
Maximum-response pixel
        │
        ▼
Predicted row / column
        │
        ▼
Predicted latitude / longitude
```

The maximum-response position is treated as the predicted cyclone center.

---

# 8. Why Use a Heatmap?

Heatmap localization preserves the spatial nature of the problem.

Instead of asking the network to produce two independent numerical coordinates, the network learns where cyclone-center evidence occurs within the atmospheric field.

This also allows the localization output to be inspected visually alongside the original MSLP field.

The approach is therefore useful both computationally and diagnostically.

---

# 9. Multi-Task Training

Experiment 03 jointly optimizes:

* cyclone-presence classification;
* cyclone-center localization.

For every training sample, the presence task receives supervision.

Localization supervision is handled differently.

### Positive cyclone sample

```text
Presence target     = 1
Localization target = cyclone-center heatmap
```

Both tasks contribute to training.

### Negative sample

```text
Presence target     = 0
Localization target = no cyclone center
```

The localization loss is not used to train a meaningful cyclone-center position for negative samples.

This is important because a non-cyclone atmospheric frame has no valid target cyclone center.

---

# 10. Loss Function

Training uses a combined multi-task objective containing:

```text
Presence classification loss
+
Center-localization loss
```

Presence loss is calculated using both positive and negative samples.

Localization loss is applied to positive cyclone samples.

Conceptually:

```text
Total loss =
classification component
+
localization component
```

This allows the shared network to learn both cyclone recognition and spatial localization simultaneously.

The exact weighting and implementation are retained in the Experiment 03 notebook:

```text
notebooks/center_detection_experiment_03.ipynb
```

---

# 11. Training Strategy

The model is trained using the predefined training split.

Validation samples are used for:

* monitoring generalization;
* learning-rate scheduling;
* checkpoint selection;
* early stopping.

The test set is not used to select the model.

Training therefore follows:

```text
Training set
    │
    ▼
Parameter optimization
    │
    ▼
Validation set
    │
    ├── performance monitoring
    ├── learning-rate adjustment
    ├── checkpoint selection
    └── early stopping
             │
             ▼
       Best checkpoint
             │
             ▼
        Final test set
```

---

# 12. Best-Checkpoint Selection

Experiment 03 does not select the final checkpoint solely from training loss.

A validation score combines presence-detection quality with center-localization quality.

This is necessary because the network solves two tasks.

A checkpoint that performs exceptionally well at classification but poorly at localization should not automatically be considered the best model, and vice versa.

The best Experiment 03 checkpoint occurred at:

**Epoch 11**

with a validation score of:

**0.8695**

Training continued after this checkpoint while early stopping monitored whether validation performance improved.

The final reported test evaluation reloads the best checkpoint rather than simply using the last training epoch.

---

# 13. Learning-Rate Scheduling and Early Stopping

Experiment 03 uses learning-rate reduction when validation performance stops improving.

Early stopping prevents unnecessary continued optimization after the model has stopped improving on validation data.

Training eventually stopped after:

**36 epochs**

because the required improvement had not occurred within the configured patience period.

The final model therefore corresponds to the best validation checkpoint rather than Epoch 36.

---

# 14. Presence Evaluation

Presence detection is evaluated using:

### Accuracy

Fraction of all test samples classified correctly.

### Precision

Among frames predicted as cyclone frames, the fraction that are actual cyclone frames.

### Recall

Among actual cyclone frames, the fraction successfully detected.

### F1 Score

Harmonic mean of precision and recall.

Experiment 03 also reports:

```text
True positives
True negatives
False positives
False negatives
```

This makes the error types directly interpretable.

---

# 15. Localization Evaluation

Localization error is measured as geographic distance between:

```text
Predicted cyclone center
```

and:

```text
Audited cyclone center
```

after converting the predicted heatmap position back to latitude and longitude.

Distance is reported in kilometers.

The evaluation includes:

* mean localization error;
* median localization error;
* fraction within 25 km;
* fraction within 50 km;
* fraction within 100 km.

---

# 16. Two Localization Evaluation Modes

Experiment 03 reports localization in two different ways.

## 16.1 All True Cyclone Frames

The first evaluation includes **every true cyclone frame**, even when the presence classifier incorrectly predicts that no cyclone exists.

This evaluates the raw localization head independently of whether the presence decision is correct.

For the 341 positive test frames:

```text
Mean error   = 43.19 km
Median error = 19.03 km
Within 25 km = 64.8%
Within 50 km = 90.3%
Within 100 km = 95.3%
```

---

## 16.2 Successfully Detected Cyclones

A second evaluation considers only:

```text
actual cyclone = 1
AND
predicted cyclone = 1
```

This measures localization performance when the complete detection system successfully recognizes that a cyclone exists.

There were 321 such test frames.

Results were:

```text
Mean error    = 25.22 km
Median error  = 18.57 km
Within 25 km  = 67.0%
Within 50 km  = 92.2%
Within 100 km = 97.5%
```

This distinction is important when interpreting the model.

---

# 17. End-to-End Interpretation

The complete Experiment 03 inference process can be viewed as:

```text
ERA5 MSLP
    │
    ▼
Normalize
    │
    ▼
Multi-task CNN
    │
    ├─────────────────────────────┐
    ▼                             ▼
P(cyclone)                 Center heatmap
    │                             │
    ▼                             ▼
Presence decision          Heatmap argmax
                                  │
                                  ▼
                         Grid row / column
                                  │
                                  ▼
                         Latitude / longitude
```

Operationally, the presence prediction determines whether the model believes a cyclone exists.

When a cyclone is detected, the localization output provides its estimated center.

---

# 18. Error Analysis

Experiment 03 does not stop at aggregate test metrics.

Individual predictions are stored in:

```text
results/test_predictions.csv
```

including information such as:

```text
sample identifier
timestamp
actual presence
predicted presence
presence probability
predicted grid location
predicted geographic location
true geographic location
localization error
```

This enables sample-level failure analysis.

---

# 19. False-Negative Analysis

The test set contains:

**20 false-negative frames**

belonging to:

**10 unique cyclones**

Analysis showed that missed cyclone frames tend to have weaker MSLP contrast than successfully detected cyclone frames.

A diagnostic pressure-deficit measure was defined as:

```text
Local pressure deficit =
environmental mean MSLP − local minimum MSLP
```

Observed statistics were:

| Group    | Frames | Mean deficit | Median deficit |
| -------- | -----: | -----------: | -------------: |
| Detected |    321 |    10.54 hPa |       8.65 hPa |
| Missed   |     20 |     4.61 hPa |       3.38 hPa |

This indicates that weak pressure signatures represent an important failure mode for the MSLP-only model.

---

# 20. False-Positive Analysis

Experiment 03 produced:

**20 false positives**

These predictions were examined against IBTrACS in both time and space.

The analysis showed that most false positives could not simply be explained as mislabeled observations near an existing best-track cyclone.

However, at least one high-confidence false positive showed an organized low-pressure structure before its later IBTrACS representation.

This highlights an important labeling issue:

```text
negative according to best-track timing
```

does not necessarily mean:

```text
no physically organized disturbance exists
```

Therefore, some apparent classification errors may require meteorological interpretation rather than purely numerical evaluation.

---

# 21. Motivation for Subsequent Experiments

Experiment 03 establishes a strong MSLP-only baseline but reveals a specific weakness:

> Cyclones with weak or structurally ambiguous pressure signatures are more likely to be missed.

This motivates testing whether additional dynamical information can complement pressure structure.

In particular:

```text
U850 → zonal wind at 850 hPa
V850 → meridional wind at 850 hPa
```

may provide information about low-level circulation that is not fully represented by MSLP alone.

This hypothesis motivates the subsequent MSLP + U850 + V850 experiment.

The objective is not simply to add more input variables, but to test whether physically relevant circulation information improves the failure cases identified by the MSLP-only model.

---

# 22. Research Interpretation

Experiment 03 should therefore be understood as both:

1. a successful cyclone-presence and center-localization model; and
2. a controlled baseline for investigating which atmospheric variables actually contribute useful additional information.

Its primary test results are:

```text
Presence F1                         94.13%
Detected cyclone frames            321 / 341
Mean center error (all positives)  43.19 km
Mean center error (detected only)  25.22 km
Detected centers within 50 km      92.2%
```

Detailed numerical results are provided in [`results.md`](results.md).
