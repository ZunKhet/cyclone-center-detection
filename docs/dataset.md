# Dataset

## Overview

Experiment 03 uses a materialized dataset designed for two related tasks:

1. **Cyclone presence detection**
2. **Cyclone center localization**

Each sample represents a regional atmospheric frame covering:

* **Latitude:** 0°–30°N
* **Longitude:** 75°–105°E
* **Grid size:** 121 × 121
* **Approximate spatial resolution:** 0.25°

The Experiment 03 model uses **ERA5 mean sea level pressure (MSLP)** as its only atmospheric input.

The complete dataset contains:

| Sample type                 |     Count |
| --------------------------- | --------: |
| Positive cyclone frames     |     1,900 |
| Negative non-cyclone frames |     1,900 |
| **Total**                   | **3,800** |

The materialized dataset is stored separately from the GitHub repository because of its size.

---

# 1. Source Data

## 1.1 ERA5

ERA5 reanalysis provides the gridded atmospheric fields used as model inputs.

Experiment 03 uses:

* **Mean sea level pressure (MSLP)**

Each materialized atmospheric frame covers the complete 121 × 121 regional grid.

The model receives one atmospheric channel:

```text
MSLP → [1, 121, 121]
```

Additional ERA5 variables were examined in diagnostic and follow-up experiments but are not model inputs in Experiment 03.

---

## 1.2 IBTrACS

The International Best Track Archive for Climate Stewardship (IBTrACS) provides tropical-cyclone reference information used during dataset construction and diagnostic analysis.

Relevant cyclone information includes:

```text
Cyclone identifier
Timestamp
Latitude
Longitude
```

Positive cyclone-center labels were audited before materialization.

---

# 2. Dataset Composition

The dataset is balanced between positive and negative samples:

```text
Positive cyclone frames : 1,900
Negative frames         : 1,900
Total                   : 3,800
```

The two classes use different physical storage layouts.

Positive samples are stacked together into large NumPy arrays, while negative samples are stored as individual NumPy files.

---

# 3. Positive Cyclone Samples

The positive dataset contains:

**1,900 cyclone frames**

Each positive sample consists of:

```text
ERA5 MSLP field
+
Cyclone-presence label = 1
+
Cyclone-center target heatmap
```

The MSLP input covers the complete regional domain rather than a crop centered on the cyclone.

Therefore, the model must learn:

1. whether cyclone-like pressure structure exists within the regional frame; and
2. where the cyclone center is located.

Positive materialized data originate from:

```text
center_detection_v1/
```

The Experiment 03 files are:

```text
center_detection_v1/
├── latitude.npy
├── longitude.npy
├── manifest.csv
├── mslp.npy
└── target_heatmap.npy
```

---

# 4. Positive MSLP Array

## `mslp.npy`

This file contains **all 1,900 positive MSLP samples in one NumPy array**.

Verified shape:

```text
(1900, 1, 121, 121)
```

Verified data type:

```text
float32
```

The dimensions represent:

```text
[sample, channel, latitude-grid, longitude-grid]
```

Therefore:

```text
mslp.npy.shape
→ (1900, 1, 121, 121)

mslp.npy[0].shape
→ (1, 121, 121)
```

The single channel represents MSLP.

Conceptually:

```text
mslp.npy
│
├── sample 0000 → [1, 121, 121]
├── sample 0001 → [1, 121, 121]
├── sample 0002 → [1, 121, 121]
│
├── ...
│
└── sample 1899 → [1, 121, 121]
```

Thus, although `mslp.npy` is a single file, it contains all **1,900 positive cyclone frames**.

---

# 5. Positive Center Targets

## `target_heatmap.npy`

This file contains all 1,900 positive cyclone-center localization targets.

Verified shape:

```text
(1900, 1, 121, 121)
```

Verified data type:

```text
float32
```

Therefore, one localization target has shape:

```text
(1, 121, 121)
```

Conceptually:

```text
target_heatmap.npy
│
├── target 0000 → [1, 121, 121]
├── target 0001 → [1, 121, 121]
├── target 0002 → [1, 121, 121]
│
├── ...
│
└── target 1899 → [1, 121, 121]
```

Experiment 03 does not directly regress cyclone latitude and longitude.

Instead, the audited cyclone center is represented spatially as a target heatmap.

Conceptually:

```text
Audited cyclone center
        │
        ▼
Grid row / column
        │
        ▼
Target heatmap
```

During inference:

```text
Predicted heatmap
        │
        ▼
Maximum-response location
        │
        ▼
Predicted grid row / column
        │
        ▼
Latitude / longitude
```

The resulting geographic coordinate is compared with the audited cyclone center to calculate localization error.

