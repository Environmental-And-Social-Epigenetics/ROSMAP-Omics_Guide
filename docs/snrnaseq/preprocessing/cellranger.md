# Cell Ranger

Cell Ranger v8.0.0 performs read alignment against the GRCh38 human reference genome and generates gene expression count matrices. This is the second preprocessing step, taking raw FASTQs as input and producing count matrices (and, for DeJager, BAM files) as output.

## Cell Ranger Count Command

The core command is `cellranger count`, which aligns reads, assigns them to genes, and produces per-cell count matrices:

```bash
cellranger count \
    --include-introns true \
    --nosecondary \
    --r1-length 26 \
    --transcriptome=${CELLRANGER_REF} \
    --sample <SAMPLE_ID> \
    --fastqs <FASTQ_DIR> \
    --output-dir=<OUTPUT_DIR>
```

### Key Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `--include-introns` | `true` | Critical for snRNA-seq. Nuclear RNA contains a large fraction of unspliced (intronic) reads. Without this flag, most informative reads are discarded. |
| `--nosecondary` | (flag) | Skips Cell Ranger's built-in secondary analysis (clustering, t-SNE). The pipeline performs these steps separately with more control. |
| `--r1-length` | `26` | Trims Read 1 to 26 bp. Library-specific; accommodates the barcode+UMI structure of these libraries. |
| `--transcriptome` | `${CELLRANGER_REF}` | Path to the `refdata-gex-GRCh38-2020-A` reference. Configured in `config/paths.sh`. |

### Dataset-Specific Differences

=== "DeJager"

    ```bash
    cellranger count \
        --create-bam true \
        --include-introns true \
        --nosecondary \
        --r1-length 26 \
        --id <LIBRARY_ID> \
        --transcriptome ${CELLRANGER_REF} \
        --sample <SAMPLE_ID> \
        --fastqs ${DEJAGER_FASTQS}/${LIBRARY_ID}/ \
        --output-dir=${DEJAGER_COUNTS}/${LIBRARY_ID}
    ```

    The `--create-bam true` flag is essential for the DeJager pipeline because the possorted BAM file is required by Demuxlet for genotype-based sample demultiplexing. This flag is not needed for Tsai.

=== "Tsai"

    ```bash
    cellranger count \
        --create-bam false \
        --include-introns true \
        --nosecondary \
        --r1-length 26 \
        --transcriptome ${CELLRANGER_REF} \
        --sample <SAMPLE_ID> \
        --fastqs ${TSAI_FASTQS_DIR}/<projid>/ \
        --output-dir=${TSAI_CELLRANGER_OUTPUT}/<projid>
    ```

    BAM creation is disabled (`--create-bam false`) because patient identity is known from metadata and no demultiplexing is needed. This saves substantial disk space and processing time.

    Cell Ranger automatically discovers all FASTQs in subdirectories when `--fastqs` points to the patient directory.

## Batch Processing Strategy

=== "DeJager"

    A Python script (`Count_DeJager.py`) reads library IDs from the FASTQ CSV, generates a SLURM batch script for each library, and submits all jobs to the cluster:

    ```bash
    cd Preprocessing/DeJager/02_Cellranger_Counts
    python Count_DeJager.py
    ```

    Alternatively, submit a single library using the example script:

    ```bash
    sbatch example_count.sh
    ```

