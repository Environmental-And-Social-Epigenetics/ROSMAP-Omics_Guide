# Anatomical Processing (Stage 04)

Stage 04 performs cortical surface reconstruction and morphometric tabulation in two sequential sub-stages. Sub-stage 04a runs FreeSurfer's `recon-all` pipeline to reconstruct cortical surfaces and segment subcortical structures from T1w images. Sub-stage 04b tabulates the FreeSurfer outputs into analysis-ready spreadsheets using custom atlas annotations.

## Sub-stage 04a: FreeSurfer recon-all

### Run Command

```bash
bash 04_anatomical_processing/freesurfer/submit_job_array.sh
```

This submits a SLURM job array with one task per subject. Concurrency is controlled by `FREESURFER_ARRAY_CONCURRENCY` in `config.sh` (default: 100).

### Processing Steps

FreeSurfer's `recon-all` pipeline performs:

1. **Motion correction and averaging** of multiple T1w runs (if present)
2. **Intensity normalization** and bias field correction
3. **Skull stripping** (brain extraction)
4. **Subcortical segmentation** (thalamus, caudate, putamen, hippocampus, amygdala, etc.)
5. **White surface reconstruction** (gray-white boundary)
6. **Pial surface reconstruction** (gray-CSF boundary)
7. **Cortical thickness computation** at each vertex
8. **Surface inflation and registration** to the `fsaverage` template
9. **Cortical parcellation** using Desikan-Killiany and Destrieux atlases

If T2w images are available, they are incorporated to improve pial surface placement.

### Inputs

| Input | Path | Description |
|-------|------|-------------|
| T1w images | `${BIDS_DIR}/sub-*/ses-*/anat/*T1w.nii.gz` | Primary anatomical input |
| T2w images | `${BIDS_DIR}/sub-*/ses-*/anat/*T2w.nii.gz` | Optional; improves pial surface |
| FreeSurfer container | `${FREESURFER_IMG}` | FreeSurfer 7.4.1 Singularity image |
| FreeSurfer license | `${FREESURFER_LICENSE}` | Required license file |

### Outputs

```
${OUTPUT_DIR}/freesurfer_7.4.1/
    fsaverage/                       # Template subject (auto-copied)
    sub-<id>/
        mri/                         # Volumetric outputs (brain.mgz, aseg.mgz, etc.)
        surf/                        # Surface files (lh.white, rh.pial, lh.thickness, etc.)
        label/                       # Parcellation labels (lh.aparc.annot, etc.)
        stats/                       # Summary statistics (aseg.stats, lh.aparc.stats, etc.)
        scripts/
            recon-all.log            # Processing log
```

### SLURM Resources (Per Subject)

| Parameter | Value |
|-----------|-------|
| Wall time | `2-00:00:00` (2 days) |
| Memory | `20 GB` |
| CPUs | `8` |
| Typical runtime | 4 to 20 hours per subject |

### Verification

1. Confirm subject directories exist in the output:
    ```bash
    ls -d ${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/sub-* | wc -l
    ```

2. Check that `recon-all` completed successfully by inspecting the log:
    ```bash
    tail -5 ${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/sub-<id>/scripts/recon-all.log
    ```
    A successful run ends with `recon-all finished without error`.

3. Verify that key output files exist:
    ```bash
    ls ${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/sub-<id>/surf/lh.thickness
    ls ${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/sub-<id>/stats/aseg.stats
    ```

---

## Sub-stage 04b: FreeSurfer Tabulate

### Run Command

Sub-stage 04b must be run after 04a has completed for all subjects.

```bash
bash 04_anatomical_processing/freesurfer_tabulate/submit_job_array.sh
```

Concurrency is controlled by `FS_TABULATE_ARRAY_CONCURRENCY` (default: 120).

### Processing Steps

The tabulation stage:

1. **Loads FreeSurfer outputs** for each subject
2. **Applies custom atlas annotations** from `freesurfer_tabulate/annots/` to the reconstructed surfaces
3. **Extracts regional surface statistics** (thickness, area, volume, curvature) per atlas region
4. **Computes whole-brain measures** (total brain volume, intracranial volume, subcortical volumes)
5. **Writes per-subject output tables** in TSV format

### Inputs

| Input | Path | Description |
|-------|------|-------------|
| FreeSurfer outputs | `${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/sub-*` | Completed recon-all results |
| Atlas annotations | `04_anatomical_processing/freesurfer_tabulate/annots/` | Custom parcellation files |
| fMRIPrep container | `${FMRIPREP_IMG}` | Used for surface utilities |
| neuromaps container | `${NEUROMAPS_IMG}` | Used for atlas conversions |

### Outputs

```
${OUTPUT_DIR}/freesurfer_tabulate/
    sub-<id>/
        sub-<id>_brainmeasures.tsv             # Whole-brain volumetric measures
        sub-<id>_regionsurfacestats.tsv         # Per-region cortical statistics
```

The `brainmeasures` file contains global metrics including estimated total intracranial volume (eTIV), total cortical gray matter volume, and subcortical structure volumes. The `regionsurfacestats` file contains per-region columns for mean thickness, surface area, gray matter volume, and mean curvature, organized by atlas parcellation.

### SLURM Resources (Per Subject)

| Parameter | Value |
|-----------|-------|
| Wall time | `3-00:00:00` (3 days) |
| Memory | `8 GB` |
| CPUs | `4` |
| Typical runtime | 1 to 6 hours per subject |

### Verification

1. Confirm per-subject output directories exist:
    ```bash
    ls -d ${OUTPUT_DIR}/freesurfer_tabulate/sub-* | wc -l
    ```

2. Check that both output file types are present for a sample subject:
    ```bash
    ls ${OUTPUT_DIR}/freesurfer_tabulate/sub-<id>/*brainmeasures*
    ls ${OUTPUT_DIR}/freesurfer_tabulate/sub-<id>/*regionsurfacestats*
    ```

3. Verify that TSV files contain data (not just headers):
    ```bash
    wc -l ${OUTPUT_DIR}/freesurfer_tabulate/sub-<id>/*regionsurfacestats.tsv
    ```

## Common Issues

**FreeSurfer license error.** Both sub-stages require a valid FreeSurfer license. Verify the path in `config.sh` points to a non-empty `license.txt` file.

**Tabulation fails on atlas conversion.** If sub-stage 04b fails with annotation-related errors, verify that the annotation files in `freesurfer_tabulate/annots/` are present and that `NEUROMAPS_IMG` is accessible.

**No subjects found for tabulation.** Sub-stage 04b looks for completed FreeSurfer outputs. If 04a has not finished (or failed), the tabulation script will find no subjects to process. Confirm that `${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/sub-*` directories exist before running 04b.

**Incomplete recon-all runs.** If `recon-all` was interrupted (e.g., by a SLURM timeout), the subject directory may exist but be incomplete. Check `recon-all.log` for error messages. Resubmitting the job for that subject will typically resume from the last completed step.

**Missing fsaverage.** Some FreeSurfer operations require the `fsaverage` subject in the output directory. The pipeline copies it automatically, but if it is missing, copy it from the FreeSurfer container:

```bash
cp -r ${FREESURFER_HOME}/subjects/fsaverage ${OUTPUT_DIR}/freesurfer_${FREESURFER_VERSION}/
```