---

# 6. Coordinate Arrays

## `latitude.npy`

Contains the latitude coordinates associated with the regional grid.

Verified shape:

```text
(121,)
```

Verified data type:

```text
float32
```

The 121 values correspond to the latitude dimension of the 121 × 121 atmospheric fields.

---

## `longitude.npy`

Contains the longitude coordinates associated with the regional grid.

Verified shape:

```text
(121,)
```

Verified data type:

```text
float32
```

The 121 values correspond to the longitude dimension of the 121 × 121 atmospheric fields.

Together, `latitude.npy` and `longitude.npy` allow predicted grid locations to be converted back into geographic coordinates.

---

# 7. Positive Manifest

## `manifest.csv`

Contains sample-level metadata associated with the positive cyclone frames.

It provides the metadata required to associate materialized positive samples with their corresponding cyclone observations.

---

# 8. Negative Samples

Experiment 03 introduces negative atmospheric frames so that the model must distinguish cyclone and non-cyclone conditions.

The negative dataset contains:

**1,900 non-cyclone frames**

Each negative sample contains:

```text
ERA5 MSLP field
+
Cyclone-presence label = 0
+
No valid cyclone center
```

The positive and negative classes are balanced:

```text
Positive : Negative
1900     : 1900
```

or:

```text
1 : 1
```

Unlike the positive samples, negative samples are stored individually.

---

# 9. Negative Materialized Data

Negative data are stored under:

```text
center_detection_v2/
```

with the structure:

```text
center_detection_v2/
├── combined_manifest.csv
├── negative_manifest.csv
├── negative_materialized_manifest.csv
│
├── negative_inputs/
│   ├── negative_000000.npy
│   ├── negative_000001.npy
│   ├── ...
│   └── negative_001899.npy
│
└── negative_targets/
    ├── negative_000000.npy
    ├── negative_000001.npy
    ├── ...
    └── negative_001899.npy
```

There are exactly:

```text
1,900 negative input files
1,900 negative target files
```

---

# 10. Negative Input Arrays

Each file in:

```text
negative_inputs/
```

contains one non-cyclone MSLP frame.

Verified per-file shape:

```text
(1, 121, 121)
```

Verified data type:

```text
float32
```

For example:

```text
negative_inputs/
├── negative_000000.npy → [1, 121, 121]
├── negative_000001.npy → [1, 121, 121]
├── ...
└── negative_001899.npy → [1, 121, 121]
```

There are 1,900 individual negative input arrays.

---

# 11. Negative Target Arrays

Each file in:

```text
negative_targets/
```

contains the corresponding localization target for a negative sample.

Verified per-file shape:

```text
(1, 121, 121)
```

Verified data type:

```text
float32
```

There are 1,900 individual negative target arrays.

Because a negative frame contains no labeled cyclone center, it does not provide a meaningful positive localization target.

---

# 12. Positive vs. Negative Storage Layout

The positive and negative datasets contain the same number of samples but use different storage formats.

## Positive

```text
mslp.npy
└── shape = (1900, 1, 121, 121)
    └── contains all 1,900 positive inputs
```

and:

```text
target_heatmap.npy
└── shape = (1900, 1, 121, 121)
    └── contains all 1,900 positive targets
```

## Negative

```text
negative_inputs/
├── negative_000000.npy
├── negative_000001.npy
├── ...
└── negative_001899.npy
```

and:

```text
negative_targets/
├── negative_000000.npy
├── negative_000001.npy
├── ...
└── negative_001899.npy
```

Each negative file has shape:

```text
(1, 121, 121)
```

Therefore, both sides contain 1,900 samples; only the storage organization differs.

---

# 13. Combined Manifest

Experiment 03 uses:

```text
center_detection_v2/combined_manifest.csv
```

as the primary combined sample manifest.

It connects the positive and negative samples and preserves their predefined dataset assignments.

The split assignments should not be randomly regenerated when reproducing the reported Experiment 03 results.

This is especially important when comparing Experiment 03 with subsequent experiments.

---

# 14. Train / Validation / Test Splits

The dataset uses fixed:

```text
train
validation
test
```

assignments.

The final Experiment 03 test set contains:

| Class            | Samples |
| ---------------- | ------: |
| Positive cyclone |     341 |
| Negative         |     341 |
| **Total**        | **682** |

The test set remains separate from model optimization and checkpoint selection.

---

# 15. Normalization

MSLP normalization statistics are calculated using **training data only**.

The resulting Experiment 03 statistics are stored with the trained model:

```text
models/
├── mslp_mean.npy
└── mslp_std.npy
```

Input MSLP is standardized conceptually as:

```text
normalized MSLP =
(MSLP - training mean) / training standard deviation
```

The same statistics must be used during validation, testing, and inference with the trained Experiment 03 model.

Using training-only statistics prevents validation or test information from leaking into preprocessing.

---

# 16. Model Inputs and Targets

For a positive cyclone sample:

```text
Input:
    MSLP
    shape = [1, 121, 121]
    dtype = float32

Targets:
    presence = 1

    center heatmap
    shape = [1, 121, 121]
    dtype = float32
```

For a negative sample:

```text
Input:
    MSLP
    shape = [1, 121, 121]
    dtype = float32

Targets:
    presence = 0

    no valid cyclone center
```

Presence classification is supervised using both positive and negative samples.

Center-localization supervision is applied to positive cyclone samples.

---

# 17. Verified Array Summary

| Data              |                           Number | Shape                 | dtype     |
| ----------------- | -------------------------------: | --------------------- | --------- |
| Positive MSLP     | 1 array containing 1,900 samples | `(1900, 1, 121, 121)` | `float32` |
| Positive heatmaps | 1 array containing 1,900 targets | `(1900, 1, 121, 121)` | `float32` |
| Latitude grid     |                                1 | `(121,)`              | `float32` |
| Longitude grid    |                                1 | `(121,)`              | `float32` |
| Negative inputs   |                      1,900 files | `(1, 121, 121)` each  | `float32` |
| Negative targets  |                      1,900 files | `(1, 121, 121)` each  | `float32` |

---

# 18. Dataset Package

The materialized Experiment 03 dataset is distributed separately from the GitHub repository.

The package is organized as:

```text
cyclone-center-detection-dataset/
│
├── DATASET_README.md
│
├── positive/
│   ├── latitude.npy
│   ├── longitude.npy
│   ├── manifest.csv
│   ├── mslp.npy
│   └── target_heatmap.npy
│
└── negative/
    ├── combined_manifest.csv
    ├── negative_manifest.csv
    ├── negative_materialized_manifest.csv
    ├── negative_inputs/
    └── negative_targets/
```

---

# 19. Local Repository Layout

After downloading the dataset, arrange the files locally as:

```text
cyclone-center-detection/
│
├── data/
│   ├── center_detection_v1/
│   │   ├── latitude.npy
│   │   ├── longitude.npy
│   │   ├── manifest.csv
│   │   ├── mslp.npy
│   │   └── target_heatmap.npy
│   │
│   └── center_detection_v2/
│       ├── combined_manifest.csv
│       ├── negative_manifest.csv
│       ├── negative_materialized_manifest.csv
│       ├── negative_inputs/
│       └── negative_targets/
│
├── models/
├── notebooks/
├── results/
└── docs/
```

The local `data/` directory is excluded from Git.

---

# 20. Additional Variables

The original positive materialization also contains:

```text
u850.npy
v850.npy
```

These variables are **not used by Experiment 03**.

They are intentionally excluded from the Experiment 03 dataset package.

They are relevant to subsequent experiments investigating whether low-level wind information provides complementary circulation information to MSLP.

---

# 21. Dataset Limitations

## Best-track dependence

Cyclone labels depend on best-track information and the cyclone-center auditing procedure.

## Negative-label interpretation

A negative sample means that the frame was selected as a non-cyclone example according to the dataset construction procedure.

It does not necessarily mean that no organized atmospheric disturbance exists.

Experiment 03 error analysis identified cases in which apparently negative frames contained organized low-pressure structure.

## Fixed geographic domain

The dataset uses a fixed:

```text
0°–30°N
75°–105°E
```

domain.

Models trained using this dataset should therefore be considered region-specific unless generalization to other regions is explicitly evaluated.

## ERA5 resolution

Cyclone structure is represented at ERA5 spatial resolution.

Fine-scale characteristics available from higher-resolution observational or operational products may therefore not be represented.

## Multiple frames per cyclone

Multiple samples can belong to the same cyclone system.

Individual atmospheric frames should therefore not automatically be interpreted as completely independent meteorological events.

---

# 22. Reproducibility

To reproduce Experiment 03:

1. Download the materialized dataset.
2. Place the positive data under `data/center_detection_v1/`.
3. Place the negative data under `data/center_detection_v2/`.
4. Preserve the split assignments in `combined_manifest.csv`.
5. Use training-only normalization statistics.
6. Run the Experiment 03 notebook.
7. Select the model using validation performance.
8. Evaluate the selected checkpoint on the untouched test split.

The primary experiment notebook is:

```text
notebooks/center_detection_experiment_03.ipynb
```

Reported results are documented in:

```text
docs/results.md
```
