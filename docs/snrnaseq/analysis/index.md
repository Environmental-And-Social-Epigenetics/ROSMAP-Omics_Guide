# Analysis Overview

After processing produces annotated AnnData objects with cell type labels, the pipeline enters the analysis phase. This phase applies three complementary approaches to identify genes, transcription factors, and regulatory networks associated with phenotypes of interest.

## Analysis Types

| Type | Method | Purpose |
|------|--------|---------|
| DEG | Pseudobulk (DESeq2 or edgeR) | Identify differentially expressed genes between phenotype groups within each cell type |
| TF | DoRothEA | Infer transcription factor activity scores and map TF-target regulatory relationships |
| SCENIC | pySCENIC | Reconstruct single-cell gene regulatory networks to identify active regulons per cell type |

These three analyses answer related but distinct questions. DEG analysis identifies which genes change expression. TF analysis identifies which transcription factors are driving those changes. SCENIC identifies co-regulated gene modules (regulons) and the transcription factors that govern them.

## Directory Structure

Analyses are organized first by phenotype, then by analysis type, then by dataset:

```
Analysis/
├── _template/          # Copy to start a new phenotype
├── ACE/                # Adverse Childhood Experiences
│   ├── DEG/
│   │   ├── DeJager/
│   │   └── Tsai/
│   ├── TF/
│   │   ├── DeJager/
│   │   └── Tsai/
│   └── SCENIC/
│       ├── DeJager/
│       └── Tsai/
├── Resilient/          # Cognitive Resilience
│   └── (same structure)
└── SocIsl/             # Social Isolation
    └── (same structure)
```

## Data Requirements

All three analysis types require:

- Annotated AnnData objects from Stage 3 of the processing pipeline, with `cell_type` and `projid` columns in `obs`. These can be downloaded from the NAS without running the pipeline — see [Data Access](../data-access.md) (Entry Point D).
- Clinical phenotype data from the `Data/Phenotypes/` directory (referenced via `${PHENOTYPE_DIR}` in `config/paths.sh`). These CSVs are tracked in git and available immediately after cloning.

## Adding a New Phenotype

To analyze a new phenotype, copy the template directory and populate it with analysis scripts:

```bash
cp -r Analysis/_template/ Analysis/NewPhenotype/
```

Edit `Analysis/NewPhenotype/README.md` to define the phenotype, patient selection criteria, and any phenotype-specific covariates.

## Resource Requirements

| Analysis | Cores | Memory | Time |
|----------|-------|--------|------|
| DEG (pseudobulk) | 8 | 64 GB | 1 to 2 hours |
| TF (DoRothEA) | 8 | 64 GB | 2 to 4 hours |
| SCENIC (pySCENIC) | 32+ | 256 GB+ | 24 to 48 hours |

SCENIC is by far the most resource-intensive analysis. Plan for dedicated high-memory node allocation when running it on full datasets.
