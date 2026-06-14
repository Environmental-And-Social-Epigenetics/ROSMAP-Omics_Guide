# Analysis Overview

After processing produces annotated AnnData objects with cell type labels, the pipeline enters the analysis phase. This phase applies complementary approaches to identify genes, metabolic pathways, transcription factors, and regulatory networks associated with phenotypes of interest.

## Analysis Types

| Type | Method | Purpose | Implemented For |
|------|--------|---------|-----------------|
| [DEG](deg.md) | Pseudobulk → DESeq2 | Identify differentially expressed genes between phenotype groups within each cell type | ACE, SocIsl, Resilient |
| [SCENIC](scenic.md) | pySCENIC | Reconstruct single-cell gene regulatory networks to identify active regulons per cell type | ACE, SocIsl |
| [Metabolic (COMPASS)](tf-analysis.md) | COMPASS | Estimate metabolic flux and test for pathway differences between phenotype groups | ACE, SocIsl |
| [TF activity](tf-analysis.md#tfactivity-dorothea-transcription-factor-activity) | DoRothEA | Infer transcription-factor activity from curated TF–target priors | ACE |
| [GSEA](gsea.md) | WebGestaltR (ranked GSEA) | Ranked gene set enrichment across GO, KEGG, Reactome, and other databases | ACE, SocIsl |

These methods answer complementary questions. **DEG** identifies *which genes* change expression.
**GSEA** places those changes in *pathway* context. **SCENIC** discovers co-regulated gene modules and the
transcription factors that govern them, *de novo* from the data. **COMPASS** characterizes *metabolic*
pathway-activity differences. **TF activity (DoRothEA)** scores transcription-factor activity from curated
prior knowledge — a knowledge-based complement to SCENIC's data-driven networks.

## Phenotype Status

Coverage reflects the scripts actually present in each `Analysis/<phenotype>/<method>/` directory:

| Phenotype | Description | DEG | GSEA | SCENIC | COMPASS | TF activity |
|-----------|-------------|-----|------|--------|---------|-------------|
| **ACE** | Adverse Childhood Experiences | ✓ (Tsai + DeJager) | ✓ | ✓ | ✓ | ✓ (TFActivity) |
| **SocIsl** | Social Isolation | ✓ (Tsai + DeJager) | ✓ | ✓ | ✓ | — |
| **Resilient** | Cognitive Resilience | ✓ (DEG only) | — | — | — | — |

All DEG is **pseudobulk + DESeq2** (the `nebulaAnalysis7` env name is legacy — see
[DEG](deg.md#ace-deg-pipeline)). Resilient has a DEG implementation; its other method directories are
scaffolded but empty. See [Adding a New Phenotype](#adding-a-new-phenotype) to populate them.

## Directory Structure

Analyses are organized first by phenotype, then by analysis type, then by dataset:

Conda environment specs live under the **top-level** `envs/analysis/` directory (not inside `Analysis/`):

```
envs/analysis/                   # Conda environment specs (one dir per env)
├── deg/environment.yml          # DESeq2, edgeR, limma
├── scenic/environment.yml       # pySCENIC, loompy
├── compass/environment.yml      # COMPASS (requires IBM CPLEX)
├── gsea/environment.yml         # WebGestaltR, clusterProfiler
└── nebula/environment.yml       # ACE DEG env (nebulaAnalysis7, legacy name)

Analysis/                        # Analysis code, organized by phenotype → method → dataset
├── _template/                   # Copy to start a new phenotype
│   ├── DEG/{DeJager,Tsai}/
│   ├── COMPASS/{DeJager,Tsai}/
│   └── SCENIC/{DeJager,Tsai}/
├── ACE/                         # Adverse Childhood Experiences (most complete)
│   ├── DEG/{Tsai,DeJager}/      # pseudobulk DESeq2 (+ AD-confounding arms)
│   ├── GSEA/{Tsai,DeJager}/     # ranked GSEA (WebGestaltR)
│   ├── SCENIC/{Tsai,DeJager}/   # pySCENIC
│   ├── COMPASS/{Tsai,DeJager}/  # metabolic flux (COMPASS)
│   └── TFActivity/Tsai/         # TF activity (DoRothEA)
├── SocIsl/                      # Social Isolation
│   ├── _data_prep/              # Legacy data preparation scripts
│   ├── DEG/                     # pseudobulk DESeq2 (Tsai + DeJager)
│   ├── GSEA/                    # ranked GSEA (Tsai + DeJager)
│   ├── SCENIC/                  # pySCENIC (Tsai + DeJager)
│   └── COMPASS/                 # metabolic flux (Tsai + DeJager)
└── Resilient/                   # Cognitive Resilience
    ├── DEG/                     # pseudobulk DESeq2
    ├── COMPASS/                 # scaffolded (empty)
    └── SCENIC/                  # scaffolded (empty)
```

!!! note "`TF/` was renamed to `COMPASS/`"
    Earlier versions of the repo had a `TF/` directory for metabolic analysis. It is now **`COMPASS/`**,
    and transcription-factor activity lives in a separate **`TFActivity/`** directory (ACE). If you see
    `TF/` referenced anywhere, it is stale.

## Analysis Environments

The analysis conda environments are installed via `setup/install_envs.sh --analysis`:

| Environment | YAML Spec | Key Packages |
|-------------|-----------|--------------|
| `deg_analysis` | `envs/analysis/deg/environment.yml` | DESeq2, edgeR, limma, scanpy |
| `scenic_analysis` | `envs/analysis/scenic/environment.yml` | pySCENIC, loompy (~3.5 GB motif databases required separately) |
| `compass_analysis` | `envs/analysis/compass/environment.yml` | COMPASS (requires IBM CPLEX academic license) |
| `gsea_analysis` | `envs/analysis/gsea/environment.yml` | WebGestaltR, clusterProfiler |
| `nebulaAnalysis7` | `envs/analysis/nebula/environment.yml` | ACE DEG (DESeq2, zellkonverter, scran) — legacy name |

!!! note "The `nebulaAnalysis7` environment is created automatically"
    The ACE DEG pipeline uses the `nebulaAnalysis7` environment (referenced as `NEBULA_ENV` in
    `config/paths.sh`). Despite the name, it runs **DESeq2 pseudobulk**, not the NEBULA framework, and it
    **is** created by `install_envs.sh --analysis` (from `envs/analysis/nebula/environment.yml`). No manual
    creation is required. See [Differential Expression](deg.md#ace-deg-pipeline).

## Data Requirements

All analysis types require:

- Annotated AnnData objects from Stage 3 of the processing pipeline, with a `cell_type` column plus the dataset's patient-unit column in `obs` — `projid` for Tsai, `library_id` for DeJager (per `config/datasets.yaml`). These can be downloaded from the NAS without running the pipeline — see [Data Access](../data-access.md) (Entry Point D).
- Clinical phenotype data from the `Data/Phenotypes/` directory (referenced via `${PHENOTYPE_DIR}` in `config/paths.sh`). These CSVs are tracked in git and available immediately after cloning.

## Adding a New Phenotype

To analyze a new phenotype, copy the template directory and populate it with analysis scripts:

```bash
cp -r Analysis/_template/ Analysis/NewPhenotype/
```

Edit `Analysis/NewPhenotype/README.md` to define the phenotype, patient selection criteria, and any phenotype-specific covariates. Then add analysis scripts under the appropriate `DEG/`, `COMPASS/`, `SCENIC/`, or `GSEA/` subdirectories.

The `ACE/` phenotype is the most complete worked example; its `DEG/Tsai/` pseudobulk DESeq2 pipeline is the most structured. See [Differential Expression](deg.md) for a detailed walkthrough.

## Resource Requirements

Per-job; SCENIC and COMPASS parallelize per cell type (and per sex for COMPASS).

| Analysis | Cores | Memory | Time | Notes |
|----------|-------|--------|------|-------|
| DEG split (`prep_celltype_splits.py`) | varies | high | a few hours | Run once per integration (reads ~80 GB h5ad) |
| DEG (pseudobulk DESeq2) | 4 (ACE) / 8 (SocIsl) | 200 GB (ACE) / 64 GB (SocIsl) | up to 24 h (ACE) / 1–2 h (SocIsl) | Loops cell type × sex internally |
| GSEA | 8 (ACE Tsai) / 45 (ACE DeJager) | 64–100 GB | 5–12 hours | Reads DESeq2 `.rda` |
| SCENIC | 32+ | 256 GB+ | 24 to 48 hours | Per cell type |
| COMPASS | 40 (OPC: 60) | 600 GB | 24 hours | Per cell type per sex; requires IBM CPLEX |
