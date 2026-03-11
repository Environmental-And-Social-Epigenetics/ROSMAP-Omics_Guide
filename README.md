# EASE Multi-Omics Guide

Practical guides for analyzing multi-omics data from the ROSMAP (Religious Orders Study and Memory and Aging Project) dataset. Covers single nucleus RNA sequencing, lipidomics, and MRI processing pipelines.

Published at: https://environmental-and-social-epigenetics.github.io/ROSMAP-Omics_Guide/

## Source Repositories

- [ROSMAP-SingleNucleusRNAseq](https://github.com/Environmental-And-Social-Epigenetics/ROSMAP-SingleNucleusRNAseq)
- [ROSMAP-SI-Lipidomics](https://github.com/Environmental-And-Social-Epigenetics/ROSMAP-SI-Lipidomics)
- [ROSMAP-SI-MRI](https://github.com/Environmental-And-Social-Epigenetics/ROSMAP-SI-MRI)

## Local Development

```bash
conda create -n mkdocs-env python=3.11 -y
conda activate mkdocs-env
pip install -r requirements.txt
mkdocs serve
```