=== "Tsai"

    Due to the 1 TB scratch space limit on the cluster, the 480 Tsai patients are processed in **16 batches of 30 patients** each. An automated orchestrator manages the entire process:

    ```bash
    cd Preprocessing/Tsai/02_Cellranger_Counts

    # Generate per-patient batch scripts (dry-run first)
    python Scripts/generate_batch_scripts.py --dry-run
    python Scripts/generate_batch_scripts.py

    # Fix problematic FASTQ directories for 13 specific patients
    python Scripts/fix_220311Tsa_patients.py

    # Submit the master pipeline (recommended)
    sbatch Scripts/pipeline_slurm_wrapper.sh
    ```

    The orchestrator processes each batch sequentially:

    1. Submit 30 Cell Ranger jobs in parallel
    2. Wait for all Cell Ranger jobs to complete
    3. Submit 30 CellBender jobs in parallel (GPU)
    4. Wait for all CellBender jobs to complete
    5. Copy final `.h5` files to permanent storage
    6. Delete scratch files for the batch (freeing ~750 GB)
    7. Move to the next batch

    Manual batch control is also available:

    ```bash
    # Run a specific batch
    ./Scripts/run_batch.sh 1

    # Run individual steps
    ./Scripts/run_batch.sh 1 --cellranger-only
    ./Scripts/run_batch.sh 1 --cellbender-only
    ./Scripts/run_batch.sh 1 --cleanup-only
    ```

## Monitoring Progress

```bash
# Check running jobs
squeue -u $USER

# Check completed patients (Tsai)
wc -l Tracking/cellranger_completed.txt
wc -l Tracking/cellbender_completed.txt

# Check failures
cat Tracking/cellranger_failed.txt

# Monitor scratch usage
du -sh ${SCRATCH_ROOT}/Tsai/
```

## Output

Cell Ranger produces the following output structure for each sample:

```
<output_dir>/
+-- outs/
    +-- raw_feature_bc_matrix.h5          # Raw counts (all barcodes, input to CellBender)
    +-- filtered_feature_bc_matrix.h5     # Cell Ranger-filtered counts
    +-- possorted_genome_bam.bam          # Aligned reads (DeJager only)
    +-- possorted_genome_bam.bam.bai      # BAM index (DeJager only)
    +-- metrics_summary.csv               # QC metrics
    +-- web_summary.html                  # Interactive QC report
```

The `raw_feature_bc_matrix.h5` file is the primary output used by the next step (CellBender). It contains counts for all barcodes, including empty droplets, which CellBender needs to model the ambient RNA profile.

## SLURM Resource Requirements

=== "DeJager"

    | Parameter | Value |
    |-----------|-------|
    | Partition | `mit_normal` |
    | Cores | 32 |
    | Memory | 128 GB |
    | Time | 47 hours |
    | Disk per library | ~10-20 GB (including BAM) |

=== "Tsai"

    | Parameter | Value |
    |-----------|-------|
    | Partition | `mit_preemptable` |
    | Cores | 16 |
    | Memory | 64 GB |
    | Time | 2 days |
    | Disk per patient | ~5-10 GB (no BAM) |

    Jobs on `mit_preemptable` may be preempted by higher-priority jobs. The pipeline orchestrator handles this by automatically requeuing preempted jobs using SLURM's `--requeue` flag.

## Known Patient Fixes (Tsai)

### 220311Tsa Patients

Thirteen patients have FASTQs in a flat directory structure that Cell Ranger cannot scan properly. The `fix_220311Tsa_patients.py` script creates symlink directories to resolve this:

Affected patients: 2518573, 10310236, 21000054, 22396591, 27586957, 34542628, 38264019, 42099069, 50111971, 62574447, 64336939, 65002324, 65499271.

### Patient 10490993

This patient has an invalid FASTQ directory entry in the source CSV. The Cell Ranger script was manually corrected to exclude the invalid directory.

## Troubleshooting

**Job failures.** Check error logs in `Logs/Errs/`:

```bash
cat Logs/Errs/cellranger_<projid>_*.err
```

**Retry failed patients (Tsai).** Use the retry script:

```bash
./Scripts/retry_failed.sh --dry-run
./Scripts/retry_failed.sh
```

**Scratch space full.** If scratch fills unexpectedly, manually run cleanup for completed batches:

```bash
./Scripts/cleanup_batch.sh 1 --force
```

**Sample naming (DeJager).** Some libraries have two samples (A and B lanes). The `Count_DeJager.py` script handles this automatically.
