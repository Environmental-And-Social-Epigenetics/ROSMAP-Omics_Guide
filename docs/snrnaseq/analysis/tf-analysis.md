# Transcription Factor and Metabolic Analysis

The `TF/` directory in the repository houses metabolic pathway analysis using COMPASS, which characterizes metabolic flux differences between phenotype groups at the single-cell level.

!!! info "Implementation status"
    COMPASS metabolic analysis is implemented for the **SocIsl (Social Isolation)** phenotype with scripts and results for both Tsai and DeJager datasets. ACE and Resilient phenotypes have placeholder directories only. DoRothEA-based transcription factor activity analysis is planned but not yet implemented.

## COMPASS Metabolic Analysis

### Method

COMPASS (Characterizing Of Metabolic PAtterns at the Single cell Scale) estimates metabolic flux through known biochemical reactions for each cell. It uses single-cell gene expression data to infer the activity of metabolic reactions and pathways, then identifies reactions with significantly different flux between phenotype groups.

The analysis pipeline has two phases:

1. **Flux estimation** — COMPASS estimates per-cell metabolic flux scores using an optimization-based approach that requires the IBM CPLEX solver
2. **Statistical testing** — Wilcoxon rank-sum tests compare reaction activity between phenotype groups (e.g., socially isolated vs. non-isolated), with FDR correction and Cohen's d effect sizes

### Scripts

Scripts are in `Analysis/SocIsl/TF/Tsai/`:

| Script | Purpose |
|--------|---------|
| `compassRunAstTsai.sh` | Main COMPASS SLURM wrapper for astrocytes |
| `compassRun{CellType}{Sex}.sh` | Per-cell-type, per-sex COMPASS wrappers (20+ scripts) |
| `compass_analysis.py` | Post-COMPASS statistical analysis |

### COMPASS Execution

Each COMPASS run processes one cell type for one sex:

```bash
compass \
    --data matrix500_{sex}_{CellType}.tsv \
    --num-processes 40 \
    --species homo_sapiens \
    --output-dir CompassP{Sex}{CellType}New
```

The input TSV is a preprocessed gene expression matrix for the specified cell type and sex subset.

### Post-COMPASS Analysis (`compass_analysis.py`)

After COMPASS completes, the Python analysis script:

1. Loads COMPASS penalty scores and converts to reaction consistency scores (log scale)
2. Performs Wilcoxon rank-sum tests per metabolic reaction between isolated and non-isolated groups
3. Computes Cohen's d effect sizes
4. Applies FDR correction
5. Clusters correlated reactions into meta-reactions using hierarchical clustering
6. Maps results to named metabolic pathways (PGM, LDH, PDH, TPI, FACOAL, etc.)

### Cell Types

| Cell Type | Scripts |
|-----------|---------|
| Ast (Astrocytes) | `compassRunAstTsai.sh`, `compassRunAstF.sh`, `compassRunAstM.sh` |
| Exc (Excitatory neurons) | `compassRunExcTsai.sh` and variants |
| Inh (Inhibitory neurons) | `compassRunInhTsai.sh` and variants |
| Mic (Microglia) | `compassRunMicTsai.sh` and variants |
| Oli (Oligodendrocytes) | `compassRunOliTsai.sh` and variants |
| OPC (OPC) | `compassRunOPCTsai.sh` and variants |

Each cell type is run separately for female and male subjects.

### Environment

Uses `compass_analysis` (from `Analysis/envs/compass.yml`):

- Python >= 3.10
- COMPASS package
- scipy, pandas, numpy
- scanpy, anndata

!!! warning "IBM CPLEX required"
    COMPASS requires the IBM CPLEX solver, which is proprietary software. Academic licenses are available at [https://www.ibm.com/academic/](https://www.ibm.com/academic/). Set `CPLEX_STUDIO_DIR` and add the CPLEX binary to `PATH` before running.

### Outputs

| File Pattern | Description |
|-------------|-------------|
| `*_pVals{CellType}.csv` | Per-reaction p-values |
| `*_metaDF{CellType}.csv` | Meta-reaction clustering results |

### Resource Requirements

| Parameter | Value |
|-----------|-------|
| Cores | 40 |
| Memory | 600 GB |
| Time | 24 hours |
| Partition | High-memory nodes required |

---

## DoRothEA Transcription Factor Activity (Planned)

DoRothEA-based TF activity analysis is planned for future phenotypes but not yet implemented. DoRothEA is a curated resource of TF-target interactions compiled from ChIP-seq, TF binding motifs, gene expression, and literature. For each cell, it computes a TF activity score by aggregating the expression of the TF's target genes, producing a cells-by-TFs activity matrix.

### Relationship to Other Analyses

- **DEG** identifies which genes change expression, but does not explain why.
- **COMPASS** identifies metabolic pathway differences associated with the phenotype.
- **DoRothEA** (when implemented) would identify which transcription factors are likely driving expression changes, based on curated prior knowledge.
- **SCENIC** discovers regulatory networks de novo from the data, without relying on prior knowledge databases.
