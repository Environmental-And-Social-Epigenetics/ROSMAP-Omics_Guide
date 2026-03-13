# Data Access and Entry Points

After cloning the repository, the `Data/` directory is the canonical local location for all pipeline data. Only the small clinical phenotype CSVs (`Data/Phenotypes/`) are tracked in git and available immediately. All transcriptomics data — FASTQs, CellRanger outputs, CellBender outputs, and processing outputs — must be populated using the scripts in `Data_Access/`.

This page explains how to populate `Data/` at any stage so you can enter the pipeline wherever you need to.

## Canonical Data Layout

```
ROSMAP-SingleNucleusRNAseq/
├── Data/
│   ├── Phenotypes/                          # Tracked in git — available immediately
│   │   ├── ROSMAP_clinical.csv
│   │   ├── dataset_652_basic_12-23-2021.csv
│   │   ├── TSAI_DEJAGER_all_patients_wACEscores.csv
│   │   └── DeJager_ID_Map.csv
│   └── Transcriptomics/
│       ├── Tsai/
│       │   ├── FASTQs/                      # 480 patient directories (~9 TB)
│       │   ├── Cellranger_Output/           # 480 sample directories
│       │   ├── Cellbender_Output/           # 478 sample directories
│       │   └── Processing_Outputs/          # QC-filtered, doublet-removed, annotated
│       └── DeJager/
│           ├── FASTQs/                      # 127 library directories
│           ├── Cellranger_Output/           # ~47 sample directories
│           ├── Cellbender_Output/           # ~131 sample directories
│           └── Processing_Outputs/          # QC-filtered, doublet-removed, annotated
├── Data_Access/                             # Transfer scripts for populating Data/
│   ├── README.md                            # Full data location reference
│   ├── Transcriptomics/
│   │   ├── Tsai_Server/                     # SFTP transfers to/from NAS
│   │   └── Engaging-Openmind_Transfer/      # Globus transfers between clusters
│   └── Phenotypes/
└── config/paths.sh                          # All path variables
```

## Choose Your Starting Point

You do not need to run every pipeline phase. The table below shows what to download depending on where you want to start:

| Starting Point | What You Skip | Disk Space Needed | Software Required | Next Step |
|---|---|---|---|---|
| **FASTQs** | Nothing | ~9 TB (Tsai) or ~10 TB (DeJager) | Cell Ranger, CellBender (GPU), all conda envs | [Cell Ranger](preprocessing/cellranger.md) |
| **CellRanger output** | FASTQ acquisition + alignment | ~1–2 TB per dataset | CellBender (GPU), processing conda envs | [CellBender](preprocessing/cellbender.md) |
| **CellBender output** | All preprocessing | ~0.5–1 TB per dataset | Processing conda envs only (no GPU) | [QC Filtering](processing/qc-filtering.md) |
| **Annotated H5ad** | Preprocessing + Processing | ~83 GB (Tsai) | Analysis conda envs only | [Analysis](analysis/index.md) |

---

### Entry Point A: Start from FASTQs

If you want to run the full pipeline from raw sequencing reads.

=== "Tsai"

    The Tsai FASTQs (~9 TB across 480 patients) can be obtained via:

    | Method | Command |
    |--------|---------|
    | **NAS download** (recommended) | `cd Data_Access/Transcriptomics/Tsai_Server/Tsai && DATA_TYPES=FASTQs ./download_from_nas.sh` |
    | **Globus from Engaging** | `cd Data_Access/Transcriptomics/Engaging-Openmind_Transfer/Tsai && ./send_globus.sh` |

    After populating `Data/Transcriptomics/Tsai/FASTQs/`, proceed to [Cell Ranger](preprocessing/cellranger.md).

=== "DeJager"

    The DeJager FASTQs (~10 TB across ~127 libraries) can be obtained via:

    | Method | Command |
    |--------|---------|
    | **Synapse download** | `sbatch Preprocessing/DeJager/01_FASTQ_Download/Download_FASTQs.sh` |
    | **NAS download** | `cd Data_Access/Transcriptomics/Tsai_Server/DeJager && DATA_TYPES=FASTQs ./download_from_nas.sh` |

    Synapse requires an account with access to project `syn21438684` and a personal access token. See [FASTQ Acquisition](preprocessing/fastq-acquisition.md) for setup details.

    After populating `Data/Transcriptomics/DeJager/FASTQs/`, proceed to [Cell Ranger](preprocessing/cellranger.md).

---

### Entry Point B: Start from CellRanger Output

If you want to skip FASTQ alignment and start from count matrices.

=== "Tsai"

    ```bash
    cd Data_Access/Transcriptomics/Tsai_Server/Tsai
    DATA_TYPES=Cellranger_Output ./download_from_nas.sh
    ```

=== "DeJager"

    ```bash
    cd Data_Access/Transcriptomics/Tsai_Server/DeJager
    DATA_TYPES=Cellranger_Output ./download_from_nas.sh
    ```

After populating `Data/Transcriptomics/{dataset}/Cellranger_Output/`, proceed to [CellBender](preprocessing/cellbender.md).

!!! note "You still need a GPU"
    CellBender requires a CUDA-capable GPU. If you do not have GPU access, consider starting from CellBender output instead.

---

### Entry Point C: Start from CellBender Output

If you want to skip all preprocessing and start directly with the processing pipeline. This is the most common entry point for users who want to run their own QC, integration, and annotation.

=== "Tsai"

    ```bash
    cd Data_Access/Transcriptomics/Tsai_Server/Tsai
    DATA_TYPES=Cellbender_Output ./download_from_nas.sh
    ```

=== "DeJager"

    ```bash
    cd Data_Access/Transcriptomics/Tsai_Server/DeJager
    DATA_TYPES=Cellbender_Output ./download_from_nas.sh
    ```

