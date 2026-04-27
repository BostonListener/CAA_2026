# Classification Model

## Overview

This notebook trains a CNN-based binary classifier for archaeological site detection using multi-spectral satellite imagery and terrain data. The model:

- **Trains on Amazon region** data with balanced positive/negative sampling
- **Evaluates on held-out test sets** including cross-geography transfer (Central Asia)
- **Optimizes for high recall** while maintaining precision (critical for site discovery)
- **Supports few-shot transfer learning** for new regions with limited labels

The trained model achieves strong performance on Amazon test data and demonstrates effective transfer to Central Asia with minimal fine-tuning.

---

## Prerequisites

### 1. Google Cloud Platform Setup

You need the same GCP resources as the data preparation pipeline:

#### **GCP Project ID** (for authentication)
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Use the same project from data preparation
3. Ensure you're authenticated (notebook handles this automatically in Colab)

#### **GCS Bucket Name** (for accessing datasets)
1. Use the same bucket containing your prepared datasets:
   - `gs://YOUR-BUCKET/dataset/positives.zarr`
   - `gs://YOUR-BUCKET/dataset/integrated_negatives.zarr`
   - `gs://YOUR-BUCKET/dataset/landcover_negatives.zarr` or Landcover negatives tar archives in `gs://YOUR-BUCKET/packs/`

2. The notebook will also save checkpoints and results here:
   - `gs://YOUR-BUCKET/stage1_runs/YOUR-RUN-NAME/`

---

## Configuration

### User Configuration (First Code Cell)

In the **Configuration** section, update these values:

```python
# ============================================================================
# MODIFY THESE SETTINGS FOR YOUR ENVIRONMENT
# ============================================================================

# GCS bucket containing landcover-based negative samples (tar archives)
GCS_TAR_BUCKET = "YOUR-TAR-BUCKET-NAME"

# GCS bucket containing positive samples and integrated negatives (Zarr format)
GCS_ZARR_BUCKET = "YOUR-ZARR-BUCKET-NAME"

# Name for this training run (used in checkpoint and report filenames)
STAGE1_RUN_NAME = "YOUR-RUN-NAME"
```

**Example:**
```python
GCS_TAR_BUCKET = "caa-2026-dataset"
GCS_ZARR_BUCKET = "caa-2026"
STAGE1_RUN_NAME = "amazon_classifier_v1"
```

### Training Hyperparameters

The default configuration is optimized for archaeological site detection. You can modify these if needed:

```python
BATCH_SIZE = 64              # Number of samples per training batch
NUM_EPOCHS = 8               # Total training epochs
LEARNING_RATE = 1e-4         # Initial learning rate
WEIGHT_DECAY = 1e-4          # L2 regularization strength
POS_WEIGHT = 2.0             # Positive class weight (>1 improves recall)
```

**Training Split:**
```python
TRAIN_FRAC = 0.70           # 70% for training
VAL_FRAC = 0.15             # 15% for validation
TEST_FRAC = 0.15            # 15% for testing
```

---

## Input Data

### Required Datasets

The notebook expects datasets prepared by the data preparation pipeline:

#### **1. Amazon Training Data**

From GCS Zarr stores:
- **Positives**: `gs://YOUR-BUCKET/dataset/positives.zarr`
- **Integrated Negatives**: `gs://YOUR-BUCKET/dataset/integrated_negatives.zarr`
- **Metadata**: Parquet files in `gs://YOUR-BUCKET/dataset/metadata/`

From GCS tar archives:
- **Landcover Negatives**: `gs://YOUR-BUCKET/dataset/landcover_negatives.zarr` or `gs://YOUR-BUCKET/packs/lneg_*.tar`

#### **2. Central Asia Evaluation Data (Optional)**

Downloaded from Hugging Face Hub:
- Dataset: `lldbrett/archaeological-sites-central-asia`
- Format: RAR archive containing grid images
- Used for transfer learning evaluation

### Data Structure

Each sample is a 100×100 pixel grid with 11 channels:

| Channel | Description | Source |
|---------|-------------|--------|
| B2, B3, B4 | Sentinel-2 RGB (Blue, Green, Red) | Sentinel-2 |
| B8 | Near-Infrared (NIR) | Sentinel-2 |
| B11, B12 | Shortwave Infrared (SWIR1, SWIR2) | Sentinel-2 |
| DEM | Digital Elevation Model | NASADEM |
| Slope | Terrain slope (degrees) | Derived from DEM |
| NDVI | Normalized Difference Vegetation Index | Calculated |
| NDWI | Normalized Difference Water Index | Calculated |
| BSI | Bare Soil Index | Calculated |

---

## Usage

### Quick Start

1. **Configure** paths in the first code cell
2. **Run all cells** sequentially - the pipeline is fully automated
3. **Monitor training** progress in real-time
4. **Review results** in the final evaluation section

### Pipeline Stages

The notebook executes the following stages automatically:

#### **Stage 1: Data Loading**
- Downloads datasets from GCS
- Extracts tar archives
- Builds unified sample index
- Creates train/val/test splits (group-aware to prevent leakage)

#### **Stage 2: Preprocessing**
- Computes normalization statistics from training set
- Applies z-score standardization
- Validates data quality

#### **Stage 3: Model Training** (GPU recommended)
- Trains CNN classifier with:
  - Class-balanced sampling
  - Data augmentation (flips, rotations, noise)
  - Automatic mixed precision (AMP) for speed
  - Learning rate scheduling
- Saves best checkpoint based on validation metrics

#### **Stage 4: Evaluation**
- Evaluates on validation, test, and Central Asia sets
- Reports metrics at multiple thresholds:
  - **Best F1**: Balanced precision/recall
  - **Best F2**: Recall-weighted (2× importance)
  - **Best Precision @ Recall ≥ 0.85**: High recall guarantee
