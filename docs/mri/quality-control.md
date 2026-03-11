# Quality Control / MRIQC (Stage 02)

Stage 02 runs MRIQC to generate participant-level image quality metrics and visual HTML reports for the BIDS dataset. MRIQC evaluates structural (T1w, T2w) and functional (BOLD) images, producing quantitative measures such as signal-to-noise ratio, contrast-to-noise ratio, and artifact indicators. These reports are used to identify subjects with poor data quality before committing compute resources to the more expensive processing stages.

## Run Command

To process all subjects in the BIDS directory:

```bash
bash 02_quality_control/mriqc/submit_job_array.sh
```

To process a specific subset of subjects:

```bash
bash 02_quality_control/mriqc/submit_job_array.sh sub-00123456 sub-00999999
```

The script sources `config.sh`, identifies subjects in `BIDS_DIR`, and submits a SLURM job array with one task per subject. Array concurrency is controlled by `MRIQC_ARRAY_CONCURRENCY` in `config.sh` (default: 80 simultaneous tasks).

## Inputs

| Input | Path | Description |
|-------|------|-------------|
| BIDS dataset | `${BIDS_DIR}/sub-*/ses-*/` | Subject directories with NIfTI and JSON files |
| T1w images | `sub-*/ses-*/anat/*T1w.nii.gz` | Required for structural QC |
| T2w images | `sub-*/ses-*/anat/*T2w.nii.gz` | Optional; processed if present |
| BOLD images | `sub-*/ses-*/func/*bold.nii.gz` | Optional; processed if present |
| Container image | `${MRIQC_IMG}` | MRIQC Singularity image |
| TemplateFlow | `${TEMPLATEFLOW_DIR}` | Pre-downloaded standard templates |

The modalities processed are controlled by `MRIQC_MODALITIES` in `config.sh`, which defaults to `T1w T2w bold`.

## Outputs

MRIQC outputs are written to `${OUTPUT_DIR}/mriqc_${MRIQC_VERSION}/`:

```
mriqc_22.0.6/
    sub-<id>.html              # Per-subject visual report
    sub-<id>/                  # Per-subject metrics directory
        anat/
            sub-<id>_ses-<ses>_T1w.json    # QC metrics (SNR, CNR, etc.)
        func/
            sub-<id>_ses-<ses>_task-rest_bold.json
    logs/                      # Processing logs
```

Each HTML report contains:

- Slice-by-slice image mosaics for visual inspection
- Quantitative image quality metrics in tabular format
- Signal and artifact distribution plots

## SLURM Resources (Per Subject)

| Parameter | Value |
|-----------|-------|
| Wall time | `4:00:00` |
| Memory | `24 GB` |
| CPUs | `12` |
| Typical runtime | 20 to 90 minutes per subject |

## Verification

After the job array completes:

1. Check that one HTML report exists per subject in the output directory:
    ```bash
    ls ${OUTPUT_DIR}/mriqc_${MRIQC_VERSION}/sub-*.html | wc -l
    ```

2. Compare the count to the number of subjects in the BIDS directory:
    ```bash
    ls -d ${BIDS_DIR}/sub-* | wc -l
    ```

3. Review SLURM logs for any failed array tasks:
    ```bash
    grep -l "ERROR\|FAILED" slurm-*.out
    ```

4. Open several HTML reports in a browser to visually confirm image quality. Look for motion artifacts, signal dropout, or unusual intensity patterns.

## Common Issues

**Container path not found.** If jobs fail immediately, verify that `MRIQC_IMG` in `config.sh` points to an existing, readable `.img` file. Test accessibility from a compute node:

```bash
srun --pty ls -la ${MRIQC_IMG}
```

**TemplateFlow errors.** MRIQC requires standard-space templates. If compute nodes lack internet access, ensure `TEMPLATEFLOW_DIR` is populated with the necessary templates (see [Environment Setup](setup.md#templateflow-setup)).

**No subjects found.** If the submit script reports zero subjects, confirm that `BIDS_DIR` is set correctly and that subject directories follow the `sub-*` naming convention.

**Mount or permission errors.** Check SLURM logs for Singularity bind-mount failures. The submit script mounts `BIDS_DIR`, `OUTPUT_DIR`, `CACHE_DIR`, and `TEMPLATEFLOW_DIR` into the container. All paths must be accessible from compute nodes.

**Incomplete reports.** If some subjects produce output directories but no HTML file, the MRIQC run likely failed partway through. Check the subject-level log for the specific error, which is often related to malformed NIfTI headers or missing JSON sidecar fields.
