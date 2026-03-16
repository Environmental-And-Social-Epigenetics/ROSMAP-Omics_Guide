# Troubleshooting

This page consolidates common issues, error messages, and resource constraints encountered when running the ROSMAP snRNA-seq pipeline. For each issue, the likely cause and recommended resolution are provided.

## Resource Requirements Summary

The pipeline includes several resource-intensive steps. The table below summarizes peak requirements to help with SLURM allocation planning.

| Step | Cores | Memory | Time | GPU | Notes |
|------|-------|--------|------|-----|-------|
| Cell Ranger (DeJager) | 32 | 128 GB | 47h | None | Per library |
| Cell Ranger (Tsai) | 16 | 64 GB | 2 days | None | Per patient; `mit_preemptable` partition |
| CellBender (DeJager) | 32 | 128 GB | 47h | A100 | Per library |
| CellBender (Tsai) | 4 | 64 GB | 4h | 1 GPU | Per patient |
| Demuxlet BAM filter | 45 | 400 GB | 3h | None | Per library, DeJager only |
| Demuxlet pileup | 10 | 500 GB | 36h | None | Per library, DeJager only |
| Stage 1: QC Filtering | 4 | 32 GB | 12h | None | Array job, per sample |
| Stage 2: Doublet Removal | 4 | 32 GB | 12h | None | Array job, per sample |
| Stage 3: Integration | 32 | 500 GB | 48h | None | Single job, all samples |
| DEG Preprocessing (NEBULA) | 8 | 200 GB | 12h | None | Single job, run once per pipeline |
| DEG per job (NEBULA) | 45 | 100 GB | 5h | None | 816 jobs total for ACE |
| DEG (DESeq2, per cell type) | 8 | 64 GB | 1-2h | None | SocIsl |
| SCENIC (Analysis) | 32+ | 256 GB+ | 24-48h | None | Per cell type |
| COMPASS (Analysis) | 40 | 600 GB | 24h | None | Per cell type per sex; requires CPLEX |
| GSEA (Analysis) | 4 | 16 GB | 1-2h | None | Per analysis |

## Scratch Space Issues

### Scratch Directories Are Cleaned Periodically

The pipeline uses scratch directories for intermediate Cell Ranger and CellBender outputs. On many clusters (including MIT Openmind), scratch filesystems are cleaned on a weekly schedule.

**Risk.** Data loss if scratch is cleaned before processing completes or before results are copied to permanent storage.

**Mitigation.**

- The Tsai batch pipeline automatically copies final CellBender outputs to permanent storage and cleans up scratch between batches.
- For manual runs, copy intermediate outputs to persistent storage promptly after completion.
- Monitor scratch usage and cleanup schedules with your cluster documentation.

```bash
# Check scratch usage
du -sh ${SCRATCH_ROOT}/Tsai/

# Manually trigger batch cleanup (Tsai)
cd Preprocessing/Tsai/02_Cellranger_Counts
./Scripts/cleanup_batch.sh 1 --force
```

### Scratch Space Full

The Tsai pipeline processes 480 patients in 16 batches of 30 specifically because of the ~1 TB scratch space limit. If scratch fills unexpectedly:

```bash
# Identify what is consuming space
du -sh ${SCRATCH_ROOT}/Tsai/*

# Force cleanup of completed batches
./Scripts/cleanup_batch.sh <batch_number> --force
```

## CellBender Issues

### CUDA Out of Memory

**Symptom.** CellBender crashes with a CUDA memory error.

**Cause.** The sample has too many barcodes for the available GPU memory.

**Fix.** Reduce `--total-droplets-included` (e.g., from 20000 to 15000) or use a GPU with more VRAM. The NVIDIA A100 (40 or 80 GB variants) handles the largest samples in this dataset.

### CellBender Convergence Issues

**Symptom.** Training loss does not decrease or oscillates erratically.

**Fix.** Increase `--epochs` (e.g., to 200 or 300) or adjust `--learning-rate`. Poor convergence can also indicate an unusual sample with very few cells or very high ambient contamination.

### Empty CellBender Output

**Symptom.** The output file is created but contains no data.

**Cause.** The input was likely the Cell Ranger `filtered_feature_bc_matrix.h5` instead of `raw_feature_bc_matrix.h5`. CellBender requires the raw (unfiltered) matrix, which includes empty droplets needed to model the ambient RNA profile.

**Fix.** Ensure the `--input` path points to `raw_feature_bc_matrix.h5`.

## Cell Ranger Issues

### Job Preempted (Tsai)

