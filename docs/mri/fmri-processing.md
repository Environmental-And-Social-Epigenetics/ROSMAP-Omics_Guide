# fMRI Processing (Stage 03)

Stage 03 runs fMRIPrep for functional MRI preprocessing followed by XCP-D for post-processing, denoising, and parcellation. This stage produces the primary functional derivatives used in downstream statistical analysis, including denoised BOLD time series, functional connectivity matrices, and ALFF maps.

## Run Command

To process all subjects:

```bash
bash 03_fmri_processing/fmriprep_xcp/submit_job_array.sh
```

To process a specific subject:

```bash
bash 03_fmri_processing/fmriprep_xcp/submit_job_array.sh sub-00123456
```

The script submits a SLURM job array with one task per subject. Concurrency is governed by `FMRIPREP_ARRAY_CONCURRENCY` in `config.sh` (default: 120).

## Processing Stages

### fMRIPrep

fMRIPrep applies the following preprocessing steps to each subject:

1. **Brain extraction** of the T1w anatomical image
2. **Tissue segmentation** into gray matter, white matter, and CSF
3. **Cortical surface reconstruction** via FreeSurfer (reused if already available)
4. **Spatial normalization** to MNI152NLin2009cAsym standard space
5. **Motion correction** of functional time series
6. **Susceptibility distortion correction** using fieldmaps (when available)
7. **Co-registration** of functional and anatomical images
8. **Confound estimation** (framewise displacement, DVARS, CompCor components, motion parameters)

### XCP-D

When enabled in the submit script, XCP-D operates on fMRIPrep outputs to perform:

1. **Confound regression** (motion parameters, CompCor components)
2. **Bandpass filtering** of the BOLD signal
3. **Parcellation** of denoised BOLD into regional time series
4. **ALFF computation** per voxel
5. **T1w/T2w ratio map** generation (used in Stage 07)

## Inputs

| Input | Path | Description |
|-------|------|-------------|
| BOLD images | `${BIDS_DIR}/sub-*/ses-*/func/*bold.nii.gz` | Resting-state functional data |
| T1w images | `${BIDS_DIR}/sub-*/ses-*/anat/*T1w.nii.gz` | Anatomical reference |
| Fieldmaps | `${BIDS_DIR}/sub-*/ses-*/fmap/*` | For distortion correction (optional) |
| fMRIPrep container | `${FMRIPREP_IMG}` | fMRIPrep Singularity image |
| XCP-D container | `${XCP_IMG}` | XCP-D Singularity image |
| FreeSurfer license | `${FREESURFER_LICENSE}` | Required for cortical reconstruction |
| TemplateFlow | `${TEMPLATEFLOW_DIR}` | Standard-space templates |

## Outputs

Outputs are written to three derivative directories:

### fMRIPrep outputs: `${OUTPUT_DIR}/fmriprep_${FMRIPREP_VERSION}/`

```
fmriprep_23.2.0a3/
    sub-<id>.html                    # Visual QC report
    sub-<id>/
        ses-<ses>/
            anat/                    # Preprocessed anatomical images
            func/                    # Preprocessed BOLD (multiple spaces)
                sub-<id>_ses-<ses>_..._desc-preproc_bold.nii.gz
                sub-<id>_ses-<ses>_..._desc-confounds_timeseries.tsv
```

### FreeSurfer outputs: `${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/`

fMRIPrep generates or reuses FreeSurfer cortical reconstructions. If Stage 04a has already completed, fMRIPrep uses the existing FreeSurfer outputs.

### XCP-D outputs: `${OUTPUT_DIR}/xcp_d_${XCP_VERSION}/`

```
xcp_d_0.6.0/
    sub-<id>/
        ses-<ses>/
            anat/
                ..._T1wT2wRatio.nii.gz     # T1/T2 ratio map
            func/
                ..._desc-denoised_bold.nii.gz
                ..._ALFF.nii.gz             # ALFF map
```

## SLURM Resources (Per Subject)

| Parameter | Value |
|-----------|-------|
| Wall time | `2-00:00:00` (2 days) |
| Memory | `16 GB` |
| CPUs | `4` |
| Typical runtime | 2 to 10 hours per subject |

## Verification

After the job array completes:

1. Confirm that each subject has an HTML report in the fMRIPrep output directory:
    ```bash
    ls ${OUTPUT_DIR}/fmriprep_${FMRIPREP_VERSION}/sub-*.html | wc -l
    ```

2. Verify derivative directories exist per subject:
    ```bash
    ls -d ${OUTPUT_DIR}/fmriprep_${FMRIPREP_VERSION}/sub-*/ses-*/func/ | head
    ```

3. If XCP-D is enabled, check for ALFF and T1/T2 ratio outputs:
    ```bash
    find ${OUTPUT_DIR}/xcp_d_${XCP_VERSION}/ -name "*ALFF*" | wc -l
    find ${OUTPUT_DIR}/xcp_d_${XCP_VERSION}/ -name "*T1wT2wRatio*" | wc -l
    ```

4. Check SLURM logs for completion messages. The final log line should indicate successful exit.

## Common Issues

**FreeSurfer license not found.** fMRIPrep requires a valid FreeSurfer license to perform cortical surface reconstruction. If the path in `FREESURFER_LICENSE` is incorrect or the file is empty, fMRIPrep will fail with a license error at startup. See [Environment Setup](setup.md#freesurfer-license) for license acquisition instructions.

**Scratch quota exceeded.** fMRIPrep writes extensive intermediate files to a per-subject scratch directory. Each subject can require 5 to 20 GB of temporary space. If your scratch filesystem has per-user quotas, consider processing subjects in smaller batches or increasing your quota.

**FreeSurfer reuse failures.** If fMRIPrep cannot read existing FreeSurfer outputs (from Stage 04a or a prior fMRIPrep run), it will attempt to re-run `recon-all`, which adds several hours to processing time. Verify that `${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/` is readable from compute nodes.

**Container or template errors.** Verify that `FMRIPREP_IMG`, `XCP_IMG`, and `TEMPLATEFLOW_DIR` all point to valid, accessible paths. Test with a single subject before launching the full array:

```bash
bash 03_fmri_processing/fmriprep_xcp/submit_job_array.sh sub-00123456
```

**Missing confound files.** If downstream analysis cannot find `confounds_timeseries.tsv` files, the fMRIPrep run may have failed after anatomical processing but before functional processing completed. Check the subject HTML report for details.
