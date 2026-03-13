# Environment Setup

This page walks through the complete first-time setup of the ROSMAP snRNA-seq pipeline on a SLURM cluster, from repository clone to a verified, ready-to-run configuration.

## Prerequisites

Before beginning, ensure you have access to:

- A SLURM cluster with GPU nodes (required for CellBender)
- `conda` or `mamba` (via Miniconda, Miniforge, or Anaconda)
- Git for cloning the repository
- Singularity (for the DeJager Demuxlet step only)
- Globus CLI (for Tsai FASTQ transfer only)

## Step 1: Clone the Repository

```bash
cd /your/workspace
git clone <repo-url> ROSMAP-SingleNucleusRNAseq
```

After cloning, the `Data/` directory inside the repo contains only small tracked files (phenotype CSVs). Large data (FASTQs, CellRanger output, CellBender output, processing outputs) is populated into `Data/Transcriptomics/` using scripts in `Data_Access/`. See [Data Access](data-access.md) for details on what to download based on your starting point.

## Step 2: Configure Paths

All paths in the pipeline are centralized in `config/paths.sh`. This file is the single source of truth: every shell wrapper sources it rather than hard-coding paths.

For per-user customization without modifying the tracked file, create `config/paths.local.sh` (which is gitignored and automatically sourced at the end of `paths.sh` if it exists):

```bash
# config/paths.local.sh
export CONDA_INIT_SCRIPT="$HOME/miniforge3/etc/profile.d/conda.sh"
export CONDA_ENV_BASE="$HOME/conda_envs"
export SCRATCH_ROOT="/scratch/$USER"
export DATA_ROOT="/data/project/mylab"
export CELLRANGER_PATH="$HOME/apps/cellranger-8.0.0"
export CELLRANGER_REF="$HOME/references/refdata-gex-GRCh38-2020-A"
export SLURM_MAIL_USER="you@university.edu"
export SLURM_PARTITION="your_partition"
```

### Key Variables

At minimum, set the following variables:

| Variable | Description |
|----------|-------------|
| `CONDA_INIT_SCRIPT` | Path to your conda/mamba initialization script |
| `CONDA_ENV_BASE` | Base directory where conda environments will be created |
| `DATA_ROOT` | Permanent storage location for processed data |
| `SCRATCH_ROOT` | Temporary scratch filesystem for intermediate outputs |
| `CELLRANGER_PATH` | Directory containing the `cellranger` executable |
| `CELLRANGER_REF` | Path to the Cell Ranger reference transcriptome directory |
| `SLURM_MAIL_USER` | Email address for SLURM job notifications |
| `SLURM_PARTITION` | Default SLURM partition (leave empty for cluster default) |

### Verify Configuration

```bash
source config/paths.sh
check_paths
```

The `check_paths` function validates that all configured directories and files exist and reports any missing dependencies.

## Step 3: Install Cell Ranger

Cell Ranger v8.0.0 is required for read alignment. It is not distributed through conda and must be downloaded directly from 10x Genomics:

1. Register at [10x Genomics Support](https://www.10xgenomics.com/support/software/cell-ranger).
2. Download Cell Ranger v8.0.0 and extract the archive.
3. Set `CELLRANGER_PATH` in your configuration to the extracted directory.

```bash
export PATH=${CELLRANGER_PATH}:$PATH
cellranger --version
```

## Step 4: Download the Reference Genome

The pipeline uses the 10x Genomics human reference `refdata-gex-GRCh38-2020-A`:

```bash
curl -O https://cf.10xgenomics.com/supp/cell-exp/refdata-gex-GRCh38-2020-A.tar.gz
tar -xzf refdata-gex-GRCh38-2020-A.tar.gz
```

Set `CELLRANGER_REF` in your configuration to the path of the extracted directory.

## Step 5: Create Conda Environments

The processing pipeline requires three separate conda environments, one per stage. An automated installer handles all three:

```bash
source config/paths.sh
bash setup/install_envs.sh
```

This creates the following environments:

| Environment | YAML Spec | Config Variable | Purpose |
|-------------|-----------|-----------------|---------|
| `qcEnv` | `envs/stage1_qc.yml` | `QC_ENV` | Stage 1: QC filtering (scanpy, anndata) |
| `single_cell_BP` | `envs/stage2_doublets.yml` | `SINGLECELL_ENV` | Stage 2: Doublet removal (R, scDblFinder, BiocParallel) |
| `BatchCorrection_SingleCell` | `envs/stage3_integration.yml` | `BATCHCORR_ENV` | Stage 3: Integration (scanpy, harmonypy, decoupler) |

The YAML specs are located in `Processing/Tsai/Pipeline/envs/`. To create environments manually instead of using the installer:

```bash
conda env create -f Processing/Tsai/Pipeline/envs/stage1_qc.yml \
    -p $CONDA_ENV_BASE/qcEnv
```

### CellBender Environment

CellBender (used in preprocessing) requires GPU/CUDA support and is installed separately:

```bash
conda create -n cellbender_env python=3.10
conda activate cellbender_env
pip install cellbender
```

Set `CELLBENDER_ENV` in your configuration to the environment path.

### Activating Environments

After sourcing `paths.sh`, activate environments using the configured variables:

```bash
source config/paths.sh
init_conda
conda activate "${QC_ENV}"          # Stage 1
conda activate "${SINGLECELL_ENV}"  # Stage 2
conda activate "${BATCHCORR_ENV}"   # Stage 3
```

## Step 6: Synapse Credentials (DeJager Only)

If you need to run the DeJager preprocessing pipeline, configure Synapse access:

1. Create an account at [synapse.org](https://www.synapse.org).
2. Request access to project syn21438684.
3. Generate a personal access token.
4. Install the Synapse client and authenticate:

```bash
pip install synapseclient
synapse login --auth-token <YOUR_TOKEN>
```

Alternatively, set the `SYNAPSE_AUTH_TOKEN` environment variable.

## Step 7: Singularity (DeJager Demuxlet Only)

The Demuxlet step uses a Singularity container providing the popscle toolkit:

```bash
module load openmind/singularity/3.10.4
singularity build Demuxafy.sif docker://drneavin/demuxafy:latest
```

Set `DEMUXAFY_SIF` in your configuration to the path of the built container.

## Step 8: Verify the Complete Setup

```bash
source config/paths.sh
check_paths
```

For a more thorough check, run the preflight script:

```bash
bash config/preflight.sh
```

This validates that all pipeline prerequisites (conda environments, input files, references) are in place.

After verifying your setup, see [Data Access and Entry Points](data-access.md) to populate the `Data/` directory for your chosen starting point.

To confirm the processing pipeline can discover samples:

```bash
cd Processing/Tsai/Pipeline
python 01_qc_filter.py --list-samples | head -5
```

## SLURM Partition Recommendations

The pipeline uses several partitions configured for the MIT Openmind cluster. Override these by setting `SLURM_PARTITION` in your configuration if your cluster uses different partition names.

| Partition | Used By | Purpose |
|-----------|---------|---------|
| `mit_preemptable` | Cell Ranger | Lower-priority partition with shorter queue times; jobs may be preempted and automatically requeued |
| `mit_normal_gpu` | CellBender | GPU partition for ambient RNA removal |
| `lhtsai` | Stage 3 Integration | Lab-specific high-memory partition (500 GB RAM required) |
| `mit_normal` | Demuxlet | Standard partition for high-memory CPU jobs |

## Expected Directory Layout

After setup is complete, the workspace should have this structure:

```
/your/workspace/
+-- ROSMAP-SingleNucleusRNAseq/              # The repository
    +-- config/paths.sh
    +-- Preprocessing/
    +-- Processing/
    +-- Analysis/
    +-- Data/                                # Canonical data location
    |   +-- Phenotypes/                      # Git-tracked (available immediately)
    |   +-- Transcriptomics/
    |       +-- Tsai/
    |       |   +-- FASTQs/                  # Populated via Data_Access/ scripts
    |       |   +-- Cellranger_Output/
    |       |   +-- Cellbender_Output/
    |       |   +-- Processing_Outputs/
    |       +-- DeJager/
    |           +-- FASTQs/
    |           +-- Cellranger_Output/
    |           +-- Cellbender_Output/
    |           +-- Processing_Outputs/
    +-- Data_Access/                         # Transfer scripts for populating Data/
        +-- Transcriptomics/
            +-- Tsai_Server/                 # NAS (SFTP) transfers
            +-- Engaging-Openmind_Transfer/  # Globus transfers
```

The `Data/` directory inside the repository is the canonical location for all pipeline data. Phenotype CSVs are tracked in git and available immediately after cloning. All transcriptomics data (FASTQs, count matrices, processing outputs) must be populated using the scripts in `Data_Access/` — see [Data Access](data-access.md) for instructions.
