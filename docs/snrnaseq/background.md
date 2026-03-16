# Scientific Background

This page provides the scientific context behind the ROSMAP snRNA-seq pipeline, explaining the dataset, the tissue, the sequencing technology, and the rationale for each major processing decision. Readers unfamiliar with single-cell genomics or the ROSMAP cohort should start here.

## What is ROSMAP?

ROSMAP refers to two complementary longitudinal cohort studies of aging and Alzheimer's disease (AD) conducted by Rush University Medical Center:

- **Religious Orders Study (ROS)**, started in 1994, enrolling older Catholic nuns, priests, and brothers across the United States.
- **Memory and Aging Project (MAP)**, started in 1997, enrolling older adults from the greater Chicago area.

Both studies share a common design. Participants undergo annual cognitive and clinical evaluations throughout their lifetime, and all have agreed to organ donation at the time of death. This provides matched antemortem clinical data (cognitive trajectories, clinical diagnoses, lifestyle and environmental exposures) with postmortem brain tissue, making ROSMAP one of the most deeply phenotyped brain tissue repositories available for AD research.

The combination of longitudinal clinical assessment with postmortem molecular profiling enables questions that neither alone could answer: which molecular changes in specific brain cell types are associated with cognitive decline, resilience to pathology, or environmental risk factors such as adverse childhood experiences and social isolation?

## Why the Dorsolateral Prefrontal Cortex (DLPFC)?

This pipeline processes tissue from the dorsolateral prefrontal cortex (DLPFC), corresponding roughly to Brodmann areas 9 and 46. The DLPFC is one of the most commonly profiled brain regions in AD research for several reasons:

- **Cognitive relevance.** The DLPFC is involved in higher-order cognitive functions, including working memory and executive function, which decline during AD progression.
- **Broad disease spectrum.** Unlike the entorhinal cortex or hippocampus (which show pathology earliest and most severely), the DLPFC captures a wider range of disease states within a single cohort, from cognitively normal to late-stage dementia.
- **Practical availability.** The DLPFC is routinely dissected in ROSMAP autopsies and has been profiled across multiple omics modalities, facilitating cross-study integration.

## Why Single-Nucleus RNA Sequencing?

Traditional single-cell RNA sequencing (scRNA-seq) requires fresh, enzymatically dissociated tissue. This is impractical for postmortem brain samples, which are collected at autopsy and stored frozen, sometimes for years.

Single-nucleus RNA sequencing (snRNA-seq) solves this by isolating individual nuclei from frozen tissue, enabling transcriptomic profiling at single-cell resolution from archived biobank samples. This is critical for ROSMAP, where all tissue is postmortem.

snRNA-seq captures the transcriptional state of each cell type in the brain (neurons, astrocytes, microglia, oligodendrocytes, oligodendrocyte precursor cells, endothelial cells, and others), enabling:

- **Cell type-specific differential expression.** Comparing gene expression between conditions (e.g., AD vs. control) within individual cell types, rather than in bulk tissue where signal is averaged across all cell types.
- **Identification of disease-associated cell states.** Detecting subpopulations within a cell type that expand or contract in disease.
- **Cell type proportion analysis.** Characterizing shifts in cellular composition across disease conditions.

### Why `--include-introns true`?

Nuclear RNA has a different composition than cytoplasmic RNA. Because snRNA-seq captures RNA from the nucleus before most splicing is complete, a substantial fraction of reads map to intronic regions rather than exons. The Cell Ranger flag `--include-introns true` counts these intronic reads, significantly increasing the number of detected genes and UMIs per nucleus. Without this flag, a large portion of genuinely informative reads would be discarded, reducing statistical power for all downstream analyses.

## Why CellBender for Ambient RNA Removal?

During nucleus isolation, some RNA leaks out of damaged or lysed cells into the surrounding solution. This ambient RNA is then captured in droplets alongside intact nuclei, contaminating every cell's expression profile with a background signal that reflects the overall tissue composition rather than the individual cell's identity.

CellBender uses a deep generative model to distinguish true cell-associated RNA from this ambient background. Without ambient RNA correction:

- Cell type markers "bleed" across cell types. For example, neuronal markers appear in non-neuronal cells, obscuring genuine cell type boundaries.
- Differential expression results can be confounded by ambient contamination levels rather than true biological differences.
- Clustering and cell type annotation become less accurate, with increased misclassification.

The pipeline uses `--fpr 0` (or `0.01`) for a stringent false positive rate, aggressively removing ambient signal at the cost of potentially removing a small amount of true signal. This tradeoff is appropriate for downstream analyses that depend on clean cell type separation.

## Why Percentile-Based QC Filtering?

After ambient RNA removal, individual cells must be filtered to remove low-quality observations: damaged cells, empty droplets that passed CellBender, and other technical artifacts. The pipeline uses percentile-based thresholds for this filtering, which are more robust than standard deviation or MAD-based approaches for the long-tailed distributions common in single-cell data.

### Why percentiles instead of MAD?

