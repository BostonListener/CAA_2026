# Segmentation Model

## Overview

This notebook trains an **11-channel SegFormer-B0** model for pixel-level archaeological site localization using multi-spectral satellite imagery and terrain data.

The segmentation stage takes positive candidate patches and predicts a spatial mask showing where the archaeological feature is located within each 100×100 patch.

Key features:

* Uses the same **11-channel input** produced by the data preparation pipeline
* Trains with **soft Gaussian masks (`sigma=1`)** and evaluates against hard binary masks
* Uses **group-aware train/validation/test splitting** to avoid site-level data leakage
* Adapts an ImageNet-pretrained **SegFormer-B0** from 3-channel RGB to 11-channel geospatial input
* Selects the final prediction threshold using the **validation set only**
* Supports checkpoint-based resume for interrupted Colab training

---

## Prerequisites

The notebook is designed to run in **Google Colab** with access to the Google Cloud Storage bucket created during data preparation.

Required inputs:

```text
gs://YOUR-BUCKET/dataset/positives.zarr
gs://YOUR-BUCKET/dataset/metadata/positives.parquet
```

A GPU runtime is recommended for training.

---

## Configuration

Before running the notebook, update the paths and experiment name in the **Configuration** section.

```python
# Input data
GCS_DATA_ROOT = "gs://YOUR-BUCKET/dataset"

# Experiment name
RUN_NAME = "YOUR-SEGMENTATION-RUN-NAME"

# Output location
GCS_RUN_ROOT = f"gs://YOUR-BUCKET/models/{RUN_NAME}"
```

The default training configuration uses:

```python
IMAGE_SIZE = 100
BATCH_SIZE = 16
MAX_EPOCHS = 20
TRAIN_FRAC = 0.70
VAL_FRAC = 0.15
TEST_FRAC = 0.15

PRETRAINED_MODEL = "nvidia/mit-b0"
ENCODER_LR = 2e-5
NEW_LAYER_LR = 1e-4
```

---

## Input Data

Each sample is a **100×100 image with 11 channels**:

| Channel | Description      |
| ------- | ---------------- |
| B2      | Blue             |
| B3      | Green            |
| B4      | Red              |
| B8      | Near-Infrared    |
| B11     | SWIR 1           |
| B12     | SWIR 2           |
| DEM     | Elevation        |
| Slope   | Terrain slope    |
| NDVI    | Vegetation index |
| NDWI    | Water index      |
| BSI     | Bare soil index  |

The notebook uses:

* `labels_soft_sigma1` as the **training target**
* `labels_hard` for **Dice / IoU evaluation**

Normalization statistics are computed from the **training split only** and reused for validation, testing, and inference.

---

## Usage

1. Open `02_CAA_2026_Segmentation_Model.ipynb` in Google Colab.
2. Update the GCS paths and `RUN_NAME` in the configuration cell.
3. Use a GPU runtime if available.
4. Run the notebook cells sequentially.
5. Review the final Dice, IoU, centroid error, prediction visualizations, and exported model files.

The notebook automatically handles:

* GCS authentication and dataset download
* Group-aware train/validation/test splitting
* Train-only normalization
* Spatial data augmentation
* SegFormer-B0 initialization and 11-channel adaptation
* Training and checkpointing
* Validation threshold search
* Final held-out test evaluation
* Prediction visualization
* Hugging Face model export

---

## Model

The model is based on **SegFormer-B0** (`nvidia/mit-b0`).

The original 3-channel input projection is expanded to accept all 11 geospatial channels. Pretrained RGB weights are transferred to B2, B3, and B4, while the remaining eight channels are initialized to zero and learned during training.

Training uses a weighted combination of:

* **Binary Cross-Entropy Loss**
* **Soft Dice Loss**

The final binary mask threshold is selected by maximizing Dice score on the validation set before evaluating the held-out test set.

---

## Outputs

Results are saved under:

```text
gs://YOUR-BUCKET/models/YOUR-SEGMENTATION-RUN-NAME/
```

Main outputs include:

```text
checkpoints/
  best_checkpoint.pt
  resume_checkpoint.pt

reports/
  run_config.json
  split.csv
  normalization_params.json
  threshold_search.csv
  test_metrics.json
  final_summary.json

figures/
  training_loss.png
  validation_metrics.png
  prediction_vs_label.png

hf_model/
  Hugging Face model files
  geoglyph_inference_config.json
```

The exported inference configuration stores the required channel order, normalization parameters, image size, and selected prediction threshold so the trained model can be reused consistently.

---

## Notes

* Keep the channel order fixed as:
  `B2, B3, B4, B8, B11, B12, DEM, Slope, NDVI, NDWI, BSI`.
* Train/validation/test splitting is performed at the archaeological-site group level to reduce spatial leakage.
* Validation and test data are never used to compute normalization statistics.
* The test set is not used to select the final segmentation threshold.
* If training is interrupted, the notebook can automatically restore the latest resume checkpoint from GCS.

---

## Support

For implementation details, refer to the inline documentation in `02_CAA_2026_Segmentation_Model.ipynb` or open an issue in the repository.