# Metabolic and Transcription Factor Analysis

This page covers two complementary analyses: **COMPASS** metabolic-flux analysis (in the `COMPASS/`
directory) and **DoRothEA** transcription-factor activity (in the `TFActivity/` directory).

!!! note "`TF/` was renamed to `COMPASS/`"
    Metabolic analysis used to live in a `TF/` directory; it is now **`COMPASS/`**. Transcription-factor
    activity is a separate, now-implemented **`TFActivity/`** directory. Any reference to `TF/` is stale.

!!! info "Implementation status"
    **COMPASS** metabolic analysis is implemented for **SocIsl** (Tsai + DeJager, with results) and **ACE**
    (Tsai + DeJager, refactored `compassRun.sh` + `compass_analysis.py`). **TFActivity** (DoRothEA) is
    implemented for **ACE** (Tsai, plus DeJager male arms). Only **Resilient** is a placeholder.

## COMPASS Metabolic Analysis

### Method

COMPASS (Characterizing Of Metabolic PAtterns at the Single cell Scale) estimates metabolic flux through known biochemical reactions for each cell. It uses single-cell gene expression data to infer the activity of metabolic reactions and pathways, then identifies reactions with significantly different flux between phenotype groups.

The analysis pipeline has two phases:

1. **Flux estimation** — COMPASS estimates per-cell metabolic flux scores using an optimization-based approach that requires the IBM CPLEX solver
2. **Statistical testing** — Wilcoxon rank-sum tests compare reaction activity between phenotype groups (e.g., socially isolated vs. non-isolated), with FDR correction and Cohen's d effect sizes

### Scripts

The two cohorts organize COMPASS slightly differently:

=== "SocIsl"

    `Analysis/SocIsl/COMPASS/Tsai/` (and `DeJager/`) — the original layout, one wrapper per cell type × sex:

    | Script | Purpose |
    |--------|---------|
    | `compassRun{CellType}{Sex}.sh` | Per-cell-type, per-sex COMPASS wrappers (e.g. `compassRunAstF.sh`, `compassRunExcM.sh`; 20+ scripts) |
    | `compass_analysis.py` | Post-COMPASS statistical analysis |

=== "ACE"

    `Analysis/ACE/COMPASS/Tsai/` (and `DeJager/`) — a consolidated, refactored layout:

    | Script | Purpose |
    |--------|---------|
    | `compassRun.sh` | Single parameterized COMPASS SLURM wrapper |
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

The input TSV is a preprocessed gene expression matrix for the specified cell type and sex subset (`--num-processes` is typically 40, though a few cell-type scripts use 10).

!!! question "What does COMPASS actually compute?"
    COMPASS treats each cell's expression as evidence about which metabolic reactions are active. For every
    reaction in a genome-scale human metabolic model (Human-GEM), it solves a **linear program** that finds
    the flux distribution best supporting that reaction while penalizing reactions whose enzymes are weakly
    expressed. The result is a per-cell **penalty score** per reaction — low penalty means the cell can
    sustain high flux through that reaction. This is why the solver (IBM CPLEX) is required: there is one LP
    per reaction per cell.

### Post-COMPASS Analysis (`compass_analysis.py`)

After COMPASS completes, the Python analysis script:

1. Loads COMPASS penalty scores and converts them to reaction **consistency** scores via `−log(penalty + 1)`, so that higher = more active (the log stabilizes the heavy-tailed penalty distribution)
2. Performs **Wilcoxon rank-sum** tests per metabolic reaction between phenotype groups (e.g. isolated vs. non-isolated)
3. Computes Cohen's d effect sizes
4. Applies FDR correction
5. Clusters correlated reactions into meta-reactions using hierarchical clustering
6. Maps results to named metabolic pathways (PGM = phosphoglycerate mutase, LDH = lactate dehydrogenase, PDH = pyruvate dehydrogenase, TPI = triosephosphate isomerase, FACOAL = fatty-acid-CoA ligase, etc.)

A non-parametric **Wilcoxon** test is used (rather than a t-test) because the consistency scores are bounded and non-normal, so rank-based testing is more robust to their skew and to outlier cells.

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

Uses `compass_analysis` (spec: `envs/analysis/compass/environment.yml`):

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
| Cores | 40 (most cell-type scripts; a few use 10) |
| Memory | 600 GB |
| Time | 24 hours |
| Partition | High-memory nodes required |

!!! warning "IBM CPLEX must be on PATH"
    If `compass` aborts immediately with a solver error, CPLEX is not visible. Confirm
    `CPLEX_STUDIO_DIR` is set and the CPLEX binary is on `PATH` *inside* the job (export it in the SLURM
    script, not just your login shell).

---

## TFActivity: DoRothEA Transcription Factor Activity

DoRothEA-based TF activity analysis **is implemented for ACE** (`Analysis/ACE/TFActivity/Tsai/`, plus a
DeJager male-arms variant). DoRothEA is a curated resource of TF–target interactions compiled from
ChIP-seq, TF binding motifs, inferred regulons, and literature. For each cell, it computes a TF activity
score by aggregating the (signed) expression of each TF's target genes, producing a cells-by-TFs activity
matrix that can then be tested for phenotype association — paralleling the DEG/GSEA workflow but at the
level of regulators rather than genes or pathways.

### Scripts

`Analysis/ACE/TFActivity/Tsai/`:

| Script | Purpose |
|--------|---------|
| `tf_activity_analysis.py` | Compute DoRothEA TF activity and test phenotype association |
| `run_tf_activity.sh` | SLURM wrapper |
| `aceTfActT.sh`, `aceTfActT_male_arms.sh` | Batch launchers (incl. male AD-confounding arms) |
| `tf_activity_visualize.py`, `tf_convergence_analysis.py` | Plotting and convergence diagnostics |

### Relationship to Other Analyses

- **DEG** identifies *which genes* change expression, but not *why*.
- **GSEA** groups those changes into pathways.
- **COMPASS** identifies *metabolic* pathway differences associated with the phenotype.
- **TFActivity (DoRothEA)** identifies which transcription factors likely drive the expression changes,
  using **curated prior knowledge**.
- **SCENIC** discovers regulatory networks **de novo** from the data, without prior-knowledge databases —
  the data-driven counterpart to DoRothEA.
