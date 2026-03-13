# Getting Started

This page covers prerequisites and setup steps that are common across all three analysis guides.

## Cluster Access

All three pipelines are designed to run on a Linux HPC cluster with SLURM job scheduling. You will need:

- An account on your institution's SLURM cluster
- Sufficient storage allocation for intermediate and output files (the snRNA-seq pipeline alone can require several terabytes)
- Access to GPU nodes for CellBender (snRNA-seq) and optionally for other GPU-accelerated tools

## Software

The following tools are used across multiple guides:

| Tool | Used By | Installation |
|------|---------|-------------|
| conda / mamba | All guides | [Miniforge](https://github.com/conda-forge/miniforge) recommended |
| Git | All guides | Usually pre-installed on HPC systems |
| Singularity / Apptainer | MRI, snRNA-seq (Demuxlet) | Typically available as a cluster module |
| Globus CLI | snRNA-seq (Tsai transfer), MRI (data transfer) | Install in a dedicated conda environment |

Each guide specifies additional software requirements in its setup page.

## ROSMAP Data Access

ROSMAP data is distributed through the [AD Knowledge Portal on Synapse](https://www.synapse.org). To access the data:

1. Create an account at [synapse.org](https://www.synapse.org).
2. Complete the required data use agreement for the ROSMAP study.
3. Install the Synapse Python client: `pip install synapseclient`.
4. Authenticate: `synapse login -u <username> -p <password> --rememberMe`.

Not all guides require Synapse access. The Tsai snRNA-seq data is transferred via Globus from the MIT Engaging cluster, and MRI data may already reside on your institution's filesystem.

The snRNA-seq repository includes a `Data_Access/` directory with transfer scripts for multiple methods (NAS/SFTP, Globus, Synapse). Users can download data at any processing stage — from raw FASTQs through the final annotated H5ad — without needing to run the full pipeline. See the [snRNA-seq Data Access](snrnaseq/data-access.md) page for details.

## Repository Structure

Each analysis pipeline lives in its own Git repository. Clone the repository for the analysis you want to run:

```bash
# Single Nucleus RNA-seq
git clone https://github.com/Environmental-And-Social-Epigenetics/ROSMAP-SingleNucleusRNAseq.git

# Lipidomics
git clone https://github.com/Environmental-And-Social-Epigenetics/ROSMAP-SI-Lipidomics.git

# MRI
git clone https://github.com/Environmental-And-Social-Epigenetics/ROSMAP-SI-MRI.git
```

Each repository includes a centralized configuration file (`config/paths.sh` for snRNA-seq, `config.py` for Lipidomics, `config.sh` for MRI) that you edit once for your cluster environment. See the setup page of each guide for details.
