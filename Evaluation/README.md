# ReHoster Evalution Data

## Repository Overview

This repository serves as a data archive containing:
- Android firmware metadata from the FirmwareDroid project
- Build property data from various Android firmware versions
- Performance evaluation results from firmware re-hosting experiments
- Visualization outputs and analysis notebooks
- Service overview documentation

## Repository Structure
```
.
├── android_firmware.json    # Firmware metadata and partition information
├── build_properties.json    # Comprehensive build properties data
├── build_props.json         # Build properties (version 1)
├── build_props_v2.json      # Build properties (version 2)
├── build_props_v3.json      # Build properties (version 3)
├── post_injector_metrics_v1.json          # Post-injection performance metrics
├── DataImages.ipynb                        # Jupyter notebook for data analysis and visualization
├── Evaluation.xlsx                         # Evaluation results spreadsheet
├── Service_Overview.xlsx                   # Service overview documentation
├── images/                                 # Directory containing generated visualizations
└── results_builder/                        # Build and injection test results by version
    ├── 12/                                # Android 12 results
    ├── 12_1/                              # Android 12.1 results
    └── 13/                                # Android 13 results
```

## Usage

The main plots and data files are available as Jupyter Notebook: [DataImages.ipynb](DataImages.ipynb). The [Evaluation.xlsx](Evaluation.xlsx) contains for each sample the result data of success/failure for each task.


### Viewing the Data
The JSON files can be loaded and analyzed using any JSON parser. For Python:

```python
import json

# Load firmware metadata
with open('android_firmware.json', 'r') as f:
    firmware_data = json.load(f)

# Load build properties
with open('build_props_v3.json', 'r') as f:
    build_props = json.load(f)
```

### Running the Analysis Notebook
To run the analysis notebook:

```bash
jupyter notebook DataImages.ipynb
```

### Viewing Visualizations
The PDF files in the `images/` directory can be viewed with any PDF reader.

## Data Files

### Firmware Metadata

#### `android_firmware.json`
Contains metadata about analyzed Android firmware images, including:
- Firmware identification (filename, MD5 hash)
- Partition information (system, vendor, product, odm, etc.)
- Import success status for each partition
- File counts (firmware files, Android apps, build properties) per partition

### Build Properties

The repository contains multiple versions of build property data extracted from Android firmware:

#### `build_properties.json`
Comprehensive dataset of Android build properties with detailed property information.

#### `build_props.json`, `build_props_v2.json`, `build_props_v3.json`
Different versions of build property datasets containing information such as:
- System brand, device, manufacturer details
- Build fingerprints and version information
- CPU architecture details
- SDK versions and security patch levels
- Dalvik VM configuration
- Product properties (locale, ABI, etc.)

### Performance Metrics

#### `post_injector_metrics_v1.json`
Performance metrics related to post-build injection processes, tracking various performance indicators.

### Analysis and Documentation

#### `DataImages.ipynb`
Jupyter notebook containing data analysis code and visualizations including:
- Dataset overview
- Firmware import metrics
- Firmware meta-data analysis
- Re-hosting performance evaluation
- Sanity checks and coverage ratios

#### `Evaluation.xlsx`
Spreadsheet containing evaluation results from experiments.

#### `Service_Overview.xlsx`
Documentation about service configurations and overviews.

## Results Directory

The `results_builder/` directory contains experimental results organized by Android version:

### Structure
Each version subdirectory (e.g., `12/`, `12_1/`, `13/`) contains:

- **`results_build_time.json`**: Build duration metrics including:
  - Hostname of build server
  - Firmware ID
  - Build duration in seconds
  - Build status (success/failure)

- **`results_build_injection.json`**: Metrics related to build-time code injection experiments

- **`results_post_build_injections.json`**: Metrics related to post-build injection processes

### Versions
- **12/**: Results for Android 12 (API level 31)
- **12_1/**: Results for Android 12.1 (API level 32)
- **13/**: Results for Android 13 (API level 33)

## Images Directory

The `images/` directory contains PDF visualizations generated from the data analysis:

- **Dataset visualizations**:
  - `Dataset_brand_manufacturer_mixed.pdf`: Distribution of brands and manufacturers
  - `Dataset_ro_product_system_manufacturer.pdf`: System manufacturer distribution

- **Performance visualizations**:
  - `build_durations_boxplot.pdf`: Build duration statistics
  - `post_build_durations_boxplot.pdf`: Post-build injection duration statistics
  - `coverage_ratios.pdf`: Coverage analysis results

- **Version-specific duration plots**:
  - `v12_Duration_Builds.pdf` / `v12_Duration_Post_Injection.pdf`: Android 12 metrics
  - `v12_1_Duration_Builds.pdf` / `v12_1_Duration_Post_Injection.pdf`: Android 12.1 metrics
  - `v13_Duration_Builds.pdf` / `v13_Duration_Post_Injection.pdf`: Android 13 metrics
  - `SDK32_Duration_Builds.pdf` / `SDK32_Duration_Post_Injection.pdf`: SDK 32 metrics

