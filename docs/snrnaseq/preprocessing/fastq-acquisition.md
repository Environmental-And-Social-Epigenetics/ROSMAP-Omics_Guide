# FASTQ Acquisition

The first preprocessing step is acquiring raw FASTQ files. The two datasets use entirely different acquisition methods: the DeJager dataset is downloaded from Synapse, while the Tsai dataset is discovered on the MIT Engaging cluster and transferred to Openmind via Globus.

!!! tip "Skip FASTQ acquisition"
    If you want to start from CellRanger or CellBender outputs instead of raw FASTQs, see [Data Access](../data-access.md) for download instructions. FASTQs can also be downloaded from the NAS backup as an alternative to Synapse or Globus.

## Overview

=== "DeJager"

    The DeJager ROSMAP snRNA-seq FASTQs are hosted on Synapse (project syn21438684). A Python script downloads files using the Synapse client, organizing them by library ID. This requires a Synapse account with access to the project and a personal access token.

=== "Tsai"

    The Tsai FASTQs are scattered across multiple locations on the MIT Engaging cluster (`/nfs/picower*` filesystems). A discovery pipeline scans these locations, builds a master CSV index, and then transfers the files to Openmind via Globus for Cell Ranger processing.

## DeJager: Synapse Download

### Prerequisites

1. Create an account at [synapse.org](https://www.synapse.org).
2. Request access to project syn21438684.
3. Generate a personal access token from your Synapse profile settings.

### Authentication

```bash
source config/paths.sh
init_conda
conda activate "${SYNAPSE_ENV}"
synapse login --auth-token <YOUR_TOKEN>
```

Alternatively, set the `SYNAPSE_AUTH_TOKEN` environment variable in your shell or SLURM script.

### Running the Download

The download script reads a CSV mapping Synapse IDs to library IDs and downloads each FASTQ to the appropriate directory:

```bash
cd Preprocessing/DeJager/01_FASTQ_Download
sbatch Download_FASTQs.sh
```

The input CSV (`Synapse_FASTQ_IDs.csv`) is located at `${DATA_ROOT}/Data/DeJager/FASTQs_Download/FASTQ_Download_CSVs/` (as configured in `config/paths.sh`).

### Handling Failed Downloads

Network timeouts are common when downloading several terabytes. The rerun script retries failed downloads:

```bash
sbatch Download_FASTQs_Rerun.sh
```

### Output Structure

```
${DEJAGER_FASTQS}/
+-- 190403-B4-A/
|   +-- 190403-B4-A_Broad_S1_L001_R1_001.fastq.gz
|   +-- 190403-B4-A_Broad_S1_L001_R2_001.fastq.gz
|   +-- ...
+-- 190403-B4-B/
+-- ...
```

### Resource Requirements

| Parameter | Value |
|-----------|-------|
| Cores | 64 |
| Memory | 512 GB |
| Time | 47 hours |

!!! warning "Storage Requirements"
    The full DeJager FASTQ dataset is several terabytes. Verify that your scratch filesystem has sufficient space before starting the download. Monitor usage with `du -sh ${DEJAGER_FASTQS}`.

### Scripts

| Script | Description |
|--------|-------------|
| `Download_FASTQs.py` | Main download script using Synapse Python client |
| `Download_FASTQs.sh` | SLURM batch wrapper |
| `Download_FASTQs_Rerun.py` | Retry script for failed downloads |
| `Download_FASTQs_Rerun.sh` | SLURM wrapper for retry |

## Tsai: Engaging Cluster Discovery and Transfer

The Tsai acquisition pipeline has three sub-steps: discovering FASTQs on Engaging, organizing them, and transferring to Openmind.

### Step 1: Build the Master CSV

A parallel discovery script scans the Engaging filesystem to find all ROSMAP FASTQs and creates a comprehensive index:

```bash
cd Preprocessing/Tsai/01_FASTQ_Location/01_Build_Master_CSV
sbatch run_build_csv.sh
```

This produces `All_ROSMAP_FASTQs.csv`, which indexes all 5,197 FASTQ files across 480 patients and 492 unique library IDs.

### Step 2: Organize FASTQs (Optional)

Create a structured symlink hierarchy on Engaging for local use:

```bash
cd 02_Organize_FASTQs
sbatch run_organize_fastqs.sh
```

This step is optional and only needed if you plan to run Cell Ranger directly on Engaging.

### Step 3: Transfer to Openmind via Globus

Generate the Globus transfer batch file, submit the transfer, and verify:

```bash
cd 03_Globus_Transfer

# Generate the batch file mapping source to destination paths
python Scripts/generate_globus_batch.py

# Submit the Globus transfer
./Scripts/submit_globus_transfer.sh

# After transfer completes, verify on Openmind
python Scripts/verify_transfer.py
```

### Globus Endpoints

| Cluster | Endpoint ID | Name |
|---------|-------------|------|
| Engaging | `ec54b570-cac5-47f7-b2a1-100c2078686f` | MIT ORCD Engaging Collection |
| Openmind | `cbc6f8da-d37e-11eb-bde9-5111456017d9` | mithpc#openmind |

### Output Structure on Openmind

```
${TSAI_FASTQS}/
+-- <projid>/
    +-- <Library_ID>/
        +-- [run_id/]           # For multi-source samples
        |   +-- *.fastq.gz
        +-- *.fastq.gz          # For single-source samples
```

### Dataset Summary

| Metric | Value |
|--------|-------|
| Total FASTQ files | 5,197 |
| Unique patients (projids) | 480 |
| Unique Library IDs | 492 |
| Total data size | ~9 TB |
| Multi-source samples | 272 (sequenced on multiple flowcells) |

### Prerequisites

- GNU `parallel` for FASTQ discovery
- Python 3 with `pandas`
- Globus CLI (install via conda; see [Environment Setup](../setup.md))
- Access to `/nfs/picower*` filesystems on Engaging

## Data Path Configuration

All FASTQ-related paths are configured in `config/paths.sh`:

| Variable | Description |
|----------|-------------|
| `TSAI_FASTQS` | Tsai FASTQ storage directory (`Data/Transcriptomics/Tsai/FASTQs/`) |
| `DEJAGER_FASTQS_DIR` | DeJager FASTQ storage directory (`Data/Transcriptomics/DeJager/FASTQs/`) |
| `TSAI_FASTQS_CSV` | Path to `All_ROSMAP_FASTQs.csv` master index |

Verify your configuration with:

```bash
source config/paths.sh
check_paths
```
