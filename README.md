# Cyclone Presence Detection and Center Localization from ERA5 MSLP

A deep-learning research project for detecting tropical cyclone presence and localizing cyclone centers from regional ERA5 mean sea level pressure (MSLP) fields over the Bay of Bengal and surrounding region.

The project investigates whether a convolutional neural network can learn both:

1. **Cyclone presence detection** — determine whether a regional atmospheric frame contains a tropical cyclone.
2. **Cyclone center localization** — estimate the cyclone center when a cyclone is present.

The current primary experiment uses **MSLP only**, providing a simple meteorologically interpretable baseline for cyclone detection and localization.

---

## Study Region

The model operates on a fixed regional domain:

* **Latitude:** 0°–30°N
* **Longitude:** 75°–105°E
* **Grid:** 121 × 121
* **Spatial resolution:** approximately 0.25°
* **Primary atmospheric variable:** ERA5 mean sea level pressure (MSLP)

This domain covers the Bay of Bengal and surrounding regions affected by tropical cyclones.

---

## Dataset

The Experiment 03 dataset contains:

| Class              |   Samples |
| ------------------ | --------: |
| Cyclone frames     |     1,900 |
| Non-cyclone frames |     1,900 |
| **Total**          | **3,800** |

Positive samples contain cyclone events with audited cyclone-center labels.

Negative samples are ERA5 regional frames selected outside known cyclone periods.

The dataset uses fixed **training, validation, and test splits** so that the same samples are used consistently across experiments.

The final test set contains:

* 341 cyclone frames
* 341 non-cyclone frames
* **682 total samples**

Large materialized arrays are not stored directly in this repository. Dataset access and structure are described in [`docs/dataset.md`](docs/dataset.md).

---
## Dataset Access

The materialized Experiment 03 dataset is hosted separately on Google Drive because the NumPy arrays are too large to include directly in this repository.

