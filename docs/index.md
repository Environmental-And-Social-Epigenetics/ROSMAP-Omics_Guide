# EASE Multi-Omics Guide

This site provides practical, step-by-step guides for analyzing multi-omics data from the Religious Orders Study and Memory and Aging Project (ROSMAP) dataset. Each guide walks through an entire analysis pipeline, from raw data acquisition to final results.

The guides are maintained by the Environmental and Social Epigenetics (EASE) laboratory and are intended for researchers who want to reproduce or extend these analyses on an HPC cluster.

## Guides

### [Single Nucleus RNA Sequencing](snrnaseq/index.md)

Process single-nucleus RNA-seq data from frozen postmortem brain tissue. This guide covers FASTQ alignment with Cell Ranger, ambient RNA removal with CellBender, quality control filtering, doublet removal, batch-corrected integration with Harmony, cell type annotation, and downstream differential expression and regulatory network analyses.

### [Lipidomics](lipidomics/index.md)

Analyze brain lipidomics data to test associations between social isolation scores and lipid abundance. This guide covers data normalization, quality control, per-lipid regression models, sex-stratified ANCOVA interaction analyses, sensitivity analyses excluding Alzheimer's disease pathology cases, and publication-quality visualization.

### [MRI](mri/index.md)

Process structural, functional, and diffusion MRI data through containerized neuroimaging pipelines. This guide covers BIDS dataset construction, quality control with MRIQC, fMRI processing with fMRIPrep and XCP-D, anatomical segmentation with FreeSurfer, diffusion processing with QSIPrep, and statistical analysis of brain measures.

---

**EASE Laboratory**

- Ravikiran Raju, MD, PhD (Principal Investigator, Boston Children's Hospital)
- Nina Khera (Harvard University)
- Mahmoud Abdelmoneum (Massachusetts Institute of Technology)

[Source code on GitHub](https://github.com/Environmental-And-Social-Epigenetics){ .md-button }
