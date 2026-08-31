# PCB Anomaly Detection with PatchCore

A computer vision pipeline for **automatic anomaly detection on Printed Circuit Boards (PCBs)** using **PatchCore** and image-based alignment.

The pipeline learns the visual characteristics of **healthy PCB images** and detects abnormal regions in new boards without requiring a large labeled defect dataset.

---

## Overview

The system processes PCB images through the following pipeline:

```text
Raw PCB Images
      │
      ▼
PCB Detection
      │
      ▼
Crop & Rotation
      │
      ▼
ORB Feature Matching
      │
      ▼
Homography Alignment
      │
      ▼
Image-Level Dataset Split
      │
      ├───────────────┐
      ▼               ▼
   Training       Calibration
      │               │
      ▼               ▼
   PatchCore      Threshold
      │
      ▼
   Holdout Evaluation
      │
      ▼
   New Image Inference
      │
      ▼
   Anomaly Score
      │
      ▼
   Anomaly Heatmap
```

The main goal is not only to determine whether a PCB is **normal or anomalous**, but also to provide a visual indication of **where the anomaly is located**.

---

## Key Features

* PCB detection and automatic cropping
* Automatic orientation correction
* ORB feature-based image alignment
* Homography estimation with RANSAC
* Image-level dataset splitting
* Overlapping `512 × 512` image tiling
* PatchCore-based anomaly detection
* Calibration-based anomaly threshold
* Holdout evaluation
* Image-level anomaly scoring
* Pixel-level anomaly heatmaps
* Batch inference on new PCB images
* Reusable trained model checkpoint

---

## Method

### 1. PCB Detection

The input image is converted to HSV color space and the PCB region is detected using green-color masks.

The largest valid green contour is considered the PCB region.

### 2. Crop and Alignment

After PCB detection, the board is cropped and rotated into a consistent orientation.

ORB features are then extracted from both the input board and a healthy reference image.

Feature matching followed by **RANSAC Homography** is used to align the board with the reference coordinate system.

This step is important because the same PCB components should appear at approximately the same locations across images.

### 3. Image-Level Dataset Split

Successfully aligned images are divided before tiling into:

```text
70%  → Training
15%  → Calibration
15%  → Holdout
```

The split is performed at the **image level**, preventing tiles from the same PCB image from being distributed across different subsets.

### 4. Tiling

Each aligned PCB image is divided into overlapping:

```text
512 × 512
```

tiles with:

```text
128 pixel overlap
```

The overlap helps preserve information around tile boundaries.

### 5. PatchCore

PatchCore is trained using healthy PCB images.

Current configuration:

```text
Backbone:        Wide ResNet-50-2
Feature Layers:  layer2, layer3
Coreset Ratio:   0.01
Nearest Neighbors: 9
```

The model learns a representation of normal PCB appearance and uses feature-space distances to identify unusual regions.

### 6. Threshold Calibration

For each PCB image, the image-level anomaly score is calculated from the mean score of the highest-scoring tiles.

The current configuration uses:

```text
Top-K tiles:     2
Calibration:     99th percentile
```

The resulting threshold is used to classify new PCB images as normal or anomalous.

### 7. Holdout Evaluation

A separate set of healthy PCB images is used to evaluate the calibrated threshold.

This provides a simple measurement of false positives on images that were not used during training or threshold calibration.

### 8. Heatmap

Tile-level anomaly maps are projected back into the original PCB coordinate system.

Overlapping tile predictions are averaged to produce a full-image anomaly heatmap.

The final output combines:

```text
Aligned PCB + Anomaly Heatmap
```

This provides an intuitive visual representation of suspicious regions.

---

## Project Structure

A typical project layout is:

```text
pcb-anomaly-detection/
│
├── PCB_PatchCore.ipynb
├── README.md
├── requirements.txt
├── .gitignore
│
├── raw_data/              # local training/reference data
├── raw_test_data/         # local test images
├── outputs/               # generated results
└── patchcore_tr83031.ckpt # trained model, kept outside Git
```

Large datasets, generated outputs, and model checkpoints are intentionally excluded from the Git repository.

---

## Installation

Install the required packages with:

```bash
pip install "anomalib==2.6.0"
```

Depending on the environment, additional scientific Python packages such as OpenCV, NumPy, Pandas, and Matplotlib may already be installed.

For a reproducible environment, use the provided `requirements.txt`.

---

## Data Preparation

Place the healthy PCB images in the configured raw-data directory.

The reference image should be named:

```text
Healthy.png
```

The reference image is used for geometric alignment.

Example:

```text
raw_data/
├── Healthy.png
├── image_001.jpg
├── image_002.jpg
├── image_003.jpg
└── ...
```

New PCB images for inference can be placed in a separate test directory.

---

## Running the Pipeline

Open:

```text
PCB_PatchCore.ipynb
```

and run the notebook from top to bottom.

The pipeline will:

1. validate the input data
2. align the PCB images
3. split the dataset
4. generate image tiles
5. train PatchCore
6. calibrate the anomaly threshold
7. evaluate the holdout set
8. generate anomaly heatmaps
9. run inference on new PCB images

---

## Inference

Once the model has been trained, new PCB images can be tested without retraining.

The inference stage performs:

```text
New Image
   ↓
PCB Detection
   ↓
Alignment
   ↓
Tiling
   ↓
PatchCore Prediction
   ↓
Image Score
   ↓
Normal / Anomalous
   ↓
Heatmap
```

The final result contains both an anomaly score and a visual localization of suspicious regions.

---

## Example Result

The system can produce an output similar to:

```text
Image Score: 0.627987
Threshold:   0.431276
Prediction:  Anomalous
```

and a corresponding heatmap highlighting the regions that contributed most strongly to the anomaly score.

---

## Evaluation Example

During one holdout evaluation on healthy images, the pipeline produced:

```text
Holdout healthy images: 13
False positives:        1
False positive rate:    0.077
```

This result is dataset-specific and should not be interpreted as a general benchmark for PatchCore.

---

## Model Output

The trained model is saved as a PyTorch Lightning checkpoint:

```text
patchcore_tr83031.ckpt
```

The checkpoint can be reused for inference without retraining the model, provided that the same preprocessing and model configuration are used.

---

## Important Notes

This project is designed for PCB images with a reasonably consistent physical layout.

Alignment quality is important because the anomaly detector assumes that corresponding PCB regions appear in approximately the same locations.

The current preprocessing and threshold values were selected for this PCB dataset and may need adjustment for other boards, cameras, lighting conditions, or image resolutions.

---

## Technologies

* Python
* OpenCV
* NumPy
* Pandas
* Matplotlib
* PyTorch
* Anomalib
* PatchCore
* ORB
* RANSAC Homography

---

## License

Add the license appropriate for your project here.

---

## Acknowledgements

This project uses the PatchCore implementation provided through the Anomalib framework.

For research and implementation details, refer to the original PatchCore paper and the Anomalib project.