After populating `Data/Transcriptomics/{dataset}/Cellbender_Output/`, proceed to [QC Filtering (Stage 1)](processing/qc-filtering.md).

!!! tip "No GPU or Cell Ranger needed"
    Starting from CellBender output requires only the processing conda environments. No GPU, Cell Ranger installation, or reference genome download is needed.

---

### Entry Point D: Start from the Annotated H5ad

If you want to skip straight to downstream analysis (differential expression, SCENIC, TF activity).

=== "Tsai"

    ```bash
    cd Data_Access/Transcriptomics/Tsai_Server/Tsai
    ./download_processing_outputs_from_nas.sh 03_Integrated
    ```

    This downloads `tsai_annotated.h5ad` (~83 GB) into `Data/Transcriptomics/Tsai/Processing_Outputs/03_Integrated/`.

=== "DeJager"

    ```bash
    cd Data_Access/Transcriptomics/Tsai_Server/DeJager
    ./download_processing_outputs_from_nas.sh 03_Integrated
    ```

    This downloads `dejager_annotated.h5ad` into `Data/Transcriptomics/DeJager/Processing_Outputs/03_Integrated/`.

After downloading, proceed to [Analysis](analysis/index.md). You will also need the phenotype data in `Data/Phenotypes/` (already available after cloning).

---

## Quick Reference: Where Is My Data?

| Data | Path Variable | Primary Location | Backup | How to Get It |
|------|--------------|-----------------|--------|---------------|
| Tsai FASTQs | `$TSAI_FASTQS` | Engaging cluster | NAS | Globus or NAS download |
| DeJager FASTQs | `$DEJAGER_FASTQS_DIR` | Synapse (`syn21438684`) | NAS | Synapse or NAS download |
| Tsai CellRanger | `$TSAI_CELLRANGER` | `Data/Transcriptomics/Tsai/Cellranger_Output/` | NAS | `download_from_nas.sh` |
| DeJager CellRanger | `$DEJAGER_CELLRANGER` | `Data/Transcriptomics/DeJager/Cellranger_Output/` | NAS | `download_from_nas.sh` |
| Tsai CellBender | `$TSAI_CELLBENDER` | `Data/Transcriptomics/Tsai/Cellbender_Output/` | NAS | `download_from_nas.sh` |
| DeJager CellBender | `$DEJAGER_CELLBENDER` | `Data/Transcriptomics/DeJager/Cellbender_Output/` | NAS | `download_from_nas.sh` |
| Tsai Annotated | `$TSAI_INTEGRATED` | `Data/Transcriptomics/Tsai/Processing_Outputs/03_Integrated/` | NAS | `download_processing_outputs_from_nas.sh` |
| DeJager Annotated | `$DEJAGER_INTEGRATED` | `Data/Transcriptomics/DeJager/Processing_Outputs/03_Integrated/` | NAS | `download_processing_outputs_from_nas.sh` |
| Phenotypes | `$PHENOTYPE_DIR` | `Data/Phenotypes/` | NAS | Already in repo (git-tracked) |

All path variables are defined in `config/paths.sh`. Source it before using any scripts:

```bash
source config/paths.sh
```

## Transfer Methods

### NAS (SFTP) — Primary Backup

The Tsai Lab NAS (`tsailabnas.mit.edu`) mirrors the local `Data/` structure and is the recommended source for downloading data.

**Prerequisites:**

1. Create a password file at `~/.smb_tsailabnas`:
    ```
    username = YOUR_USERNAME
    password = YOUR_PASSWORD
    ```
2. `chmod 600 ~/.smb_tsailabnas`
3. `sshpass` must be available (installed system-wide on OpenMind)

**Scripts** are located in `Data_Access/Transcriptomics/Tsai_Server/{Tsai,DeJager}/`:

- `download_from_nas.sh` — download FASTQs, CellRanger, or CellBender output
- `upload_to_nas.sh` — upload data to NAS
- `download_processing_outputs_from_nas.sh` — download processing outputs (QC, doublets, annotated)
- `upload_processing_outputs_to_nas.sh` — upload processing outputs to NAS

### Globus — Cluster-to-Cluster Transfers

Globus is recommended for large inter-cluster transfers (e.g., Engaging to Openmind).

**Prerequisites:**

1. Activate the Globus conda environment
2. Run `globus login` (one-time setup)
3. Verify with `globus whoami`

**Endpoint IDs:**

| Cluster | Endpoint ID |
|---------|-------------|
| Openmind | `cbc6f8da-d37e-11eb-bde9-5111456017d9` |
| Engaging | `c52fcff2-761c-11eb-8cfc-cd623f92e1c0` |

**Scripts** are in `Data_Access/Transcriptomics/Engaging-Openmind_Transfer/{Tsai,DeJager}/`.

### Synapse — DeJager FASTQs Only

The DeJager FASTQs are hosted on Synapse (project `syn21438684`). See [FASTQ Acquisition](preprocessing/fastq-acquisition.md) for setup and download instructions.

## Phenotype Data

Clinical phenotype CSVs are tracked in git at `Data/Phenotypes/` and are available immediately after cloning:

| File | Description | Rows |
|------|-------------|------|
| `ROSMAP_clinical.csv` | Core clinical variables (sex, education, APOE, cogdx, Braak, CERAD, PMI) | 3,584 |
| `dataset_652_basic_12-23-2021.csv` | Comprehensive phenotype extract (cognitive, neuropath, biomarkers, lifestyle) | 3,681 |
| `TSAI_DEJAGER_all_patients_wACEscores.csv` | Tsai + DeJager patients with ACE scores | 296 |
| `DeJager_ID_Map.csv` | Maps `projid` to `individualID` for DeJager samples | ~20 |
