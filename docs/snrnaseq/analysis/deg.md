# Differential Expression Analysis

Differential expression gene (DEG) analysis identifies genes whose expression differs between phenotype groups (for example, high vs. low social isolation) within individual cell types.

## Approach: Pseudobulk

The pipeline uses a pseudobulk approach rather than testing at the single-cell level. For each cell type in each patient, single-cell counts are aggregated (summed) into a single pseudobulk profile. This produces one expression vector per patient per cell type, which is then analyzed with standard bulk RNA-seq differential expression tools (DESeq2 or edgeR).

Pseudobulk is preferred over single-cell-level tests for several reasons:

- It correctly accounts for the fact that cells from the same patient are not independent observations.
- It avoids inflated p-values that arise from treating thousands of correlated cells as independent samples.
- It leverages well-validated bulk RNA-seq statistical frameworks with established FDR control.

## Workflow

1. Load the annotated AnnData object from Stage 3 of processing.
2. Filter to patients that have phenotype data available in the clinical metadata.
3. For each cell type, aggregate raw counts across all cells belonging to each patient.
4. Export pseudobulk count matrices and metadata for DESeq2 or edgeR.
5. Fit the differential expression model with appropriate covariates (age at death, sex, education, post-mortem interval, APOE genotype).
6. Apply FDR correction and report significant genes.

## Running the Analysis

Navigate to the appropriate phenotype and dataset directory:

```bash
cd Analysis/<Phenotype>/DEG/<Dataset>/
```

Follow the phenotype-specific README in that directory for the exact commands and configuration.

## Inputs

| File | Source |
|------|--------|
| Annotated AnnData (`.h5ad`) | `Processing/<Dataset>/Pipeline/` output from Stage 3 |
| Clinical phenotype CSV | `Data/Phenotypes/` |

## Outputs

The analysis produces per-cell-type result tables containing:

| Column | Description |
|--------|-------------|
| gene | Gene symbol |
| log2FoldChange | Effect size (log2 scale) |
| pvalue | Unadjusted p-value |
| padj | FDR-adjusted p-value |

## Resource Requirements

| Parameter | Value |
|-----------|-------|
| Cores | 8 |
| Memory | 64 GB |
| Time | 1 to 2 hours |