- Exports predictions and metrics

#### **Stage 5: Few-Shot Transfer (Optional)**
- Tests transfer learning with limited Central Asia labels
- Evaluates performance at 1%, 10%, 25%, 50% data fractions
- Measures label efficiency

---

## Output Files

### Checkpoints

Saved to `gs://YOUR-BUCKET/stage1_runs/YOUR-RUN-NAME/`:

- `YOUR-RUN-NAME_best.pth`: Best validation performance
- `YOUR-RUN-NAME_last.pth`: Final epoch

**Checkpoint contents:**
- Model weights
- Optimizer state
- Training configuration
- Best metric value and thresholds

### Reports

Saved to `/content/reports/`:

- `YOUR-RUN-NAME_metrics.json`: Comprehensive metrics (programmatic access)
- `YOUR-RUN-NAME_report.md`: Human-readable summary tables
- `YOUR-RUN-NAME_history.json`: Training history (loss, metrics per epoch)
- `YOUR-RUN-NAME_train_channel_stats.json`: Normalization parameters

### Predictions

Saved to `/content/reports/`:

- `YOUR-RUN-NAME_val_predictions.csv`
- `YOUR-RUN-NAME_test_predictions.csv`
- `YOUR-RUN-NAME_central_asia_predictions.csv`

Each CSV contains:
- Sample metadata (sample_id, group_id, source, etc.)
- True label (`y_true`)
- Predicted probability (`y_prob`)
- Binary predictions at three operating thresholds

### Data Splits

Saved to `/content/splits/`:

- `YOUR-RUN-NAME_amazon_split.csv`: Full dataset with split assignments
- `YOUR-RUN-NAME_train.csv`: Training set
- `YOUR-RUN-NAME_val.csv`: Validation set
- `YOUR-RUN-NAME_test.csv`: Test set

---

## Understanding Results

### Operating Thresholds

The model reports performance at three operating points:

| Threshold | Optimizes For | Use Case |
|-----------|---------------|----------|
| **Best F1** | Balanced precision/recall | General site detection |
| **Best F2** | Recall (2× weight) | Minimize missed sites |
| **Precision @ Recall ≥ 0.85** | High recall + best precision | Survey planning (ensure 85%+ coverage) |

### Key Metrics

**Overall Performance:**
- **AUC** (Area Under ROC Curve): Model's ability to rank sites vs non-sites
- **AP** (Average Precision): Summary of precision-recall curve

**Classification Metrics:**
- **Precision**: Of predictions labeled "site", what % are correct?
- **Recall**: Of actual sites, what % did we find?
- **F1/F2**: Harmonic means balancing precision and recall
- **Specificity**: Of actual non-sites, what % did we correctly identify?

---

## Advanced Usage

### Resuming Training

If training is interrupted, the notebook can resume from the last checkpoint:

```python
# Set this before the training loop
RESUME_FROM_CHECKPOINT = True
RESUME_CHECKPOINT_PATH = "/content/checkpoints/YOUR-RUN-NAME_last.pth"
```

### Adjusting for Class Imbalance

If your dataset has severe imbalance, adjust the positive class weight:

```python
POS_WEIGHT = 3.0  # Higher values = more emphasis on finding sites
```

---

## Model Architecture

### Network Structure

**Input:** 11 channels × 100×100 pixels  
**Architecture:** Custom CNN with progressive downsampling

```
Input                          → 11 × 100 × 100

Conv Block 1 (11→24→24)        → 24 × 100 × 100
MaxPool                        → 24 × 50 × 50

Conv Block 2 (24→48→48)        → 48 × 50 × 50
MaxPool                        → 48 × 25 × 25

Conv Block 3 (48→96→96)        → 96 × 25 × 25
MaxPool                        → 96 × 12 × 12

Conv Block 4 (96→144→144)      → 144 × 12 × 12

AdaptiveAvgPool                → 144 features
AdaptiveMaxPool                → 144 features
Concatenate                    → 288 features

Classifier (288→128→32→1)      → Binary logit
```

**Total Parameters:** ~500K (lightweight, fast training)

### Design Rationale

- **Dual pooling**: Captures both average and peak responses
- **Progressive channels**: Efficiently extracts features at multiple scales
- **Lightweight**: Relatively small CNN for efficient training and inference
- **Regularization**: Dropout (0.20, 0.10) and batch normalization

---

## Troubleshooting

### Common Issues

#### **"Best checkpoint not found"**
- Ensure GCS paths are correct
- Check authentication with `gcloud auth list`
- Verify bucket permissions

#### **"Missing required names" error**
- Run cells sequentially from the beginning
- Don't skip the configuration section

#### **Out of memory**
- Reduce `BATCH_SIZE` (try 32 or 16)
- Enable CPU-only mode: `device = torch.device("cpu")`
- Reduce `NUM_WORKERS` (try 0 or 1)

#### **Poor validation performance**
- Check class balance in training set
- Adjust `POS_WEIGHT` if recall is low
- Verify normalization statistics are computed correctly
- Ensure data augmentation is enabled (`training=True` in dataset)

#### **Training instability (NaN loss)**
- Reduce learning rate: `LEARNING_RATE = 5e-5`
- Check for corrupted data samples
- Verify input normalization

### GPU Recommendations

- **Recommended:** T4, V100, or better (Colab Pro)
- **Memory:** 8GB+ GPU RAM for batch_size=64

---

## Support

For issues or questions:
- Refer to inline documentation in the notebook
- Check the [troubleshooting section](#troubleshooting) above
- Open an issue in the repository