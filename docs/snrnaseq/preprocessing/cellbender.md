# CellBender

CellBender removes ambient RNA contamination from Cell Ranger count matrices using a deep generative model. Ambient RNA, which leaks from damaged or lysed cells during nucleus isolation, contaminates every droplet with background signal that does not reflect the individual cell's true expression profile. CellBender models this contamination and produces corrected count matrices.

This step requires GPU acceleration and is the third preprocessing step for both datasets.

## How CellBender Works

CellBender fits a variational autoencoder to distinguish two sources of RNA in each droplet:

1. **Cell-associated RNA**, representing the genuine transcriptome of the captured nucleus.
2. **Ambient RNA**, representing leaked RNA from the surrounding solution.

The model is trained on all barcodes (including empty droplets, which contain only ambient RNA) to learn the ambient profile. It then subtracts the estimated ambient contribution from each cell-containing droplet, producing a corrected count matrix.

## Command

```bash
cellbender remove-background \
    --cuda \
    --input <RAW_MATRIX.h5> \
    --fpr 0 \
    --output <OUTPUT.h5>
```

### Key Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `--cuda` | (flag) | Use GPU acceleration. Required for practical runtimes. |
| `--input` | `raw_feature_bc_matrix.h5` | Cell Ranger raw output (all barcodes, not the filtered matrix). CellBender needs empty droplets to model the ambient profile. |
| `--fpr` | `0` or `0.01` | False positive rate. Controls how aggressively ambient RNA is removed. |
| `--output` | `<OUTPUT.h5>` | Path for the corrected count matrix. |

### Dataset-Specific Parameters

=== "DeJager"

    ```bash
    cellbender remove-background \
        --cuda \
        --input ${DEJAGER_COUNTS}/${LIBRARY_ID}/outs/raw_feature_bc_matrix.h5 \
        --fpr 0 \
        --output ${DEJAGER_PREPROCESSED}/${LIBRARY_ID}/processed_feature_bc_matrix.h5
    ```

    | Parameter | Value |
    |-----------|-------|
    | `--fpr` | `0` (most stringent) |

=== "Tsai"

    ```bash
    cellbender remove-background \
        --cuda \
        --input ${TSAI_CELLRANGER_OUTPUT}/<projid>/outs/raw_feature_bc_matrix.h5 \
        --expected-cells 5000 \
        --total-droplets-included 20000 \
        --fpr 0.01 \
        --epochs 150 \
        --output ${TSAI_CELLBENDER_SCRATCH}/<projid>/cellbender_output.h5
    ```

    | Parameter | Value | Description |
    |-----------|-------|-------------|
    | `--expected-cells` | 5000 | Expected number of real cells in the sample |
    | `--total-droplets-included` | 20000 | Total droplets to include in the model (cells + empty) |
    | `--fpr` | 0.01 | Allows 1% false positive rate |
    | `--epochs` | 150 | Training epochs for the model |

### Understanding the False Positive Rate (`--fpr`)

The `--fpr` parameter controls the tradeoff between removing ambient RNA and retaining true cell-associated signal:

- `--fpr 0`: Most stringent. Removes all detected ambient RNA. May remove a small amount of true signal, but produces the cleanest cell type separation.
- `--fpr 0.01`: Allows a 1% false positive rate. Slightly more permissive, retaining marginally more signal along with marginally more noise.
- Higher values retain more signal but also more ambient contamination.

For analyses that depend on clean cell type boundaries (clustering, annotation, cell type-specific differential expression), stringent FPR settings are appropriate.

## GPU Requirements

CellBender requires a CUDA-capable GPU. Without GPU acceleration, processing a single sample can take hours to days instead of minutes.

| Requirement | Specification |
|-------------|---------------|
| Minimum GPU | Any CUDA GPU with 16 GB VRAM |
| Recommended GPU | NVIDIA A100 |
| CUDA support | Required (`--cuda` flag) |

!!! warning "GPU Memory"
    Large libraries with many barcodes may exceed the memory of smaller GPUs. If CellBender reports a CUDA out-of-memory error, try reducing `--total-droplets-included` or use a GPU with more VRAM (e.g., A100 with 40 or 80 GB).

## SLURM Configuration

=== "DeJager"

    ```bash
    #SBATCH --partition=mit_normal_gpu
    #SBATCH --gres=gpu:a100:1
    #SBATCH --cpus-per-task=32
    #SBATCH --mem=128G
    #SBATCH --time=47:00:00
    ```

=== "Tsai"

    ```bash
    #SBATCH --partition=mit_normal_gpu
    #SBATCH --gres=gpu:1
    #SBATCH --cpus-per-task=4
    #SBATCH --mem=64G
    #SBATCH --time=4:00:00
    ```

## Running CellBender

=== "DeJager"

    Generate batch scripts using the Jupyter notebook and submit:

    ```bash
    cd Preprocessing/DeJager/03_Cellbender
    # Run DeJager_Cellbender.ipynb to generate per-library scripts
    sbatch example_cellbender.sh
    ```

=== "Tsai"

    CellBender is integrated into the Tsai batch processing pipeline. The orchestrator runs CellBender automatically after Cell Ranger completes for each batch:

    ```bash
    cd Preprocessing/Tsai/02_Cellranger_Counts
    sbatch Scripts/pipeline_slurm_wrapper.sh
    ```

    To run CellBender separately for a specific batch:

    ```bash
    ./Scripts/run_batch.sh 1 --cellbender-only
    ```

## Output

CellBender produces corrected count matrices:

=== "DeJager"

    ```
    ${DEJAGER_PREPROCESSED}/${LIBRARY_ID}/
    +-- processed_feature_bc_matrix.h5            # Corrected counts
    +-- processed_feature_bc_matrix_filtered.h5   # Filtered version
    +-- processed_feature_bc_matrix.pdf           # QC diagnostic plots
    ```

=== "Tsai"

    ```
    ${TSAI_PREPROCESSED}/<projid>/
    +-- cellbender_output.h5                      # Corrected counts
    +-- cellbender_output_filtered.h5             # Filtered version
    ```

The `_filtered.h5` file is the primary input for the Processing pipeline (Stage 1: QC Filtering). It contains the corrected count matrix with only cell-containing barcodes.

## Output Verification

After CellBender completes, verify the output:

1. **Check that output files exist** and are non-empty:

    ```bash
    ls -lh ${TSAI_PREPROCESSED}/<projid>/cellbender_output_filtered.h5
    ```

2. **Inspect the QC plots** (DeJager): The PDF output includes diagnostic plots showing the model's cell/empty droplet classification and the estimated ambient profile.

3. **Check training convergence**: CellBender logs training loss per epoch to stderr. A steadily decreasing loss indicates successful convergence. Erratic or increasing loss suggests convergence issues.

## Troubleshooting

**CUDA out of memory.** Reduce `--total-droplets-included` or use a GPU with more VRAM. The A100 (40 GB or 80 GB variants) handles the largest libraries in this dataset.

**Convergence issues.** If the training loss does not decrease, try adjusting `--epochs` (increase to 200-300) or `--learning-rate`. Poor convergence can also indicate an unusual sample (e.g., very few cells or very high ambient contamination).

**Empty output.** Verify that the input file is the `raw_feature_bc_matrix.h5` (not the filtered version). CellBender needs the full barcode set, including empty droplets, to model ambient RNA.

**Scratch space.** CellBender writes temporary files during training. For the Tsai pipeline, the orchestrator cleans up scratch between batches, freeing approximately 750 GB per batch.
