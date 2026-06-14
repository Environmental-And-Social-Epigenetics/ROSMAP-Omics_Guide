# QC Filtering (Stage 1)

Stage 1 applies quality control filtering to each sample individually, removing low-quality cells (damaged, dying, or empty droplets that survived CellBender) based on percentile-based thresholds. For the DeJager dataset, this stage also assigns patient IDs to cell barcodes using the Demuxlet output.

## Script and Environment

| Component | Value |
|-----------|-------|
| Script | `01_qc_filter.py` |
| SLURM wrapper | `01_qc_filter.sh` |
| Conda environment | `QC_ENV` (spec: `envs/processing/stage1_qc/environment.yml`) |
| Job type | SLURM array (one task per sample) |

## Input

The input is the CellBender-filtered count matrix for each sample:

=== "Tsai"

    ```
    ${TSAI_CELLBENDER}/{projid}/processed_feature_bc_matrix_filtered.h5
    ```

=== "DeJager"

    ```
    ${DEJAGER_CELLBENDER}/{library}/processed_feature_bc_matrix_filtered.h5
    ```

!!! warning "Tsai filename reconciliation required"
    The Tsai integrated CellBender step writes `cellbender_output_filtered.h5`, but this script reads `processed_feature_bc_matrix_filtered.h5`. Apply the symlink workaround from [CellBender → Reconciling the Tsai filename for Stage 1](../preprocessing/cellbender.md#reconciling-the-tsai-filename-for-stage-1) before running, or Stage 1 will report no samples.

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

The same percentile *values* (4.5 / 96 / 5) are used for every sample, but the actual cutoffs are recomputed from each sample's own distribution. A shallowly sequenced sample and a deeply sequenced one therefore get different absolute count thresholds while receiving the same *relative* stringency — which is the point of percentile filtering.

### Why an Upper Bound on Total Counts?

The 96th-percentile cap on `log1p_total_counts` targets the opposite failure mode from the lower bound: cells (really droplets) with abnormally high total counts are usually **doublets or multiplets** — two or more nuclei captured together — or dense debris aggregates that survived CellBender. These inflate counts and gene numbers and would otherwise masquerade as a distinct high-expression "cell type." Removing the extreme upper tail here is a cheap first pass; the dedicated [doublet-removal](doublet-removal.md) stage that follows catches the doublets that fall *within* the normal count range.

### Why a 10% Hard Cap on Mitochondrial Percentage?

High mitochondrial content is a well-established marker of cell stress and damage in single-cell RNA-seq. In postmortem brain tissue, some degree of mitochondrial RNA is expected, but cells exceeding 10% are very likely damaged. The 10% threshold is standard in the field for brain tissue snRNA-seq and provides a reliable upper bound that is less sensitive to sample-specific variation than a purely MAD-based approach.

## DeJager-Specific: Patient ID Assignment

For the DeJager dataset, Stage 1 also assigns patient IDs to cell barcodes before QC filtering:

- **Multiplexed libraries:** Cell barcodes are mapped to patient IDs using `cell_to_patient_assignmentsFinal0.csv`, the aggregated Demuxlet output.
- **"Alone" libraries** (suffix `-alone`): The R-number is extracted from the library name and mapped to a patient ID via `patient_id_overrides.json`.
- **Unmapped cells:** Cells without a patient ID mapping are dropped.

This assignment step ensures that each cell in the output h5ad file has a known patient identity.

!!! warning "DeJager: unmapped cells are dropped silently"
    Cells whose barcodes have no Demuxlet patient assignment are removed **before** the QC filters run, so they never appear in `n_cells_before`. If a library's Demuxlet output is missing or incomplete, this can quietly discard most of its cells. Cross-check `cells_after` against the Cell Ranger / CellBender cell estimate for each library, and confirm `cell_to_patient_assignmentsFinal0.csv` covers every library before launching the array.

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
| `n_cells_before` | Number of cells in the CellBender input |
| `n_cells_after` | Number of cells after QC filtering |
| `n_removed` | Cells removed (`n_cells_before − n_cells_after`) |
| `pct_retained` | Percentage of cells retained |
| `n_outlier` | Cells flagged by the count/gene percentile filters |
| `n_mt_outlier` | Cells flagged by the mitochondrial-percentage filter |
| `median_genes` | Median genes per cell (pre-filter, all cells) |
| `median_counts` | Median UMI counts per cell (pre-filter, all cells) |
| `median_pct_mt` | Median mitochondrial percentage (pre-filter, all cells) |

(Header order matches `append_qc_summary()` in `01_qc_filter.py`. The `n_outlier` and `n_mt_outlier` counts can overlap — a cell can be both a count/gene outlier and an MT outlier — so they do not sum to `n_removed`.)

A few rows of a representative `qc_summary.csv` look like:

```csv
sample_id,n_cells_before,n_cells_after,n_removed,pct_retained,n_outlier,n_mt_outlier,median_genes,median_counts,median_pct_mt
10100574,8421,7765,656,92.2,512,201,3010,5180,1.84
10100862,6190,5402,788,87.3,447,398,2740,4615,2.91
20151388,4903,2987,1916,60.9,388,1620,2120,3380,6.72
```

The third row is a low-quality sample: a high `n_mt_outlier` drives most of its attrition, and its `median_pct_mt` is correspondingly elevated — exactly the pattern flagged under [Expected Results](#expected-results) below.

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
| Array | Tsai: `1-478%32`, DeJager: `1-200%32` |
