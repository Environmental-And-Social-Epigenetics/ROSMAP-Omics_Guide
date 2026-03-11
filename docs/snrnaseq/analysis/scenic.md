# SCENIC

SCENIC (Single-Cell rEgulatory Network Inference and Clustering) reconstructs gene regulatory networks from single-cell expression data. It identifies transcription factors, their target genes, and the co-regulated gene modules (regulons) that are active in each cell type.

## Method

SCENIC operates in three stages:

1. **GRNBoost2**: Infers co-expression modules by identifying genes whose expression correlates with each transcription factor across cells. This produces candidate regulatory links.
2. **RcisTarget**: Filters the candidate links by checking whether the target genes share enriched transcription factor binding motifs in their promoter regions. Only TF-target pairs supported by motif evidence are retained, forming regulons.
3. **AUCell**: Scores each cell for the activity of each regulon using an area-under-the-curve metric. This produces a cells-by-regulons activity matrix.

The output is a set of regulons (TF + target gene sets) and per-cell activity scores, which can then be compared across phenotype groups.

## Running the Analysis

Navigate to the appropriate phenotype and dataset directory:

```bash
cd Analysis/<Phenotype>/SCENIC/<Dataset>/
```

Follow the phenotype-specific README for the exact commands. SCENIC is typically run through pySCENIC, the Python implementation.

## Inputs

| File | Source |
|------|--------|
| Annotated AnnData (`.h5ad`) | `Processing/<Dataset>/Pipeline/` output from Stage 3 |
| TF motif rankings database | Downloaded separately (species-specific, from the SCENIC resources page) |
| TF motif annotation database | Downloaded separately |

The motif databases are large files (several GB) that must be downloaded once and referenced in the SCENIC configuration.

## Outputs

| File | Description |
|------|-------------|
| Regulon list | Each regulon: a transcription factor and its validated target genes |
| AUCell matrix | Per-cell activity scores for each regulon |
| Regulon specificity scores | Which regulons are preferentially active in which cell types |

## Resource Requirements

| Parameter | Value |
|-----------|-------|
| Cores | 32+ |
| Memory | 256 GB+ |
| Time | 24 to 48 hours |

!!! warning "Resource requirements"
    SCENIC is the most computationally demanding analysis in the pipeline. The GRNBoost2 step in particular requires substantial memory and benefits from many cores. Request a dedicated high-memory node for this analysis.
