# Scientific Background

This page provides context on the MRI modalities acquired in the ROSMAP study, the rationale for each modality in the context of social isolation and brain health research, and the computational infrastructure that underlies the processing pipeline.

## MRI Modalities

### Structural MRI (T1-weighted and T2-weighted)

T1-weighted (T1w) images provide high-contrast delineation of gray and white matter boundaries, enabling volumetric segmentation of cortical and subcortical structures. T1w scans are the foundation for cortical surface reconstruction (via FreeSurfer), which produces measures of cortical thickness, surface area, volume, and curvature for each brain region.

T2-weighted (T2w) images offer complementary tissue contrast, with sensitivity to fluid content and myelin density. When combined with T1w images to compute a T1w/T2w ratio, the resulting map serves as an in vivo proxy for intracortical myelin content. This ratio has been validated against histological myelin staining and is particularly useful for studying microstructural integrity in aging and neurodegeneration.

### Functional MRI (Resting-State BOLD)

Resting-state functional MRI measures spontaneous fluctuations in the blood-oxygen-level-dependent (BOLD) signal while the participant lies quietly in the scanner without performing a task. These low-frequency fluctuations (typically 0.01 to 0.1 Hz) reflect intrinsic neural activity and can be used to identify functional connectivity networks, meaning sets of brain regions whose activity is temporally correlated.

A key derivative of resting-state fMRI is the amplitude of low-frequency fluctuations (ALFF), which quantifies the magnitude of spontaneous BOLD signal oscillations within each voxel. ALFF provides a measure of regional brain activity at rest and has been associated with cognitive function, aging trajectories, and neurodegenerative pathology.

### Diffusion-Weighted Imaging (DWI)

Diffusion-weighted imaging measures the directional diffusion of water molecules in brain tissue. In white matter, water diffuses preferentially along axonal fibers, and this directional bias can be modeled to reconstruct fiber tract trajectories. The ROSMAP dataset uses multi-shell acquisitions (e.g., 45-direction HARDI protocols) to enable advanced reconstruction methods beyond simple diffusion tensor imaging.

Generalized q-sampling imaging (GQI), the reconstruction method used in this pipeline, estimates a spin distribution function at each voxel that captures complex fiber geometries including crossing fibers. Automated tractography then identifies major white matter bundles (e.g., the corpus callosum forceps minor, cingulum, corticospinal tract) for tract-level analysis of microstructural properties.

## Relevance to Social Isolation Research

Social isolation has been associated with accelerated cognitive decline and increased dementia risk in older adults. The ROSMAP cohort, with its longitudinal clinical assessments and postmortem neuropathology, provides a unique setting to investigate the neural correlates of social isolation across multiple imaging modalities:

- **Structural MRI** can reveal whether social isolation is associated with regional atrophy, reduced cortical thickness, or altered surface morphology in regions implicated in social cognition and emotion regulation (e.g., prefrontal cortex, anterior cingulate, insula).
- **Functional MRI** measures intrinsic brain activity patterns that may differ between socially isolated and connected individuals, particularly in the default mode network and salience network, both of which are relevant to social cognitive processing.
- **Diffusion MRI** assesses white matter integrity in tracts connecting regions involved in social and cognitive processing. Reduced fractional anisotropy or altered T1w/T2w ratios along specific tracts may indicate microstructural degradation associated with social isolation.

## Containerized Neuroimaging Pipelines

Modern neuroimaging processing relies on complex software stacks with many dependencies. This pipeline uses containerized tools (run via Apptainer/Singularity on HPC clusters) to ensure reproducibility and portability.

### fMRIPrep

fMRIPrep is a robust preprocessing pipeline for functional MRI data. It applies brain extraction, spatial normalization, motion correction, susceptibility distortion correction, and confound estimation. fMRIPrep produces standardized outputs regardless of acquisition parameters, reducing the degrees of freedom in preprocessing decisions that can introduce variability across studies.

### XCP-D

XCP-D (XCP Engine - DCAN edition) is a post-processing pipeline that operates on fMRIPrep outputs. It performs confound regression, bandpass filtering, and parcellation to produce denoised BOLD time series and functional connectivity matrices. XCP-D also computes ALFF maps and generates quality assessment metrics.

### FreeSurfer

FreeSurfer performs automated cortical surface reconstruction from T1w images. Its `recon-all` pipeline segments subcortical structures, reconstructs white and pial surfaces, computes cortical thickness and volume, and maps results to standardized surface atlases. FreeSurfer outputs are used both as standalone anatomical measures and as inputs to other pipelines (fMRIPrep uses FreeSurfer surfaces for surface-based registration).

### QSIPrep and QSIRecon

QSIPrep preprocesses diffusion MRI data with head motion correction, eddy current correction, susceptibility distortion correction, and spatial normalization. QSIRecon, its reconstruction companion, applies user-specified reconstruction workflows. In this pipeline, the reconstruction specification uses DSI Studio's GQI method followed by automated tractography to identify major white matter bundles.

### MRIQC

MRIQC computes image quality metrics (signal-to-noise ratio, contrast-to-noise ratio, artifact measures) for structural and functional MRI data. It produces individual HTML reports that allow visual inspection and quantitative assessment of data quality before further processing.

## The BIDS Standard

The Brain Imaging Data Structure (BIDS) is a community standard for organizing neuroimaging data. BIDS specifies a directory hierarchy and file naming convention:

```
sub-<label>/
    ses-<label>/
        anat/
            sub-<label>_ses-<label>_T1w.nii.gz
            sub-<label>_ses-<label>_T1w.json
        func/
            sub-<label>_ses-<label>_task-rest_bold.nii.gz
            sub-<label>_ses-<label>_task-rest_bold.json
        dwi/
            sub-<label>_ses-<label>_dwi.nii.gz
            sub-<label>_ses-<label>_dwi.bval
            sub-<label>_ses-<label>_dwi.bvec
        fmap/
            (fieldmap files)
```

BIDS compliance is a prerequisite for fMRIPrep, QSIPrep, MRIQC, and most modern neuroimaging tools. These pipelines auto-discover input files based on BIDS naming conventions, eliminating the need for manual file path specification. Stage 01 of this pipeline converts the raw ROSMAP MRI data into a valid BIDS dataset.