**Symptom.** Cell Ranger jobs on `mit_preemptable` are killed before completion.

**Cause.** Higher-priority jobs preempted the preemptable partition.

**Fix.** The Tsai pipeline orchestrator handles this automatically via SLURM's `--requeue` flag. If running manually, resubmit the job. Cell Ranger can resume from checkpoints in some cases.

### Sample Naming Issues (DeJager)

**Symptom.** Cell Ranger cannot find FASTQs for a library.

**Cause.** Some DeJager libraries have two samples (A and B lanes) with different naming conventions.

**Fix.** The `Count_DeJager.py` script handles this automatically. If running manually, verify that the `--sample` argument matches the FASTQ file prefix.

### 220311Tsa Patients (Tsai)

**Symptom.** Cell Ranger fails for 13 specific patients with FASTQ discovery errors.

**Cause.** These patients have FASTQs in a flat directory structure that Cell Ranger cannot scan properly.

**Fix.** Run the fix script before Cell Ranger:

```bash
python Scripts/fix_220311Tsa_patients.py
```

Affected patients: 2518573, 10310236, 21000054, 22396591, 27586957, 34542628, 38264019, 42099069, 50111971, 62574447, 64336939, 65002324, 65499271.

## Demuxlet Issues (DeJager Only)

### No Cells Assigned

**Possible causes and fixes:**

- **VCF missing PL field.** Verify with `bcftools query -l your.vcf.gz`. The VCF must contain Phred-scaled genotype likelihoods.
- **Patient ID mismatch.** Ensure patient IDs in the VCF sample names match those in `individPat*.txt`.
- **Barcode format error.** Barcodes must be in plain text format (no header, one per line).

### Too Many Doublets

**Symptom.** More than 15-20% of cells classified as doublets.

**Fix.** Check library quality metrics from the Cell Ranger web summary. Consider lowering `--doublet-prior` from 0.1. High doublet rates may also reflect genuine loading issues.

### Low Assignment Confidence

**Possible causes:**

- **Chromosome naming inconsistency.** BAM and VCF must use the same convention (e.g., both `chr1` or both `1`). Use `sort_vcf_same_as_bam.sh` to reconcile.
- **Wrong VCF type.** Use the SNP-only VCF, not the full variant VCF.
- **VCF not indexed.** Run `tabix -p vcf your.vcf.gz`.

### Pileup Out of Memory

**Symptom.** The pileup generation job is killed by the SLURM OOM killer.

**Fix.** Verify that BAM filtering (Step A) completed successfully; an unfiltered BAM will consume far more memory during pileup. If the filtered BAM is still large, increase `--mem` in the SLURM script (up to 500 GB). The VCF may contain too many non-SNP variants if filtering was not applied.

## Processing Pipeline Issues

### Stage 3 Out of Memory

**Symptom.** The integration job is killed during sample loading or concatenation.

**Cause.** Loading all samples into a single AnnData object requires approximately 500 GB of RAM for the full Tsai dataset (476 samples).

**Fix.** Ensure your SLURM allocation requests at least 500 GB. Use a high-memory partition (e.g., `lhtsai` on MIT Openmind). For initial testing, use `--sample-ids` to process a small subset.

### R Package Missing During Doublet Removal

**Symptom.** Stage 2 fails with an R error about a missing package (e.g., `GenomeInfoDbData`).

**Fix.** The script attempts to bootstrap missing packages automatically. If this fails (e.g., due to network restrictions on compute nodes), install manually:

```bash
conda activate "${SINGLECELL_ENV}"
R -e 'BiocManager::install("GenomeInfoDbData")'
```

### Harmony Does Not Converge

**Symptom.** Harmony runs for many iterations without converging, or the corrected embedding shows poor batch mixing.

**Fix.** Check the batch variable. Too many levels (e.g., 480 patient IDs) can cause convergence issues. Use `derived_batch` (~41 levels) instead. Adjust `--harmony-theta` (lower values apply less correction).

### Poor Cell Type Annotation

**Symptom.** Clusters are annotated with unexpected or ambiguous cell types.

**Diagnosis.** Check `cluster_annotation_top3.csv`. If the top two cell types have similar scores, the cluster may represent a mixed or transitional state.

**Fix.** Try a finer clustering resolution (1.0 instead of 0.5) to resolve the cluster into subtypes. Verify that the marker gene reference (`Brain_Human_PFC_Markers_Mohammadi2020.rds`) is accessible and not corrupted.

## Pipeline Naming Confusion

