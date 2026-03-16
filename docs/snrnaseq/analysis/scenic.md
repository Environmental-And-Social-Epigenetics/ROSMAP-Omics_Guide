# SCENIC

SCENIC (Single-Cell rEgulatory Network Inference and Clustering) reconstructs gene regulatory networks from single-cell expression data. It identifies transcription factors, their target genes, and the co-regulated gene modules (regulons) that are active in each cell type.

!!! info "Implementation status"
    SCENIC is implemented for the **SocIsl (Social Isolation)** phenotype with scripts and results for both Tsai and DeJager datasets. ACE and Resilient phenotypes have placeholder directories only.

## Method

SCENIC operates in three stages:

1. **GRNBoost2**: Infers co-expression modules by identifying genes whose expression correlates with each transcription factor across cells. This produces candidate regulatory links.
2. **RcisTarget**: Filters the candidate links by checking whether the target genes share enriched transcription factor binding motifs in their promoter regions. Only TF-target pairs supported by motif evidence are retained, forming regulons.
3. **AUCell**: Scores each cell for the activity of each regulon using an area-under-the-curve metric. This produces a cells-by-regulons activity matrix.

The output is a set of regulons (TF + target gene sets) and per-cell activity scores, which can then be compared across phenotype groups.

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

Before running SCENIC, cells are aggregated into fixed pools of 30 cells using `fixed_micropool()`. This reduces noise by averaging expression across small groups of same-type cells while retaining single-cell-level granularity. The pooled expression matrix is the input for GRNBoost2.

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
| Phenotype CSV | `Data/Phenotypes/dataset_652_basic_12-23-2021.csv` |
| Motif ranking databases | Downloaded separately (see above) |

### Outputs

Results are saved per cell type and per sex:

| File Pattern | Description |
|-------------|-------------|
| `{sex}_regulonsFULL_Tsai{CellType}.csv` | Regulon list: each regulon contains a TF and its validated target genes |
| `{sex}_auc_mtxFULL_Tsai{CellType}.csv` | Per-cell AUCell activity scores for each regulon |

### Environment

Uses `scenic_analysis` (from `Analysis/envs/scenic.yml`):

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
    SCENIC is the most computationally demanding analysis in the pipeline. The GRNBoost2 step in particular requires substantial memory and benefits from many cores. Request a dedicated high-memory node for this analysis.
