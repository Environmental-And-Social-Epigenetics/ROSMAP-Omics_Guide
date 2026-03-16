# Gene Set Enrichment Analysis

Gene Set Enrichment Analysis (GSEA) maps differentially expressed genes to known biological pathways, identifying which functional categories are overrepresented among significant DEG results. This contextualizes individual gene-level findings within broader biological processes.

!!! info "Implementation status"
    GSEA is implemented for the **SocIsl (Social Isolation)** phenotype with scripts and results for both Tsai and DeJager datasets. ACE and Resilient phenotypes do not yet have GSEA implementations.

## Method: WebGestaltR Over-Representation Analysis

The pipeline uses WebGestaltR to perform Over-Representation Analysis (ORA). ORA tests whether genes from the DEG results are significantly overrepresented in curated pathway gene sets compared to a background gene list.

### Pathway Databases

The analysis tests enrichment across multiple pathway databases:

| Database | Type |
|----------|------|
| KEGG | Metabolic and signaling pathways |
| Reactome | Curated biological pathways |
| Panther | Protein classification pathways |
| Wikipathway | Community-curated pathways |
| GO Biological Process (BP) | Gene Ontology biological processes |
| GO Cellular Component (CC) | Gene Ontology cellular compartments |
| GO Molecular Function (MF) | Gene Ontology molecular functions |

## Scripts

Scripts are in `Analysis/SocIsl/GSEA/Tsai/` and `Analysis/SocIsl/GSEA/DeJager/`:

| Script | Purpose |
|--------|---------|
| `tsaiGseaResults.Rscript` | Primary GSEA pipeline (Tsai) |
| `tsaiGseaResults_v2.Rscript` | Updated version with additional pathway databases |
| `gsea.sh` | SLURM wrapper |
| `gseaResults.Rscript` | GSEA pipeline (DeJager) |

## Inputs

| File | Source |
|------|--------|
| DEG gene lists | Output from [DEG analysis](deg.md) (significant genes per cell type) |
| Background gene list | All tested genes from the DEG analysis |

## Outputs

Results are saved per sex and pathway database:

| File Pattern | Description |
|-------------|-------------|
| `plotGsea{Sex}{Pathway}.csv` | Enrichment results for each pathway database |

Each results file contains enriched pathway terms with p-values, FDR-adjusted p-values, enrichment ratios, and member gene lists.

## Environment

Uses `gsea_analysis` (from `Analysis/envs/gsea.yml`):

- R >= 4.2
- WebGestaltR
- clusterProfiler
- bioconductor-org.hs.eg.db (human gene annotations)
- ggplot2, dplyr

## How to Run

```bash
source config/paths.sh
cd Analysis/SocIsl/GSEA/Tsai/
sbatch gsea.sh
```

## Resource Requirements

| Parameter | Value |
|-----------|-------|
| Cores | 4 |
| Memory | 16 GB |
| Time | 1 to 2 hours |