Percentile thresholds provide stable cutoffs regardless of distribution shape, avoiding the assumption of approximate normality that MAD-based methods require. In single-cell data, QC metric distributions often have heavy tails (from doublets, debris, or highly variable cell types), which can distort MAD estimates. Fixed percentile thresholds are simpler to interpret and reproduce across datasets.

### What is filtered?

Cells with QC metrics falling beyond defined thresholds are removed:

| Metric | Threshold | Rationale |
|--------|-----------|-----------|
| `log1p_total_counts` | Below 4.5th or above 96th percentile | Removes cells with abnormally high or low total RNA counts |
| `log1p_n_genes_by_counts` | Below 5th percentile | Removes cells expressing too few genes (likely empty or damaged) |
| `pct_counts_mt` | Above 10% | Removes cells with high mitochondrial content, a marker of cell damage |

The 4.5th/96th percentile range for total counts is relatively permissive, retaining genuine biological variation while removing extreme outliers at both ends. The 10% hard cap for mitochondrial percentage reflects the consistent association between high mitochondrial content and cell death or damage in postmortem brain tissue.

## Why Harmony for Batch Correction?

Even with identical protocols, technical differences between sequencing runs (reagent lots, flowcells, instruments, handling variation) introduce systematic variation that can confound biological signal. Cells from different sequencing batches may cluster by batch rather than by cell type if this variation is not corrected.

Harmony addresses this by iteratively adjusting the PCA embedding so that cells from different batches overlap in reduced-dimensional space. Several properties make Harmony well-suited for this pipeline:

- **Operates on the PCA embedding only.** The raw count matrix is not modified, preserving original expression values for downstream differential expression analysis.
- **Scalable.** Harmony handles hundreds of thousands of cells efficiently, which is necessary for the combined Tsai dataset (480 patients).
- **Configurable.** The batch variable and correction strength (`theta`) are adjustable via command-line arguments.

### The Batch Variable: `derived_batch`

Choosing the correct batch variable is critical. Using patient ID (`projid`) as the batch variable is too aggressive: with 480 levels, Harmony would force cells from all patients to overlap in PC space, removing genuine inter-individual biological variation needed for pseudobulk differential expression analysis.

The computational batch groupings in `patient_metadata.csv` (values 1-16) are also unsuitable, as these are arbitrary SLURM scheduling groups that do not reflect actual technical variation.

Instead, the pipeline uses `derived_batch`, produced by the `derive_batches.py` script. This script extracts Illumina flowcell IDs from FASTQ headers and groups samples by shared flowcells, yielding approximately 41 natural batch groups for the Tsai dataset. These groups reflect actual technical variation (samples sequenced on the same flowcell share reagent lots, instruments, and handling conditions) and are the appropriate covariate for batch correction.

The batch variable is configurable via the `--harmony-batch-key` argument to `03_integration_annotation.py`, allowing comparison between correction strategies.

## The Two Datasets

### DeJager Dataset

- **Source.** Downloaded from Synapse (project syn21438684).
- **Library design.** Libraries are multiplexed: each sequencing library contains cells from multiple patients pooled together before sequencing.
- **Extra preprocessing step.** Genotype-based demultiplexing (Demuxlet) using whole-genome sequencing (WGS) data is required to assign each cell barcode to its patient of origin.
- **Preprocessing flow.** Download FASTQs from Synapse, Cell Ranger alignment, CellBender ambient RNA removal, then Demuxlet.

### Tsai Dataset

- **Source.** Located on the MIT Engaging cluster.
- **Library design.** One patient per library; patient assignments are known from sequencing metadata.
- **No demultiplexing needed.** Cell-to-patient assignment comes directly from the library metadata.
- **Scale.** 480 patients, 5,197 FASTQ files (~9 TB), processed in 16 batches of 30 patients to accommodate scratch space limits.
- **Cohorts.** Patients are drawn from three study arms:
    - **ACE** (Adverse Childhood Experiences)
    - **Resilient** (Cognitive Resilience, maintained cognitive function despite AD pathology)
    - **SocIsl** (Social Isolation)

Both datasets are processed through the same core pipeline (Cell Ranger, CellBender, QC filtering, doublet removal, integration, annotation) to ensure methodological comparability.

## What the Pipeline Produces

The final output is an annotated AnnData object (e.g., `tsai_annotated.h5ad`) containing:

- Single-cell gene expression data for all patients, filtered and quality-controlled
- Cell type annotations assigned via over-representation analysis (ORA) using Mohammadi 2020 brain cell type markers for the prefrontal cortex
- Batch-corrected embeddings (Harmony-adjusted PCA and UMAP coordinates)
- Patient and batch metadata for each cell

This object is the starting point for downstream analysis:

- **Differential expression (DEG).** Compare gene expression between conditions within specific cell types using NEBULA mixed models (ACE phenotype) or pseudobulk methods such as DESeq2 (SocIsl phenotype).
- **SCENIC.** Infer gene regulatory networks and identify active transcription factor regulons per cell type using pySCENIC.
- **Metabolic analysis.** Estimate metabolic flux differences between phenotype groups at the single-cell level using COMPASS.
- **Gene set enrichment.** Map differentially expressed genes to known biological pathways using WebGestaltR.