Some legacy scripts in `Processing/DeJager/_legacy/` are named with "TsaiPipeline" (e.g., `firstStageTsaiPipeline.py`). This is a historical artifact: the pipeline was developed on the Tsai dataset first. The current production scripts in `Processing/DeJager/Pipeline/` use generic stage-based names (`01_qc_filter.py`, etc.) and this naming issue applies only to archived legacy scripts.

## Configuration Issues

### Verifying Configuration

Always verify your path configuration before running any pipeline step:

```bash
source config/paths.sh
check_paths
```

For a comprehensive check of all prerequisites:

```bash
bash config/preflight.sh
```

### Common Configuration Errors

**Conda initialization fails.** Verify `CONDA_INIT_SCRIPT` points to the correct conda init script for your installation (e.g., `$HOME/miniforge3/etc/profile.d/conda.sh`).

**Environment not found.** Ensure conda environments have been created (run `setup/install_envs.sh`) and that environment variables in `config/paths.sh` match the actual environment paths.

**Reference genome not found.** Set `CELLRANGER_REF` to the directory containing the extracted `refdata-gex-GRCh38-2020-A` reference (not the tarball).

## Analysis Pipeline Issues

### NEBULA Convergence Issues

**Symptom.** NEBULA returns warnings about non-convergence or produces NA p-values for some genes.

**Cause.** Typically occurs with cell types that have very few cells or subjects in a specific stratum (e.g., rare inhibitory neuron subtypes in a sex-stratified analysis).

**Fix.** The pipeline skips analyses with fewer than 50 cells or 10 subjects automatically. For borderline cases, check the convergence field in the NEBULA output object (`re$convergence`). Non-converged results should be interpreted with caution.

### NEBULA Environment Missing

**Symptom.** `run_nebula.sh` fails with "Could not find conda environment: nebulaAnalysis7".

**Fix.** The NEBULA environment is not created by `install_envs.sh --analysis`. Create it manually:

```bash
conda create -n nebulaAnalysis7 -c conda-forge r-base>=4.2
conda activate nebulaAnalysis7
R -e 'install.packages("nebula"); BiocManager::install(c("edgeR", "zellkonverter", "SingleCellExperiment"))'
R -e 'install.packages(c("dplyr", "ggplot2", "ggrepel"))'
```

### COMPASS CPLEX License Error

**Symptom.** COMPASS fails with an error about missing CPLEX or license file.

**Fix.** COMPASS requires the IBM CPLEX solver with a valid license. Obtain an academic license from [https://www.ibm.com/academic/](https://www.ibm.com/academic/) and set the environment variables before running:

```bash
export CPLEX_STUDIO_DIR=/path/to/cplex
export PATH=$CPLEX_STUDIO_DIR/bin:$PATH
```

### SCENIC Motif Database Not Found

**Symptom.** pySCENIC fails with errors about missing `.feather` ranking files or motif annotation table.

**Fix.** The motif databases (~3.5 GB total) must be downloaded separately from [https://resources.aertslab.org/cistarget/](https://resources.aertslab.org/cistarget/). Update the file paths in the SCENIC scripts to point to the downloaded databases.

### SCENIC Out of Memory

**Symptom.** The GRNBoost2 step is killed by the OOM killer during SCENIC.

**Fix.** GRNBoost2 is the most memory-intensive step. Request at least 256 GB of RAM. For large cell types (>50,000 cells), consider increasing to 512 GB or using the micropooling step to reduce the effective number of observations.

## External Data Dependencies

The following external data is not included in the repository and must be obtained separately:

| Data | Required For | Source |
|------|-------------|--------|
| WGS VCF | Demuxlet (DeJager) | ROSMAP WGS data, processed and lifted to GRCh38 |
| Reference genome | Cell Ranger | 10x Genomics (`refdata-gex-GRCh38-2020-A`) |
| Synapse credentials | DeJager FASTQ download | [synapse.org](https://www.synapse.org) account with access to syn21438684 |
| Clinical phenotype data | Downstream analysis | Tracked in `Data/Phenotypes/` (available immediately after cloning) |
| NAS credentials | Data download via `Data_Access/` scripts | Password file at `~/.smb_tsailabnas` (see [Data Access](data-access.md)) |
| SCENIC motif databases | SCENIC analysis | [resources.aertslab.org/cistarget](https://resources.aertslab.org/cistarget/) (~3.5 GB) |
| IBM CPLEX license | COMPASS analysis | [ibm.com/academic](https://www.ibm.com/academic/) (academic license) |
