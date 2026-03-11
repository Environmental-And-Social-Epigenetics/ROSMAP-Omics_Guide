# Environment Setup

This page covers the one-time configuration steps required before running any pipeline stage. All runtime settings are centralized in `config.sh`, which is sourced automatically by every submit script.

## Editing config.sh

After cloning the repository, open `config.sh` and set the following variables for your cluster environment.

### Core Paths

```bash
# Root of your BIDS dataset (after Stage 01 construction)
export BIDS_DIR="/path/to/bids_dataset"

# Scratch space for intermediate processing files
export SCRATCH_DIR="/path/to/scratch/${USER}/mri_proc"

# Derivatives output root (default: inside BIDS directory)
export OUTPUT_DIR="${BIDS_DIR}/derivatives"
```

### Container and Cache Paths

```bash
# Directory containing all .img container files
export CONTAINER_DIR="/path/to/container_images"

# Cache directory (for TemplateFlow templates, pip cache, etc.)
export CACHE_DIR="/path/to/.cache"

# TemplateFlow template directory
export TEMPLATEFLOW_DIR="${CACHE_DIR}/templateflow"

# FreeSurfer license file (see below)
export FREESURFER_LICENSE="/path/to/freesurfer/license.txt"
```

### Cluster Modules

These variables specify the module names available on your cluster. Update them to match your system's module naming scheme:

```bash
export APPTAINER_MODULE="openmind8/apptainer/1.1.7"
export MATLAB_MODULE="mit/matlab/2023a"
export FREESURFER_HELPER_MODULE="openmind/freesurfer/6.0.0"
```

### Array Concurrency Limits

These throttle how many SLURM array tasks run simultaneously per stage, preventing cluster overload:

```bash
export MRIQC_ARRAY_CONCURRENCY="80"
export FMRIPREP_ARRAY_CONCURRENCY="120"
export FREESURFER_ARRAY_CONCURRENCY="100"
export FS_TABULATE_ARRAY_CONCURRENCY="120"
export QSIPREP_ARRAY_CONCURRENCY="80"
```

Adjust these values based on your cluster's available resources and fair-share policies.

## Container Images

The pipeline requires six Apptainer/Singularity container images. The expected filenames (derived from version variables in `config.sh`) are:

| Container | Default filename | Source |
|-----------|-----------------|--------|
| MRIQC | `mriqc_22.0.6.img` | [nipreps/mriqc](https://hub.docker.com/r/nipreps/mriqc) |
| fMRIPrep | `fmriprep_23.2.0a3.img` | [nipreps/fmriprep](https://hub.docker.com/r/nipreps/fmriprep) |
| XCP-D | `xcp_d_0.6.0.img` | [pennlinc/xcp_d](https://hub.docker.com/r/pennlinc/xcp_d) |
| FreeSurfer | `freesurfer_7.4.1.img` | [freesurfer/freesurfer](https://hub.docker.com/r/freesurfer/freesurfer) |
| QSIPrep | `qsiprep_0.20.0.img` | [pennbbl/qsiprep](https://hub.docker.com/r/pennbbl/qsiprep) |
| neuromaps | `neuromaps_0.0.5dev.img` | [neuromaps](https://github.com/netneurolab/neuromaps) |

To build a container image from Docker Hub:

```bash
apptainer build mriqc_22.0.6.img docker://nipreps/mriqc:22.0.6
```

Place all images in the directory specified by `CONTAINER_DIR`.

!!! warning "Container file permissions"
    Ensure all `.img` files are readable by your compute nodes. On shared filesystems, this typically means group-readable permissions. Jobs will fail immediately if the container file cannot be opened.

## FreeSurfer License

Stages 03 (fMRIPrep), 04 (FreeSurfer), and 05 (QSIPrep, if FreeSurfer-based reconstruction is used) require a valid FreeSurfer license file. This is a plain text file, not a binary license key.

To obtain a license:

1. Register at [https://surfer.nmr.mgh.harvard.edu/registration.html](https://surfer.nmr.mgh.harvard.edu/registration.html).
2. A `license.txt` file will be emailed to you.
3. Place the file at the path specified by `FREESURFER_LICENSE` in `config.sh`.

The file typically contains three or four lines of text. Verify its contents are not empty:

```bash
cat "$FREESURFER_LICENSE"
```

## TemplateFlow Setup

fMRIPrep and MRIQC use TemplateFlow to retrieve standard-space templates (e.g., MNI152NLin2009cAsym). On clusters without internet access on compute nodes, you must pre-download templates:

```bash
pip install templateflow
python -c "from templateflow import api; api.get('MNI152NLin2009cAsym')"
```

Set `TEMPLATEFLOW_DIR` to the directory containing the downloaded templates. The submit scripts bind-mount this directory into the container at runtime.

## SLURM Cluster Requirements

The pipeline assumes:

- **SLURM scheduler** with `sbatch` and job arrays supported.
- **Apptainer or Singularity** available as a loadable module or in the default `PATH`.
- **Shared filesystem** accessible from both login and compute nodes (for BIDS data, containers, and outputs).
- **Sufficient scratch space** for per-subject intermediate files. fMRIPrep and QSIPrep can each produce 5 to 20 GB of temporary files per subject.

Minimum per-subject resource requirements are documented in each stage page. The most demanding stages are:

- fMRIPrep: 2 days wall time, 16 GB memory, 4 CPUs
- FreeSurfer recon-all: 2 days wall time, 20 GB memory, 8 CPUs
- QSIPrep: 2 days wall time, 16 GB memory, 8 CPUs

## Optional: MRtrix Environment (Stage 07)

Stage 07 (T1/T2 ratio postprocessing) uses `tcksample` from MRtrix3, which is not included in the container images. Install MRtrix via conda:

```bash
conda create -p /path/to/conda_envs/MRtrix -c mrtrix3 mrtrix3
```

Then set the following in `config.sh`:

```bash
export CONDA_SH_PATH="/path/to/anaconda/etc/profile.d/conda.sh"
export MRTRIX_CONDA_ENV="/path/to/conda_envs/MRtrix"
```

## Optional: Globus CLI (Stage 08)

Stage 08 uses the Globus CLI to transfer BIDS data to a remote cluster. Install and authenticate:

```bash
pip install globus-cli
globus login
```

Configure endpoint IDs in `config.sh`:

```bash
export TRANSFER_SOURCE_ENDPOINT="your-source-endpoint-uuid"
export TRANSFER_DEST_ENDPOINT="your-destination-endpoint-uuid"
export TRANSFER_SOURCE_DIR="${BIDS_DIR}"
export TRANSFER_DEST_DIR="/remote/path/to/destination"
```

You can find endpoint UUIDs via the [Globus web interface](https://app.globus.org/) or by running `globus endpoint search`.

## Verification Checklist

Before submitting any jobs, confirm:

- [ ] `config.sh` has no remaining `/path/to/` placeholder values
- [ ] All container images exist at `CONTAINER_DIR` and are readable
- [ ] `FREESURFER_LICENSE` points to a non-empty license file
- [ ] `TEMPLATEFLOW_DIR` contains downloaded templates (or compute nodes have internet access)
- [ ] `BIDS_DIR` contains at least one `sub-*/ses-*/` directory structure
- [ ] The Apptainer/Singularity module loads successfully: `module load ${APPTAINER_MODULE}`
- [ ] SLURM is available: `sbatch --version`
