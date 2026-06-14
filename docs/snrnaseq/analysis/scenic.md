# SCENIC

SCENIC (Single-Cell rEgulatory Network Inference and Clustering) reconstructs gene regulatory networks from single-cell expression data. It identifies transcription factors, their target genes, and the co-regulated gene modules (regulons) that are active in each cell type.

!!! info "Implementation status"
    SCENIC is implemented for **ACE** (`Analysis/ACE/SCENIC/`) and **SocIsl** (`Analysis/SocIsl/SCENIC/`),
    each with Tsai and DeJager scripts. Only **Resilient** has no SCENIC implementation.

## Method

SCENIC operates in three stages:

1. **GRNBoost2**: Infers co-expression modules. For each transcription factor it fits a gradient-boosted-tree regression predicting that TF's expression from all other genes, then ranks candidate targets by importance. This tree-based approach captures non-linear, multicollinear relationships that simple pairwise correlation misses. The result is candidate regulatory links.
2. **cisTarget** (via `pyscenic.prune.prune2df`): Filters the candidate links by checking whether the target genes share enriched transcription-factor binding motifs in their promoter regions. Only TF–target pairs supported by motif evidence are retained, forming regulons.
3. **AUCell**: Scores each cell for the activity of each regulon using an area-under-the-curve metric over the cell's ranked expression. This produces a cells-by-regulons activity matrix.

The output is a set of **regulons** and per-cell activity scores, which can then be compared across phenotype groups.

!!! question "What is a regulon?"
    A *regulon* is a transcription factor together with the target genes it directly regulates — here,
    targets that both co-vary with the TF (GRNBoost2) **and** carry the TF's binding motif (cisTarget).
    Because a regulon's genes share a regulatory mechanism, a cell's regulon activity (AUCell score, 0–1)
    summarizes whether that regulatory *program* is engaged — more robust and interpretable than any single
    gene's expression.

## SocIsl Implementation

### Scripts

Scripts are in `Analysis/SocIsl/SCENIC/Tsai/`:

| Script | Purpose |
|--------|---------|
| `tsaiAdataScenic.py` | Primary pySCENIC pipeline with fixed micropooling |
| `tsaiAdataScenic.sh` | SLURM wrapper |
| `newScenic.py` | Alternative implementation |
| `newScenic.sh` | SLURM wrapper for alternative |

### Preprocessing: Micropooling

Before running SCENIC, cells are aggregated into fixed-size pools **within each patient** using
`fixed_micropool()` (it sums raw counts per pool; the pooled matrix is then normalized to CPM before
GRNBoost2). The pool size varies by script and sex: `tsaiAdataScenic.py` uses **100** (male) / **150**
(female), and `newScenic.py` uses **50** (the `fixed_micropool` default argument is 30 but is overridden at
each call site).

!!! question "Why micropool at all?"
    Single-nucleus counts are sparse, and GRNBoost2's tree regressions are noisy on near-binary expression
    vectors. Summing a handful of same-patient, same-cell-type cells per pool raises the per-feature count
    depth — sharpening co-expression signal — while pooling *within patient* avoids mixing donors. The
    pool size trades noise reduction against the number of observations: larger pools denoise more but
    leave fewer pools for the regression, which is why larger pools are used for the more abundant strata.

### Cell Types

| Cell Type | Description |
|-----------|-------------|
| Ast | Astrocytes |
| Exc | Excitatory neurons |
| Inh | Inhibitory neurons |
| Mic | Microglia |
| Oli | Oligodendrocytes |
| OPC | Oligodendrocyte precursor cells |

Each cell type is processed separately, and analyses are sex-stratified (female and male run independently).

### Reference Databases

SCENIC requires motif ranking databases (~3.5 GB total, not included in the repository). Download from the [SCENIC resources page](https://resources.aertslab.org/cistarget/):

| File | Size | Description |
|------|------|-------------|
| `hg38_10kbp_up_10kbp_down_full_tx_v10_clust.genes_vs_motifs.rankings.feather` | ~1.2 GB | Extended promoter motif rankings |
| `hg38_500bp_up_100bp_down_full_tx_v10_clust.genes_vs_motifs.rankings.feather` | ~1.1 GB | Proximal promoter motif rankings |
| `motifs-v10nr_clust-nr.hgnc-m0.001-o0.0.tbl` | ~1.2 GB | Motif-to-TF annotation table |
| `hg.txt` | <1 KB | Human transcription factor gene list |

### Inputs

| File | Source |
|------|--------|
| Cell-type-specific h5ad | Preprocessed from annotated AnnData (e.g., `excAnno.h5ad`) |
| Phenotype CSV | `dataset_652_basic_03-23-2022.csv` (read as a relative path from the working directory) |
| Motif ranking databases | Downloaded separately (see above) |

### Outputs

Results are saved per cell type and per sex:

| File Pattern | Description |
|-------------|-------------|
| `{sex}_regulonsFULL_Tsai{CellType}.csv` | Regulon list: each regulon contains a TF and its validated target genes |
| `{sex}_auc_mtxFULL_Tsai{CellType}.csv` | Per-cell AUCell activity scores for each regulon |

### Environment

Uses the `scenic_analysis` environment (defined as `SCENIC_ANALYSIS_ENV` in `config/paths.sh`; spec:
`envs/analysis/scenic/environment.yml`):

- pySCENIC >= 0.12
- loompy >= 3.0
- scanpy, anndata, pandas, numpy, dask
- arboreto (GRNBoost2), ctxcore

### Resource Requirements

| Parameter | Value |
|-----------|-------|
| Cores | 32+ |
| Memory | 256 GB+ |
| Time | 24 to 48 hours |

!!! warning "Resource requirements"
    SCENIC is the most computationally demanding analysis in the pipeline. The GRNBoost2 step in particular requires substantial memory and benefits from many cores. Request a dedicated high-memory node for this analysis. Each cell type × sex runs as an independent job, so the wall-clock cost is per-stratum, not for the whole dataset at once.

!!! tip "What to expect in the output"
    A typical run yields on the order of **100–500 regulons** per cell type, each containing roughly
    10–200 target genes, with AUCell activity scores in **[0, 1]**. Very few regulons (single digits) usually
    means the motif databases were not found or the wrong species/version was used; tens of thousands of
    rows in the regulon CSV usually means pruning (`prune2df`) did not run. The `{sex}_auc_mtxFULL_*` matrix
    has one row per pooled observation and one column per regulon.
