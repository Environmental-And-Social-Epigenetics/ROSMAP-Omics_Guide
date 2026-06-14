# Gene Set Enrichment Analysis

Gene Set Enrichment Analysis (GSEA) maps the **full ranked** DEG result to known biological pathways,
identifying which functional categories shift coherently with the phenotype. Unlike a simple
overlap test on the significant-gene list, GSEA uses every gene's effect size and direction, so it can
detect pathways where many genes move modestly in the same direction without any single gene reaching
significance.

!!! info "Implementation status"
    GSEA is implemented for **ACE** (Tsai + DeJager, `Analysis/ACE/GSEA/`) and **SocIsl** (Tsai + DeJager,
    `Analysis/SocIsl/GSEA/`). Only **Resilient** has no GSEA implementation yet.

## Method: Ranked GSEA via WebGestaltR

The pipeline runs **WebGestaltR with `enrichMethod = "GSEA"`** — ranked gene set enrichment, **not**
over-representation analysis (ORA). Each gene is scored by a signed rank statistic, and WebGestaltR tests
whether each pathway's genes are concentrated toward the top or bottom of that ranking.

### Why GSEA over ORA?

ORA takes a thresholded list (e.g. `padj < 0.05`) and asks whether a pathway is overrepresented in it,
treating all "significant" genes equally and discarding everything below the cutoff. Ranked GSEA instead:

- uses **every** tested gene and its **direction** (up/down), so coordinated, sub-threshold shifts across a
  pathway are detectable;
- is **threshold-free**, avoiding the sensitivity of ORA to an arbitrary significance cutoff;
- yields a **normalized enrichment score (NES)** whose sign indicates whether a pathway is up- or
  down-regulated with the phenotype.

### The Rank Statistic

Genes are ranked by `sign(log2FoldChange) × −log10(pvalue)` computed from the upstream DESeq2 result.
This pushes strongly up-regulated genes to the top and strongly down-regulated genes to the bottom, with
the magnitude reflecting confidence. Genes with `NA`/infinite ranks are dropped before enrichment.

### Pathway Databases

The ACE pipeline tests **8** databases (`GSEA_DATABASES` in `gsea_analysis.R`):

| Database | Type |
|----------|------|
| `geneontology_Biological_Process_noRedundant` | GO biological processes (redundancy-reduced) |
| `geneontology_Cellular_Component_noRedundant` | GO cellular compartments (redundancy-reduced) |
| `geneontology_Molecular_Function_noRedundant` | GO molecular functions (redundancy-reduced) |
| `pathway_KEGG` | Metabolic and signaling pathways |
| `pathway_Panther` | Protein-classification pathways |
| `pathway_Reactome` | Curated biological pathways |
| `pathway_Wikipathway` | Community-curated pathways |
| `network_Transcription_Factor_target` | TF–target regulatory network |

The `noRedundant` GO variants collapse near-duplicate terms so a single signal does not appear as dozens of
overlapping hits.

### Key WebGestaltR Parameters

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `enrichMethod` | `"GSEA"` | Ranked enrichment (not ORA) |
| `organism` | `"hsapiens"` | Human gene annotations |
| `interestGeneType` | `"genesymbol"` | Ranked list is keyed by gene symbol |
| `minNum` | `10` | Minimum pathway size to test |
| `sigMethod` / `fdrThr` | `"fdr"` / `0.2` | FDR-based significance at 0.2 |

## Scripts

=== "ACE"

    `Analysis/ACE/GSEA/Tsai/` and `Analysis/ACE/GSEA/DeJager/`:

    | Script | Purpose |
    |--------|---------|
    | `gsea_analysis.R` | Core ranked-GSEA pipeline (reads DESeq2 `.rda`, ranks, runs WebGestaltR over 8 DBs) |
    | `gseaResults.Rscript` | Results assembly |
    | `run_enrichment.sh` | SLURM wrapper |
    | `aceGseaT.sh` / `aceGseaT_male_arms.sh` | Batch launchers (incl. the male AD-confounding arms) |
    | `gsea_visualize.R`, `merge_gsea_summaries.sh` | Plotting and summary merging |

=== "SocIsl"

    `Analysis/SocIsl/GSEA/Tsai/` and `Analysis/SocIsl/GSEA/DeJager/`:

    | Script | Purpose |
    |--------|---------|
    | `tsaiGseaResults.Rscript`, `tsaiGseaResults_v2.Rscript` | Tsai GSEA pipeline (v2 adds databases) |
    | `gseaResults.Rscript` | DeJager GSEA pipeline |
    | `gsea.sh` | SLURM wrapper |

## Inputs

| File | Source |
|------|--------|
| DESeq2 result objects | `deseqAnalysisACE_{phenotype}_{celltype}_{sex}.rda` from [DEG analysis](deg.md) |

GSEA consumes the **full** DESeq2 result (all tested genes with their `log2FoldChange`/`pvalue`), not just
the significant subset.

## Outputs

Results are produced per sex, cell type, and pathway database. Naming differs by cohort:

| File Pattern | Cohort |
|-------------|--------|
| `tsaiPlotGsea{Sex}{Pathway}.csv`, `{celltype}_{db}.rds`, `{celltype}_ranked_genes.csv` | ACE Tsai |
| `plotGsea{Sex}{Pathway}.csv` | SocIsl / DeJager |

Each enrichment table contains, per pathway:

| Column | Description |
|--------|-------------|
| `geneSet` | Pathway/term ID |
| `description` | Human-readable term name |
| `size` | Number of genes in the pathway |
| `NES` | Normalized enrichment score (sign = direction; magnitude = strength) |
| `FDR` | False discovery rate for the term |

A positive `NES` means the pathway is enriched among genes **up**-regulated with the phenotype; a negative
`NES` means enriched among **down**-regulated genes.

## Environment

Uses `gsea_analysis` (spec: `envs/analysis/gsea/environment.yml`):

- R >= 4.2
- WebGestaltR
- clusterProfiler
- bioconductor-org.hs.eg.db (human gene annotations)
- ggplot2, dplyr

## How to Run

```bash
source config/paths.sh

# ACE (Tsai)
cd Analysis/ACE/GSEA/Tsai/
sbatch run_enrichment.sh          # or: bash aceGseaT.sh to launch all arms

# SocIsl (Tsai)
cd Analysis/SocIsl/GSEA/Tsai/
sbatch gsea.sh
```

## Resource Requirements

Resources differ by cohort submit script:

| Cohort | Cores | Memory | Time |
|--------|-------|--------|------|
| ACE Tsai (`run_enrichment.sh` / `aceGseaT.sh`) | 8 | 64 GB | up to 12 hours |
| ACE DeJager (`run_enrichment.sh`) | 45 | 100 GB | 5 hours |

!!! note "GSEA quality depends on upstream DESeq2"
    Because the rank list is built from DESeq2 `log2FoldChange`/`pvalue`, GSEA results are only as good as
    the DEG step that produced them. Cell types with too few patients (skipped by `aceDegT.Rscript`) will
    have no `.rda` to enrich, and genes with `NA`/infinite ranks are excluded before enrichment.
