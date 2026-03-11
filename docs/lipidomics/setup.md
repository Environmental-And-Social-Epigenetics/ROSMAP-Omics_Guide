# Environment Setup

## Clone the Repository

```bash
git clone https://github.com/Environmental-And-Social-Epigenetics/ROSMAP-SI-Lipidomics.git
cd ROSMAP-SI-Lipidomics
```

## Install Python Dependencies

The pipeline requires Python 3.8 or later. Install all dependencies from the provided requirements file:

```bash
python -m pip install -r requirements.txt
```

The required packages are:

| Package | Purpose |
|---------|---------|
| `jupyter` | Notebook execution environment |
| `matplotlib` | Plotting backend for figures |
| `numpy` | Numerical array operations |
| `pandas` | Data frame manipulation |
| `scikit-learn` | Local Outlier Factor (LOF) outlier detection |
| `scipy` | Shapiro-Wilk normality tests, statistical functions |
| `seaborn` | Statistical visualization (volcano plots, scatter plots) |
| `statsmodels` | OLS regression and ANCOVA models |
| `rpy2` | Optional; bridge for executing R code from Python |

The `rpy2` package is optional and only needed if you intend to call R-based analysis functions from within Python notebooks or scripts. It requires a working R installation.

## Data Acquisition

The pipeline expects four CSV files placed in specific subdirectories under `data/raw/`. These files are not included in the repository and must be obtained separately.

### Required Input Files

| File | Path | Description |
|------|------|-------------|
| SERRF-normalized lipidomics matrix | `data/raw/lipidomics_data/ROSMAP_Brain_SERRF_normalization_internal.csv` | Primary lipidomics data with lipid labels, mode, lipid class, and sample intensity columns |
| Longitudinal phenotype metadata | `data/raw/metadata/dataset_652_basic_03-23-2022.csv` | ROSMAP longitudinal data including `projid`, `social_isolation_avg`, `niareagansc`, `msex`, `age_death`, `educ`, medication columns, and `pmi` |
| Clinical metadata | `data/raw/metadata/ROSMAP_clinical.csv` | Clinical data providing the `projid` to `individualID` mapping |
| Specimen-to-individual map | `data/raw/metadata/Lip_Ind_Map.csv` | Mapping between lipidomics specimen IDs and ROSMAP individual IDs |

### Optional Reference Files

| File | Path | Description |
|------|------|-------------|
| Historical processed output | `data/raw/original_processed_csvs/Final_Formated_Lipidomics.csv` | Reference output from a prior workflow (for validation) |
| Historical normalized output | `data/raw/original_processed_csvs/Normalized_Formatted_Lipidomics.csv` | Reference normalized output (for validation) |

### Setting Up the Directory Structure

Create the required directories and copy your data files into them:

```bash
mkdir -p data/raw/lipidomics_data data/raw/metadata data/raw/original_processed_csvs
```

Then copy or symlink the four required CSV files into the appropriate subdirectories.

### How to Obtain the Data

If you do not already have access to the raw ROSMAP lipidomics and metadata files:

1. Contact the project maintainers or data owners listed in the repository.
2. Request the four required CSVs listed above.
3. Place them in the `data/raw/` subdirectories as shown.

If you are working in an environment where a shared internal data source exists, copy files directly from that location.

!!! note "Data are not version-controlled"
    Raw and processed CSV files are intentionally excluded from version control via `.gitignore`. The repository tracks code and documentation only. Data management follows the access policies of the ROSMAP consortium.

## Configuration

The file `config.py` in the repository root defines all path constants used throughout the pipeline. You do not need to edit this file unless you change the directory structure.

Key settings in `config.py`:

```python
PROJECT_ROOT = Path(__file__).resolve().parent

DATA_DIR = PROJECT_ROOT / "data"
DATA_RAW_DIR = DATA_DIR / "raw"
DATA_PROCESSED_DIR = DATA_DIR / "processed"

RESULTS_DIR = PROJECT_ROOT / "results"
FIGURES_DIR = RESULTS_DIR / "figures"
TABLES_DIR = RESULTS_DIR / "tables"
```

Canonical input filenames are also defined as constants:

```python
RAW_LIPIDOMICS_FILENAME = "ROSMAP_Brain_SERRF_normalization_internal.csv"
META_LONGITUDINAL_FILENAME = "dataset_652_basic_03-23-2022.csv"
META_CLINICAL_FILENAME = "ROSMAP_clinical.csv"
LIPID_ID_MAP_FILENAME = "Lip_Ind_Map.csv"
```

The `ensure_project_dirs()` function creates the expected folder tree if any directories are missing. It is called automatically at the start of each script.

## Verifying the Setup

After installing dependencies and placing the data files, verify the setup by running the first step:

```bash
python scripts/01_data_processing.py
```

If the step completes without errors and produces `data/processed/Final_Formatted_Lipidomics.csv`, the environment is correctly configured.
