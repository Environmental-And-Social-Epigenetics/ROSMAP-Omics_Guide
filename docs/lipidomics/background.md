# Scientific Background

## Research Questions

This pipeline addresses two primary questions using ROSMAP brain lipidomics data:

1. **Are social isolation scores associated with lipid abundance across the brain lipidome?** Social isolation (measured by `SI_avg`, a continuous composite score) has been linked to adverse health outcomes, but its relationship to brain lipid composition is not well characterized. This pipeline tests whether higher social isolation predicts altered levels of individual lipids or lipid categories.

2. **Do SI-lipid associations differ between males and females?** Sex differences in brain lipid metabolism are well documented, and social isolation may affect males and females differently at the molecular level. The pipeline explicitly tests for sex-differential effects using ANCOVA interaction models.

## Brain Lipidomics in Aging and Alzheimer's Disease

Lipids constitute a large fraction of brain dry weight and play critical roles in membrane structure, cell signaling, myelination, and neuroinflammation. Alterations in brain lipid profiles have been associated with cognitive decline, Alzheimer's disease pathology, and neurodegeneration. ROSMAP provides a unique opportunity to study these associations because it links postmortem brain lipidomics with detailed longitudinal behavioral and clinical data, including social isolation measures collected over years of follow-up.

The lipidomics data used here are SERRF-normalized (Systematic Error Removal using Random Forest), a batch correction method designed to reduce technical variation while preserving biological signal.

## ANCOVA Rationale

To test whether SI-lipid associations differ by sex, the pipeline uses an analysis of covariance (ANCOVA) framework. The interaction model takes the form:

```
lipid ~ SI_avg * msex + age_death + niareagansc
```

The `SI_avg * msex` term expands to three components: the main effect of `SI_avg`, the main effect of `msex`, and their interaction (`SI_avg:msex`). The interaction coefficient captures whether the slope of the SI-lipid relationship differs between males and females. A significant interaction term indicates that social isolation is associated with a given lipid differently in men versus women.

This approach is preferable to running fully sex-stratified analyses alone because it directly tests the difference between sexes in a single model, with proper statistical power, rather than comparing p-values across separate models.

## Covariates

The regression models adjust for the following covariates, each selected for its known or plausible confounding relationship with both social isolation and brain lipid levels:

| Covariate | Variable Name | Rationale |
|-----------|--------------|-----------|
| Age at death | `age_death` | Age is a primary driver of both lipid composition changes and social network size |
| Education | `educ` | Education level correlates with social engagement and may independently affect brain composition |
| Postmortem interval | `pmi` | Time between death and tissue collection affects lipid stability and measured abundances |
| APOE genotype | `apoe_genotype` | The strongest genetic risk factor for AD; influences lipid metabolism directly |
| NIA-Reagan score | `niareagansc` | Composite neuropathological diagnosis; controls for AD pathology burden |

The `pmi` covariate is optional and can be included via the `--include-pmi` flag when running scripts. The NIA-Reagan score (`niareagansc`) also serves as the filter criterion in the sensitivity analysis (Step 04), where samples with confirmed AD pathology (`niareagansc <= 2`) are excluded.

## Sex Variable

Sex is encoded as a binary variable `msex` (1 = male, 0 = female) in the ROSMAP metadata. In the ANCOVA interaction model, this variable serves both as a main effect (capturing baseline sex differences in lipid levels) and as an interaction modifier (capturing sex-differential SI effects). In the sex-stratified OLS models (Step 03), the data are split by `msex` to run separate regressions within each sex.
