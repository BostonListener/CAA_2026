# Data Preparation Pipeline

## Overview

This notebook generates a complete dataset for archaeological site detection using satellite imagery and digital elevation data from Google Earth Engine. It creates:

- **Positive samples**: Archaeological sites with multiple position offsets and rotation augmentations
- **Integrated negatives**: Landscape samples from corners around each site
- **Landcover negatives**: Samples from urban, water, and cropland areas
- **Multiple label variants**: Hard (binary) and soft (probabilistic with σ=1,3,8) labels

The output is stored as Zarr arrays on Google Cloud Storage for efficient cloud-native access.

---

## Prerequisites

### 1. Google Cloud Platform Setup

You need three Google Cloud resources:

#### **GCP Project ID** (for Google Earth Engine)
1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select an existing one
3. Copy the **Project ID** (not the project name)
4. Enable Earth Engine API: [Earth Engine API](https://console.cloud.google.com/apis/library/earthengine.googleapis.com)

#### **GCS Bucket Name** (for dataset storage)
1. In Google Cloud Console, go to [Cloud Storage](https://console.cloud.google.com/storage)
2. Create a new bucket or select an existing one
3. Copy the bucket name (e.g., `my-archaeological-data`)
4. Note: Use only the bucket name, not `gs://bucket-name`

#### **Google Drive Working Directory**
1. In Google Drive, create a folder for this project (e.g., `CAA2026`)
2. Inside, create the following structure:
```
CAA2026/
├── config/
│   └── settings.yaml
├── inputs/
│   ├── known_sites.csv
│   └── contours.geojson
```
3. The path will be `/content/drive/My Drive/CAA2026` or `/content/drive/MyDrive/CAA2026` (depending on your Drive setup)

---

## Configuration

### User Configuration (Notebook)

In the **first code cell** of the notebook, update these values:

```python
# Your Google Drive working directory
WORKING_DIR = 'YOUR-GOOGLE-DRIVE-WORKING-DIRECTORY'

# Path to settings YAML (relative to WORKING_DIR)
CONFIG_FILE = 'config/settings.yaml'

# Your GCP project ID for Earth Engine
GEE_PROJECT_ID = 'YOUR-GCP-PROJECT-ID'

# Your GCS bucket name (without gs:// prefix)
GCS_BUCKET_NAME = 'YOUR-GCS-BUCKET-NAME'
```

**Example:**
```python
WORKING_DIR = '/content/drive/MyDrive/CAA2026'
CONFIG_FILE = 'config/settings.yaml'
GEE_PROJECT_ID = 'my-project-2026'
GCS_BUCKET_NAME = 'archaeological-dataset-2026'
```

### Region of Interest (YAML)

To change your study region, edit `config/settings.yaml`:

```yaml
# Study Region Definition
site_region:
  type: 'bbox'
  min_lat: -17.0    # Southern boundary
  max_lat: -7.0     # Northern boundary
  min_lon: -70.0    # Western boundary
  max_lon: -60.0    # Eastern boundary
```

**Important:** Also update landcover coordinates in the notebook (Section 6.1) to match your region. The default coordinates are for Peru/Bolivia. Modify the `get_base_coordinates()` function with locations from your study area.

---

## Input Files

### 1. `known_sites.csv`

CSV file containing known archaeological site locations.

**Required columns:**

| Column | Type | Description | Required |
|--------|------|-------------|----------|
| `site_id` | string | Unique identifier for the site | ✓ Yes |
| `latitude` | float | Latitude in decimal degrees (WGS84) | ✓ Yes |
| `longitude` | float | Longitude in decimal degrees (WGS84) | ✓ Yes |
| `site_type` | string | Type of site (e.g., oval, square, circular) | Optional |
| `source` | string | Data source reference | Optional |
| `a_width` | float | Site width dimension A in meters | Optional |
| `b_width` | float | Site width dimension B in meters | Optional |

**Example:**

```csv
site_id,latitude,longitude,site_type,source,a_width,b_width
abuno,-9.979387,-66.734946,oval,Kalliola_2024,342,315
abuov,-10.079579,-66.869467,oval,Kalliola_2024,220,210
acds2,-9.758053,-67.194505,square,GE_2024.08.20,175,
acds3,-9.698118,-67.213014,square,GE_2023.09.16,164,
acds4,-10.073635,-67.320639,square,GE_2014-11-17,200,
```

**Notes:**
- `a_width` and `b_width` are used to calculate site radius for position sampling
- If missing, defaults to 250m radius
- Empty values are allowed (leave blank after comma)

### 2. `contours.geojson`

GeoJSON file containing archaeological feature contours (boundaries, walls, ditches, etc.) used to generate ground truth labels.

**Format:** Standard GeoJSON FeatureCollection with LineString or MultiLineString geometries in WGS84 (EPSG:4326). Properties are optional and ignored.

**Example:**
```json
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": {
        "id": 17
      },
      "geometry": {
        "type": "LineString",
        "coordinates": [
          [-59.16888940047158, -9.430205927062993],
          [-59.16885795791349, -9.430202173254166],
          [-59.16883297377768, -9.430205407849273],
          [-59.16881315320752, -9.430213081388029],
          [-59.16879338735879, -9.430227739217898],
          [-59.16877498293594, -9.430252534104893]
        ]
      }
    }
  ]
}
```

**Note:** Coordinates are in [longitude, latitude] format.

---

## Usage

1. **Configure** user settings in the first notebook cell
2. **Upload** input files to Google Drive
3. **Run** all cells sequentially
4. **Monitor** progress - the pipeline supports resuming from interruptions
5. **Verify** output using the visualization tools in Section 8

The pipeline will create:
- `gs://YOUR-BUCKET/dataset/positives.zarr/`
- `gs://YOUR-BUCKET/dataset/integrated_negatives.zarr/`
- `gs://YOUR-BUCKET/dataset/landcover_negatives.zarr/`
- `gs://YOUR-BUCKET/dataset/metadata/`

---

## Example Output

![Example dataset sample](images/00_example.png)

*Example showing a positive sample with RGB composite and multiple label variants (hard + soft with different sigma values)*

---

## Support

For issues or questions, please refer to the inline documentation in the notebook or open an issue in the repository.