**Google Drive dataset:** [Cyclone Center Detection — Experiment 03 Dataset](https://drive.google.com/drive/folders/1iFBOL6tDfCgMUUQl-TfOUjtCeKvNpcXn?usp=sharing)

The package contains:

* 1,900 positive cyclone MSLP frames
* 1,900 negative non-cyclone frames
* positive cyclone-center target heatmaps
* latitude and longitude grids
* sample manifests
* fixed train/validation/test split information

After downloading, place the data under:

```text
data/
├── center_detection_v1/
└── center_detection_v2/
```

Detailed file descriptions and verified array shapes are provided in [`docs/dataset.md`](docs/dataset.md).

---

## Model

Experiment 03 uses a **multi-task convolutional neural network** with a shared spatial encoder and two prediction tasks.

```text
                 Regional MSLP
                  1 × 121 × 121
                       │
                       ▼
                Shared CNN Encoder
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
        Presence Head      Localization Head
              │                 │
              ▼                 ▼
       P(cyclone)        Center Heatmap
```

### Presence Detection

The presence head predicts the probability that a cyclone exists within the regional frame.

```text
Output → P(cyclone)
```

### Center Localization

The localization head produces a spatial heatmap representing the predicted cyclone-center location.

The highest-response location in the predicted heatmap is converted from grid coordinates to latitude and longitude and compared with the audited cyclone center.

Localization loss is applied only to positive cyclone samples.

More details are available in [`docs/methodology.md`](docs/methodology.md).

---

## Experiment 03

Experiment 03 introduces negative ERA5 frames so that the model must distinguish cyclone conditions from non-cyclone atmospheric patterns while simultaneously learning cyclone-center localization.

**Input**

```text
ERA5 MSLP
1 × 121 × 121
```

**Outputs**

```text
Cyclone presence probability
+
Cyclone-center heatmap
```

The best model checkpoint was selected using validation performance with learning-rate reduction and early stopping.

The best validation checkpoint occurred at **Epoch 11**.

---

## Test Results

### Cyclone Presence Detection

| Metric    | Test Result |
| --------- | ----------: |
| Accuracy  |  **94.13%** |
| Precision |  **94.13%** |
| Recall    |  **94.13%** |
| F1 Score  |  **94.13%** |

Confusion counts:

|                 | Count |
| --------------- | ----: |
| True positives  |   321 |
| True negatives  |   321 |
| False positives |    20 |
| False negatives |    20 |

The model successfully detected **321 of 341 cyclone frames**.

---

## Cyclone Center Localization

### All True Cyclone Frames

Localization is first evaluated across all 341 true cyclone frames, including cyclone frames missed by the presence classifier.

| Metric        |  Test Result |
| ------------- | -----------: |
| Mean error    | **43.19 km** |
| Median error  | **19.03 km** |
| Within 25 km  |    **64.8%** |
| Within 50 km  |    **90.3%** |
| Within 100 km |    **95.3%** |

### Successfully Detected Cyclones

For the 321 cyclone frames correctly identified by the presence classifier:

| Metric        |  Test Result |
| ------------- | -----------: |
| Mean error    | **25.22 km** |
| Median error  | **18.57 km** |
| Within 25 km  |    **67.0%** |
| Within 50 km  |    **92.2%** |
| Within 100 km |    **97.5%** |

These results indicate that center localization is generally accurate once the cyclone has been successfully detected.

---

## Error Analysis

Detailed analysis of Experiment 03 revealed that missed cyclone frames tend to have substantially weaker local MSLP contrast.

A local pressure-deficit diagnostic was defined as:

```text
Local pressure deficit =
environmental mean MSLP − local minimum MSLP
```

The observed test-set statistics were:

| Group             | Frames | Mean Deficit | Median Deficit |
| ----------------- | -----: | -----------: | -------------: |
| Detected cyclones |    321 |    10.54 hPa |       8.65 hPa |
| Missed cyclones   |     20 |     4.61 hPa |       3.38 hPa |

This suggests that **weak pressure signatures are an important failure mode for an MSLP-only detector**.

However, the distributions overlap, indicating that pressure contrast alone does not completely explain detection success.

False-positive analysis also found that some high-confidence detections may correspond to organized low-pressure disturbances occurring before their later representation in IBTrACS.

These observations motivate further investigation using additional dynamical atmospheric variables.

---

## Research Direction

Experiment 03 establishes an MSLP-only baseline.

A natural follow-up hypothesis is:

> Can low-level wind information provide complementary circulation features for cyclones with weak or structurally complex pressure signatures?

Subsequent experiments investigate additional ERA5 variables such as:

* U-component of wind at 850 hPa
* V-component of wind at 850 hPa

The goal is to determine whether dynamical wind structure can improve upon the MSLP-only baseline rather than simply increasing model complexity.

---

## Repository Structure

```text
cyclone-center-detection/
│
├── README.md
├── .gitignore
│
├── docs/
│   ├── dataset.md
│   ├── methodology.md
│   └── results.md
│
├── models/
│   ├── center_detection_experiment_03_final.pt
│   ├── mslp_mean.npy
│   └── mslp_std.npy
│
├── notebooks/
│   ├── center_detection_experiment_03.ipynb
│   └── center_detection_error_analysis.ipynb
│
└── results/
    └── test_predictions.csv
```

---

## Reproducing Experiment 03

The materialized dataset should be placed locally under:

```text
data/
├── center_detection_v1/
└── center_detection_v2/
```

The `data/` directory is intentionally excluded from Git because the materialized arrays are significantly larger than the source code and documentation.

Detailed dataset structure and preparation instructions are provided in [`docs/dataset.md`](docs/dataset.md).

The main experiment can then be inspected and reproduced using:

```text
notebooks/center_detection_experiment_03.ipynb
```

---

## Data Sources

This project is based primarily on:

* **ERA5 reanalysis** atmospheric data
* **IBTrACS** tropical cyclone best-track information

IBTrACS information is used for cyclone-track and center-reference information, while ERA5 provides the gridded atmospheric fields used as model inputs.

Users should refer to the original ERA5 and IBTrACS providers for their respective data licenses, citation requirements, and terms of use.

---

## Limitations

The current model:

* operates on a fixed 0–30°N, 75–105°E regional domain;
* uses MSLP as its only model input in Experiment 03;
* depends on the quality of cyclone-center labels and negative-frame selection;
* can miss weak-pressure or structurally ambiguous cyclone systems;
* may respond to organized low-pressure disturbances that are not represented as cyclone observations in IBTrACS at the corresponding timestamp.

The model should therefore be considered a **research prototype**, not an operational tropical cyclone warning or forecasting system.

---

## Documentation

Additional documentation:

* [`Dataset`](docs/dataset.md) — dataset construction, files, labels, splits, and materialization
* [`Methodology`](docs/methodology.md) — architecture, inputs, outputs, training, losses, and evaluation
* [`Results`](docs/results.md) — validation/test performance and detailed error analysis

---

## Status

**Experiment 03 — MSLP-only multi-task cyclone detection and center localization**

Current test performance:

**Presence F1: 94.13% · Detected-cyclone mean center error: 25.22 km · 92.2% within 50 km**

Further experiments investigate whether additional atmospheric variables improve performance on weak or structurally complex cyclone systems.
