# Doublet Removal (Stage 2)

Stage 2 detects and removes computational doublets from each QC-filtered sample. Doublets are droplets that captured two or more nuclei, producing a combined expression profile that does not represent any real cell. These artificial observations can distort clustering, inflate apparent cell type heterogeneity, and introduce false signals in differential expression analysis.

## Script and Environment

| Component | Value |
|-----------|-------|
| Script | `02_doublet_removal.Rscript` |
| SLURM wrapper | `02_doublet_removal.sh` |
| Conda environment | `SINGLECELL_ENV` (spec: `envs/stage2_doublets.yml`) |
| Job type | SLURM array (one task per sample) |

## Method: scDblFinder

The pipeline uses **scDblFinder**, a Bioconductor package that detects doublets by simulating artificial doublets from the observed data and training a classifier to distinguish real cells from these simulated doublets.

scDblFinder was chosen over alternatives (DoubletFinder, Scrublet) because:

- It is fast, scaling well to large datasets.
- It integrates naturally with the Bioconductor/SingleCellExperiment ecosystem.
- Benchmarks show competitive or superior performance across diverse single-cell datasets.

### Key Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `seed` | 123 | Random seed for reproducibility |
| `BPPARAM` | `MulticoreParam(4)` | 4 parallel workers via BiocParallel |

## Why R for This Stage?

Stage 2 is the only pipeline stage written in R (rather than Python). This is because scDblFinder is a Bioconductor package without a mature Python equivalent. The script reads h5ad files via `zellkonverter` (which bridges AnnData and SingleCellExperiment formats) and writes the filtered result back to h5ad.

### R Environment Bootstrapping

If the active R environment is missing a required Bioconductor package (such as `GenomeInfoDbData`), the script automatically bootstraps it into a local library directory (`Processing/{Dataset}/Pipeline/R_libs/`) on first run. This avoids requiring system-wide R package installations.

## Input

QC-filtered AnnData files from Stage 1:

=== "Tsai"

    ```
    ${TSAI_QC_FILTERED}/{projid}_qc.h5ad
    ```

=== "DeJager"

    ```
    ${DEJAGER_QC_FILTERED}/{library}_qc.h5ad
    ```

## Running Stage 2

### Full Dataset (SLURM)

```bash
cd Processing/Tsai/Pipeline
./submit_pipeline.sh 2
```

Or as part of the full pipeline:

```bash
./submit_pipeline.sh all
```

### Subset (Local Testing)

```bash
Rscript 02_doublet_removal.Rscript --sample-ids 10100574,10100862
```

## Output

### Per-Sample Files

Each sample produces a singlet-only AnnData file:

=== "Tsai"

    ```
    ${TSAI_DOUBLET_REMOVED}/{projid}_singlets.h5ad
    ```

=== "DeJager"

    ```
    ${DEJAGER_DOUBLET_REMOVED}/{library}_singlets.h5ad
    ```

### Doublet Summary

The script produces `doublet_summary.csv` in the output directory, accumulating across all array tasks:

| Column | Description |
|--------|-------------|
| `sample_id` | Patient or library identifier |
| `cells_input` | Number of cells from Stage 1 |
| `singlets` | Number of cells classified as singlets |
| `doublets` | Number of cells classified as doublets |
| `doublet_rate` | Fraction of cells classified as doublets |

## Expected Doublet Rates

The expected doublet rate depends on the number of nuclei loaded into the 10x Chromium device. General expectations from 10x Genomics documentation:

| Nuclei Loaded | Expected Doublet Rate |
|--------------|----------------------|
| ~1,000 | ~0.8% |
| ~5,000 | ~4.0% |
| ~10,000 | ~8.0% |
| ~16,000 | ~12.8% |

For the ROSMAP dataset, with typical loading of ~5,000-10,000 nuclei per sample, doublet rates of 3-10% are normal. Substantially higher rates may indicate a problem with the library preparation or loading.

scDblFinder's doublet rate estimates are generally well-calibrated; they should roughly match the expected rate given the loading density.

## Processing Flow

```mermaid
graph TD
    A["{sample}_qc.h5ad<br>(Stage 1 output)"] --> B[Read h5ad via zellkonverter]
    B --> C[Convert to SingleCellExperiment]
    C --> D[Run scDblFinder<br>seed=123, 4 workers]
    D --> E{Doublet classification}
    E -->|Singlet| F[Retain cell]
    E -->|Doublet| G[Remove cell]
    F --> H[Write {sample}_singlets.h5ad]
    D --> I[Append to doublet_summary.csv]
```

## SLURM Resources

| Parameter | Value |
|-----------|-------|
| Cores | 4 |
| Memory | 32 GB |
| Time | 12 hours |
| Array | Tsai: `1-476%32`, DeJager: `1-200%32` |

## Troubleshooting

**R package errors on first run.** The script attempts to bootstrap missing Bioconductor packages automatically. If this fails (e.g., due to network issues on compute nodes), manually install the missing package in the `SINGLECELL_ENV` conda environment:

```bash
conda activate "${SINGLECELL_ENV}"
R -e 'BiocManager::install("GenomeInfoDbData")'
```

**zellkonverter conversion failures.** Ensure that the h5ad file from Stage 1 is valid and not corrupted. Re-run Stage 1 for the affected sample if needed.

**Unexpectedly high doublet rates (>15%).** This may indicate a problem with the sample rather than the software. Check the Cell Ranger web summary for that sample to verify loading metrics and overall library quality.
