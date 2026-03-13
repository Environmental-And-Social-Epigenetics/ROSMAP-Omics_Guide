# Integration and Annotation (Stage 3)

Stage 3 is the most computationally intensive step. It loads all singlet-filtered samples, concatenates them into a single AnnData object, performs normalization, highly variable gene selection, dimensionality reduction, batch correction, clustering, and cell type annotation. The result is a fully integrated and annotated dataset ready for downstream biological analysis.

## Script and Environment

| Component | Value |
|-----------|-------|
| Script | `03_integration_annotation.py` |
| SLURM wrapper | `03_integration_annotation.sh` |
| Conda environment | `BATCHCORR_ENV` (spec: `envs/stage3_integration.yml`) |
| Job type | Single SLURM job (not an array) |

## Processing Steps

The script executes the following steps in order. Each step, its scanpy/harmonypy function call, and key parameters are listed below.

### Full Parameter Table

| Step | Function | Key Parameters |
|------|----------|----------------|
| 1. Load and concatenate | `anndata.concat` | All `*_singlets.h5ad` files from Stage 2 |
| 2. Normalization | `sc.pp.normalize_total` | Per-cell median-based normalization (target sum = median total counts) |
| 3. Log transform | `sc.pp.log1p` | Natural log(1 + x) |
| 4. Store raw counts | `adata.layers["counts"]` | Raw counts preserved for HVG selection and downstream DE |
| 5. HVG selection | `sc.pp.highly_variable_genes` | `flavor="seurat_v3"`, `n_top_genes=3000`, `layer="counts"` |
| 6. PCA | `sc.tl.pca` | `n_comps=30`, `svd_solver="arpack"`, `use_highly_variable=True` |
| 7. Batch correction | `harmonypy.run_harmony` | See [Harmony Batch Correction](#harmony-batch-correction) below |
| 8. Neighbor graph | `sc.pp.neighbors` | `n_neighbors=30`, `n_pcs=30`, `metric="cosine"`, `use_rep="X_harmony"` |
| 9. Clustering | `sc.tl.leiden` | Resolutions: `0.2`, `0.5`, `1.0` |
| 10. UMAP | `sc.tl.umap` | `min_dist=0.15`, `random_state=0` |
| 11. Cell type annotation | `decoupler.run_ora` | Mohammadi 2020 PFC markers; `use_raw=False` |

### Step Details

#### Normalization and Log Transform

Each cell's counts are normalized to the median total count across all cells, then log-transformed. This corrects for differences in sequencing depth between cells and stabilizes variance for downstream steps.

The raw count matrix is preserved in `adata.layers["counts"]` for two purposes: the Seurat v3 HVG selection method operates on raw counts, and downstream pseudobulk differential expression analysis requires unnormalized counts.

#### Highly Variable Gene (HVG) Selection

The Seurat v3 method (`flavor="seurat_v3"`) is used to select the top 3,000 highly variable genes. This method models the mean-variance relationship from raw counts and selects genes with the highest residual variance, which is more robust than methods that operate on normalized data.

Only these 3,000 genes are used for PCA and subsequent dimensionality reduction steps. All genes are retained in the AnnData object for other analyses.

#### PCA

Principal Component Analysis reduces the gene expression space to 30 components using the ARPACK solver (efficient for sparse matrices). Only highly variable genes contribute to the PCA computation. The resulting embedding (`X_pca`) is the input for batch correction.

#### Harmony Batch Correction

Harmony iteratively adjusts the PCA embedding so that cells from different sequencing batches overlap in reduced-dimensional space, removing technical variation while preserving biological signal.

=== "Tsai"

    | Parameter | Value | Description |
    |-----------|-------|-------------|
    | Input representation | `X_pca` | 30-component PCA embedding |
    | Batch variable | `derived_batch` | Flowcell-based batch groups (~41 groups) |
    | Theta | 2.0 | Correction strength (default) |
    | Output | `X_harmony` | Corrected embedding stored in `obsm` |

    The `derived_batch` variable is produced by `derive_batches.py`, which extracts Illumina flowcell IDs from FASTQ headers and groups samples that share flowcells. This yields approximately 41 natural batch groups for the Tsai dataset, each reflecting actual technical variation (shared reagent lots, instruments, and handling conditions).

=== "DeJager"

    | Parameter | Value | Description |
    |-----------|-------|-------------|
    | Input representation | `X_pca` | 30-component PCA embedding |
    | Batch variable | `patient_id` | Patient identifier |
    | Theta | 2.0 | Correction strength (default) |
    | Output | `X_harmony` | Corrected embedding stored in `obsm` |

##### Why Not Use `projid` (Patient ID) for Tsai?

Using patient ID as the batch variable (as done historically) is too aggressive for the Tsai dataset. With 480 levels, Harmony forces cells from all patients to overlap in PC space, removing genuine inter-individual biological variation. This inter-individual variation is precisely what pseudobulk differential expression analysis needs to detect condition-associated differences. The flowcell-based `derived_batch` variable (~41 levels) corrects for actual technical batch effects without erasing biological signal.

##### Why Not Use the Computational Batch (1-16)?

The `batch` column in `patient_metadata.csv` (values 1-16) represents computational convenience groupings for SLURM job scheduling, not real sequencing batches. These groups do not correspond to shared technical conditions and are not appropriate for batch correction.

##### Command-Line Configuration

The batch variable and correction behavior are configurable:

```bash
# Default: flowcell-based batches
python 03_integration_annotation.py

# Use a different batch key
python 03_integration_annotation.py --harmony-batch-key projid

# Adjust correction strength
python 03_integration_annotation.py --harmony-theta 1.0

# Skip batch correction entirely
python 03_integration_annotation.py --skip-harmony
```

When `--skip-harmony` is used, the neighbor graph is built on `X_pca` instead of `X_harmony`.

#### Neighbor Graph and Clustering

The neighbor graph is built using cosine distance on the Harmony-corrected embedding (30 PCs, 30 neighbors). Leiden clustering is then performed at three resolutions:

| Resolution | Column Name | Purpose |
|------------|-------------|---------|
| 0.2 | `leiden_res0_2` | Coarse clusters (major cell types) |
| 0.5 | `leiden_res0_5` | **Primary clustering** (used for annotation) |
| 1.0 | `leiden_res1_0` | Fine clusters (subtypes and states) |

Multiple resolutions are computed to allow flexible downstream analysis. The 0.5 resolution is used by default for cell type annotation.

#### UMAP

UMAP is computed from the neighbor graph for visualization purposes. The `min_dist=0.15` parameter provides moderate separation between clusters, and `random_state=0` ensures reproducibility.

UMAP coordinates are stored in `obsm["X_umap"]` and are used for visualization only, not for clustering or other analytical computations.

#### Cell Type Annotation via ORA

Cell types are assigned to Leiden clusters (at resolution 0.5) using over-representation analysis (ORA) with the `decoupler` library. The reference marker set is from Mohammadi et al. (2020), which provides cell type-specific marker genes for the human prefrontal cortex.

The annotation process:

1. For each Leiden cluster, run ORA against the Mohammadi 2020 marker gene sets.
2. Rank cell types by enrichment score for each cluster.
3. Assign the top-ranked cell type to each cluster.
4. Map cluster-level annotations back to individual cells via `obs["cell_type"]`.

Two CSV files document the annotation:

| File | Description |
|------|-------------|
| `cluster_annotation_rankings.csv` | Full ORA rankings for all clusters and cell types |
| `cluster_annotation_top3.csv` | Top 3 cell type candidates per cluster with scores |

Expected cell types in the DLPFC include:

- Excitatory neurons (multiple subtypes)
- Inhibitory neurons
- Astrocytes
- Microglia
- Oligodendrocytes
- Oligodendrocyte precursor cells (OPCs)
- Endothelial cells
- Pericytes

## Input

All singlet-filtered h5ad files from Stage 2:

=== "Tsai"

    ```
    ${TSAI_DOUBLET_REMOVED}/*_singlets.h5ad
    ```

=== "DeJager"

    ```
    ${DEJAGER_DOUBLET_REMOVED}/*_singlets.h5ad
    ```

## Output

=== "Tsai"

    ```
    ${TSAI_INTEGRATED}/
    +-- tsai_integrated.h5ad              # After Harmony, before annotation
    +-- tsai_annotated.h5ad               # Final annotated object
    +-- cluster_annotation_rankings.csv   # Full ORA results
    +-- cluster_annotation_top3.csv       # Top 3 cell types per cluster
    +-- figures/                           # UMAP visualizations
    ```

=== "DeJager"

    ```
    ${DEJAGER_INTEGRATED}/
    +-- dejager_integrated.h5ad
    +-- dejager_annotated.h5ad
    +-- cluster_annotation_rankings.csv
    +-- cluster_annotation_top3.csv
    +-- figures/
    ```

### What the Final AnnData Object Contains

The `*_annotated.h5ad` file is the primary deliverable of the entire pipeline:

| Component | Location | Description |
|-----------|----------|-------------|
| Normalized expression | `X` | Log-normalized gene expression |
| Raw counts | `layers["counts"]` | Unnormalized counts for DE analysis |
| Cell metadata | `obs` | `projid`, `cell_type`, `leiden_res0_5`, `derived_batch`, etc. |
| Gene metadata | `var` | Gene symbols, highly_variable flag |
| PCA | `obsm["X_pca"]` | 30-component PCA embedding |
| Harmony embedding | `obsm["X_harmony"]` | Batch-corrected PCA |
| UMAP | `obsm["X_umap"]` | 2D UMAP coordinates |
| Neighbor graph | `obsp` | Connectivities and distances |

## SLURM Resources

| Parameter | Value |
|-----------|-------|
| Cores | 32 |
| Memory | 500 GB |
| Time | 48 hours |
| Partition | `lhtsai` (or equivalent high-memory partition) |

!!! warning "Memory Requirements"
    Stage 3 loads the entire dataset into memory. For the Tsai dataset (476 samples, potentially millions of cells), this requires approximately 500 GB of RAM. This is the most resource-intensive step in the pipeline. Ensure your cluster partition supports this allocation before submitting.

## Troubleshooting

**Out-of-memory during concatenation.** If the job is killed during sample loading, consider processing a subset first to estimate memory requirements, then request additional memory.

**Harmony does not converge.** Try adjusting `--harmony-theta` (lower values apply less correction). Verify that `derived_batch` has a reasonable number of levels (too many levels, e.g., one per sample, can cause convergence issues).

**Poor cell type annotation.** Check `cluster_annotation_top3.csv` to see whether the top candidates are close in score. If annotation is ambiguous, the cluster may represent a mixed or transitional cell state. Consider using a finer clustering resolution (1.0 instead of 0.5) to resolve subtypes.

**Leiden clustering produces too many or too few clusters.** The three resolutions (0.2, 0.5, 1.0) are provided for flexibility. If none produces the desired granularity, rerun with additional resolutions by modifying the script.
