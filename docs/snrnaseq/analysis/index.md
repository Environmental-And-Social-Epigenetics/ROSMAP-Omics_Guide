# Analysis Overview

After processing produces annotated AnnData objects with cell type labels, the pipeline enters the analysis phase. This phase applies complementary approaches to identify genes, metabolic pathways, transcription factors, and regulatory networks associated with phenotypes of interest.

## Analysis Types

| Type | Method | Purpose | Implemented For |
|------|--------|---------|-----------------|
| [DEG](deg.md) | NEBULA (ACE), DESeq2/limma (SocIsl) | Identify differentially expressed genes between phenotype groups within each cell type | ACE/Tsai, SocIsl |
| [SCENIC](scenic.md) | pySCENIC | Reconstruct single-cell gene regulatory networks to identify active regulons per cell type | SocIsl |
| [TF / Metabolic](tf-analysis.md) | COMPASS | Estimate metabolic flux and test for pathway differences between phenotype groups | SocIsl |
| [GSEA](gsea.md) | WebGestaltR | Gene set enrichment analysis across KEGG, Reactome, GO, and other pathway databases | SocIsl |

DEG analysis identifies which genes change expression. SCENIC discovers co-regulated gene modules and the transcription factors that govern them. COMPASS characterizes metabolic pathway activity differences. GSEA maps DEG results to known biological pathways.

## Phenotype Status

| Phenotype | Description | DEG | SCENIC | TF/Metabolic | GSEA |
|-----------|-------------|-----|--------|--------------|------|
| **ACE** | Adverse Childhood Experiences | NEBULA (Tsai) | — | — | — |
| **SocIsl** | Social Isolation | DESeq2 (Tsai, DeJager) | pySCENIC (Tsai, DeJager) | COMPASS (Tsai, DeJager) | WebGestaltR (Tsai, DeJager) |
| **Resilient** | Cognitive Resilience | — | — | — | — |

Resilient and ACE (for non-DEG analyses) have directory structures in place but no analysis scripts yet. See [Adding a New Phenotype](#adding-a-new-phenotype) for how to populate them.

## Directory Structure

Analyses are organized first by phenotype, then by analysis type, then by dataset:

```
Analysis/
├── envs/                   # Conda environment YAML specs
│   ├── deg.yml             # DESeq2, edgeR, limma
│   ├── scenic.yml          # pySCENIC, loompy
│   ├── compass.yml         # COMPASS (requires IBM CPLEX)
│   └── gsea.yml            # WebGestaltR, clusterProfiler
├── _template/              # Copy to start a new phenotype
│   ├── DEG/
│   │   ├── DeJager/
│   │   └── Tsai/
│   ├── TF/
│   │   ├── DeJager/
│   │   └── Tsai/
│   └── SCENIC/
│       ├── DeJager/
│       └── Tsai/
├── ACE/                    # Adverse Childhood Experiences
│   ├── DEG/
│   │   ├── DeJager/        # placeholder
│   │   └── Tsai/           # ← NEBULA pipeline (6 scripts, 816 jobs)
│   │       └── scripts/
│   ├── TF/                 # placeholder
│   └── SCENIC/             # placeholder
├── SocIsl/                 # Social Isolation
│   ├── _data_prep/         # Legacy data preparation scripts
│   ├── DEG/                # DESeq2/limma (Tsai + DeJager)
│   ├── TF/                 # COMPASS metabolic analysis (Tsai + DeJager)
│   ├── SCENIC/             # pySCENIC (Tsai + DeJager)
│   └── GSEA/               # WebGestaltR (Tsai + DeJager)
└── Resilient/              # Cognitive Resilience (placeholder only)
    ├── DEG/
    ├── TF/
    └── SCENIC/
```

## Analysis Environments

The analysis conda environments are installed via `setup/install_envs.sh --analysis`:

| Environment | YAML Spec | Key Packages |
|-------------|-----------|--------------|
| `deg_analysis` | `Analysis/envs/deg.yml` | DESeq2, edgeR, limma, scanpy |
| `scenic_analysis` | `Analysis/envs/scenic.yml` | pySCENIC, loompy (~3.5 GB motif databases required separately) |
| `compass_analysis` | `Analysis/envs/compass.yml` | COMPASS (requires IBM CPLEX academic license) |
| `gsea_analysis` | `Analysis/envs/gsea.yml` | WebGestaltR, clusterProfiler |

!!! warning "NEBULA environment"
    The ACE DEG pipeline uses a separate `nebulaAnalysis7` environment (referenced as `NEBULA_ENV` in `config/paths.sh`) that is **not** created by `install_envs.sh --analysis`. See [Differential Expression](deg.md#ace-deg-pipeline-nebula) for setup instructions.

## Data Requirements

All analysis types require:

- Annotated AnnData objects from Stage 3 of the processing pipeline, with `cell_type` and `projid` columns in `obs`. These can be downloaded from the NAS without running the pipeline — see [Data Access](../data-access.md) (Entry Point D).
- Clinical phenotype data from the `Data/Phenotypes/` directory (referenced via `${PHENOTYPE_DIR}` in `config/paths.sh`). These CSVs are tracked in git and available immediately after cloning.

## Adding a New Phenotype

To analyze a new phenotype, copy the template directory and populate it with analysis scripts:

```bash
cp -r Analysis/_template/ Analysis/NewPhenotype/
```

Edit `Analysis/NewPhenotype/README.md` to define the phenotype, patient selection criteria, and any phenotype-specific covariates. Then add analysis scripts under the appropriate `DEG/`, `TF/`, `SCENIC/`, or `GSEA/` subdirectories.

The ACE/DEG/Tsai NEBULA pipeline is the most structured example to follow. See [Differential Expression](deg.md) for a detailed walkthrough of its scripts and workflow.

## Resource Requirements

| Analysis | Cores | Memory | Time | Notes |
|----------|-------|--------|------|-------|
| DEG preprocessing (NEBULA) | 8 | 200 GB | 12 hours | Single job, run once |
| DEG per job (NEBULA) | 45 | 100 GB | up to 5 hours | 816 jobs total for ACE |
| DEG (DESeq2, per cell type) | 8 | 64 GB | 1 to 2 hours | SocIsl |
| SCENIC | 32+ | 256 GB+ | 24 to 48 hours | Per cell type |
| COMPASS | 40 | 600 GB | 24 hours | Per cell type per sex; requires IBM CPLEX |
| GSEA | 4 | 16 GB | 1 to 2 hours | Per analysis |
