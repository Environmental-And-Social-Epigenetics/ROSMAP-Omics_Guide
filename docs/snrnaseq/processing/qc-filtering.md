# QC Filtering (Stage 1)

Stage 1 applies quality control filtering to each sample individually, removing low-quality cells (damaged, dying, or empty droplets that survived CellBender) based on percentile-based thresholds. For the DeJager dataset, this stage also assigns patient IDs to cell barcodes using the Demuxlet output.

## Script and Environment

| Component | Value |
|-----------|-------|
| Script | `01_qc_filter.py` |
| SLURM wrapper | `01_qc_filter.sh` |
| Conda environment | `QC_ENV` (spec: `envs/stage1_qc.yml`) |
| Job type | SLURM array (one task per sample) |

## Input

The input is the CellBender-filtered count matrix for each sample:

=== "Tsai"

    ```
    ${TSAI_CELLBENDER}/{projid}/cellbender_output_filtered.h5
    ```

=== "DeJager"

    ```
    ${DEJAGER_CELLBENDER}/{library}/processed_feature_bc_matrix_filtered.h5
    ```

## QC Metrics

The script computes the following quality metrics for each cell using scanpy:

| Metric | Description |
|--------|-------------|
| `total_counts` (nCount) | Total number of UMIs in the cell |
| `n_genes_by_counts` (nFeature) | Number of genes with at least one count |
| `pct_counts_mt` | Percentage of counts from mitochondrial genes (prefix `MT-`) |
| `pct_counts_ribo` | Percentage of counts from ribosomal genes (prefix `RPS` or `RPL`) |
| `pct_counts_hb` | Percentage of counts from hemoglobin genes (prefix `HB`, excluding `HBP`) |
| `pct_counts_in_top_20_genes` | Percentage of total counts from the top 20 most-expressed genes |

## Filtering Thresholds

Cells are removed if they fall outside these thresholds:

| Metric | Lower Bound | Upper Bound | Rationale |
|--------|------------|-------------|-----------|
| `log1p_total_counts` | 4.5th percentile | 96th percentile | Removes cells with abnormally low or high total RNA counts |
| `log1p_n_genes_by_counts` | 5th percentile | None | Removes cells expressing too few genes (likely empty or damaged) |
| `pct_counts_mt` | None | 10% | Removes cells with high mitochondrial content, indicating cell damage or death |

### Why Percentile-Based Thresholds?

Fixed count thresholds (e.g., "remove cells with fewer than 200 genes") do not generalize well across samples with different sequencing depths and cell compositions. Percentile-based thresholds adapt to each sample's distribution, applying consistent stringency regardless of sequencing depth.

The log1p transformation (`log(1 + x)`) stabilizes the variance of count and gene number distributions, which are typically right-skewed. This makes the percentile thresholds more robust.

### Why a 10% Hard Cap on Mitochondrial Percentage?

High mitochondrial content is a well-established marker of cell stress and damage in single-cell RNA-seq. In postmortem brain tissue, some degree of mitochondrial RNA is expected, but cells exceeding 10% are very likely damaged. The 10% threshold is standard in the field for brain tissue snRNA-seq and provides a reliable upper bound that is less sensitive to sample-specific variation than a purely MAD-based approach.

## DeJager-Specific: Patient ID Assignment

For the DeJager dataset, Stage 1 also assigns patient IDs to cell barcodes before QC filtering:

- **Multiplexed libraries:** Cell barcodes are mapped to patient IDs using `cell_to_patient_assignmentsFinal0.csv`, the aggregated Demuxlet output.
- **"Alone" libraries** (suffix `-alone`): The R-number is extracted from the library name and mapped to a patient ID via `patient_id_overrides.json`.
- **Unmapped cells:** Cells without a patient ID mapping are dropped.

This assignment step ensures that each cell in the output h5ad file has a known patient identity.

## Running Stage 1

### Full Dataset (SLURM)

```bash
cd Processing/Tsai/Pipeline
./submit_pipeline.sh 1
```

Or submit manually:

```bash
sbatch 01_qc_filter.sh
```

### Subset (Local Testing)

```bash
python 01_qc_filter.py --sample-ids 10100574,10100862
```

### List Available Samples

```bash
python 01_qc_filter.py --list-samples
```

## Output

### Per-Sample Files

Each sample produces a QC-filtered AnnData file:

=== "Tsai"

    ```
    ${TSAI_QC_FILTERED}/{projid}_qc.h5ad
    ```

=== "DeJager"

    ```
    ${DEJAGER_QC_FILTERED}/{library}_qc.h5ad
    ```

### QC Summary

The script produces `qc_summary.csv` in the output directory, which tracks per-sample cell counts before and after filtering. Each array task appends its row, and the file accumulates across all tasks. This provides a quick overview of filtering impact:

| Column | Description |
|--------|-------------|
| `sample_id` | Patient or library identifier |
| `cells_before` | Number of cells in the CellBender input |
| `cells_after` | Number of cells after QC filtering |
| `pct_retained` | Percentage of cells retained |
| `median_genes` | Median genes per cell (post-filter) |
| `median_counts` | Median UMI counts per cell (post-filter) |
| `median_pct_mt` | Median mitochondrial percentage (post-filter) |

## Expected Results

The fraction of cells removed varies by sample quality, but typical expectations are:

- **Cells retained:** 80-95% of input cells pass QC filtering in well-prepared samples.
- **Low-quality samples:** Samples with extensive cell damage (high mitochondrial fraction) may retain as few as 50-70% of cells.
- **Mitochondrial filter:** This is typically the most impactful single filter, especially in postmortem brain tissue.

## SLURM Resources

| Parameter | Value |
|-----------|-------|
| Cores | 4 |
| Memory | 32 GB |
| Time | 12 hours |
| Array | Tsai: `1-476%32`, DeJager: `1-200%32` |
