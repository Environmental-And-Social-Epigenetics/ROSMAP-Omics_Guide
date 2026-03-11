# Interpreting Results

This page explains how to read the output files produced by the pipeline, what the key statistical quantities mean, and how to identify lipids with sex-differential social isolation associations.

## Output File Inventory

The pipeline generates tables and figures across Steps 02 through 05. All outputs are written to the `results/` directory.

### Tables (`results/tables/`)

| File | Source Step | Contents |
|------|-----------|----------|
| `qc_lof_scores.csv` | Step 02 | Per-sample Local Outlier Factor scores |
| `qc_shapiro_normality.csv` | Step 02 | Per-lipid Shapiro-Wilk test statistics and p-values (raw and FDR-adjusted) |
| `stats_lipid_all.csv` | Step 03 | Per-lipid OLS results, pooled cohort |
| `stats_lipid_male.csv` | Step 03 | Per-lipid OLS results, males only |
| `stats_lipid_female.csv` | Step 03 | Per-lipid OLS results, females only |
| `stats_category_all.csv` | Step 03 | Category-mean OLS results, pooled cohort |
| `stats_category_male.csv` | Step 03 | Category-mean OLS results, males only |
| `stats_category_female.csv` | Step 03 | Category-mean OLS results, females only |
| `ancova_sex_lipid.csv` | Step 03 | Per-lipid ANCOVA interaction results |
| `ancova_sex_category.csv` | Step 03 | Per-category ANCOVA interaction results |
| `sensitivity_noad_dataset.csv` | Step 04 | Filtered dataset excluding confirmed AD (`niareagansc > 2`) |
| `sensitivity_noad_lipid_*.csv` | Step 04 | Per-lipid OLS results in the no-AD cohort (all, male, female splits) |
| `sensitivity_noad_category_*.csv` | Step 04 | Category-mean OLS results in the no-AD cohort |
| `sensitivity_noad_lipid_zscore_*.csv` | Step 04 | Per-lipid OLS results using z-scored features in the no-AD cohort |

### Figures (`results/figures/`)

| File | Source Step | Contents |
|------|-----------|----------|
| `qc_lof_score_distribution.png` | Step 02 | Histogram of LOF scores across samples |
| `qc_shapiro_pvalue_distribution.png` | Step 02 | Histogram of per-lipid Shapiro-Wilk p-values |
| `volcano_all.png` | Step 05 | Volcano plot for pooled cohort |
| `volcano_male.png` | Step 05 | Volcano plot for males only |
| `volcano_female.png` | Step 05 | Volcano plot for females only |
| `volcano_sex_interaction.png` | Step 05 | Volcano plot for ANCOVA interaction term |
| `category_effects_all.png` | Step 05 | Category-level effect size bar plot |
| `lipid_distribution_plots/*.png` | Step 05 | Per-lipid scatter/regression plots |

## Understanding the Output Columns

### Per-Lipid and Category OLS Tables

These files (e.g., `stats_lipid_all.csv`, `stats_category_male.csv`) contain one row per lipid or category and the following columns:

| Column | Meaning |
|--------|---------|
| `lipid` or `category` | The lipid species name or category-mean feature name |
| `coef_SI_avg` | The estimated regression coefficient for `SI_avg`. Represents the change in lipid abundance per one-unit increase in social isolation score, holding covariates constant |
| `se_SI_avg` | Standard error of the `coef_SI_avg` estimate |
| `p_SI_avg` | Nominal (unadjusted) p-value testing the null hypothesis that `coef_SI_avg = 0` |
| `fdr_p_SI_avg` | Benjamini-Hochberg FDR-adjusted p-value, correcting for the number of lipids or categories tested |
| `r_squared` | Proportion of variance in the lipid feature explained by the full model |
| `n` | Number of observations included in the model (after listwise deletion of missing covariates) |

A positive `coef_SI_avg` means that higher social isolation is associated with higher lipid abundance. A negative coefficient means the reverse.

### ANCOVA Interaction Tables

The files `ancova_sex_lipid.csv` and `ancova_sex_category.csv` contain the same base columns as the OLS tables, plus three additional columns specific to the interaction term:

| Column | Meaning |
|--------|---------|
| `coef_interaction` | Coefficient of the `SI_avg:msex` interaction term. Represents the difference in the SI slope between males and females |
| `p_interaction` | Nominal p-value testing the null hypothesis that the SI-lipid association is the same in both sexes |
| `fdr_p_interaction` | FDR-adjusted p-value for the interaction term |

## What the Interaction Term Means

The ANCOVA model is:

```
lipid ~ SI_avg + msex + SI_avg:msex + age_death + niareagansc
```

The interaction term `SI_avg:msex` captures the extent to which the relationship between social isolation and lipid abundance differs by sex. Concretely:

- The **female SI slope** is `coef_SI_avg` (because `msex = 0` for females, so the interaction term drops out).
- The **male SI slope** is `coef_SI_avg + coef_interaction` (because `msex = 1` for males, so the interaction term adds to the main effect).
- The `coef_interaction` is therefore the difference between the male SI slope and the female SI slope.

A **positive `coef_interaction`** means the SI-lipid association is more positive (or less negative) in males than in females. A **negative `coef_interaction`** means the association is more negative (or less positive) in males.

A significant `p_interaction` (or `fdr_p_interaction`) indicates that social isolation is associated with the lipid differently in males versus females, regardless of the direction of the main effect.

## Identifying Lipids with Sex-Differential SI Associations

To systematically identify lipids where the SI-lipid relationship differs between sexes, use the following approach:

1. **Open `ancova_sex_lipid.csv`.** This is the primary file for sex interaction analysis.

2. **Sort by `fdr_p_interaction`.** Lipids with the smallest FDR-adjusted p-values have the strongest evidence for sex-differential associations.

3. **Apply a significance threshold.** A common threshold is `fdr_p_interaction < 0.05`, though more exploratory analyses may use `fdr_p_interaction < 0.10` or nominal `p_interaction < 0.05`.

4. **Examine `coef_interaction` direction.** For significant lipids, determine whether the SI association is stronger in males or females by checking the sign and magnitude of `coef_interaction`.

5. **Cross-reference with sex-stratified results.** Compare the significant interaction lipids against `stats_lipid_male.csv` and `stats_lipid_female.csv` to see the SI coefficients within each sex. This confirms the direction and magnitude of the sex-specific effects.

6. **Check the interaction volcano plot.** The file `volcano_sex_interaction.png` provides a visual summary. Lipids in the upper corners of this plot have both large interaction effect sizes and high significance.

7. **Validate with sensitivity analysis.** Compare findings against `sensitivity_noad_lipid_*.csv` to determine whether the sex-differential associations persist after excluding confirmed AD cases.

## Example Interpretation Workflow

Consider a lipid with these values in `ancova_sex_lipid.csv`:

| Column | Value |
|--------|-------|
| `lipid` | PC(36:4) |
| `coef_SI_avg` | -0.15 |
| `coef_interaction` | 0.28 |
| `p_interaction` | 0.003 |
| `fdr_p_interaction` | 0.04 |

This indicates:

- In **females** (`msex = 0`), the SI slope for PC(36:4) is -0.15, meaning higher social isolation is associated with lower PC(36:4) abundance.
- In **males** (`msex = 1`), the SI slope is -0.15 + 0.28 = +0.13, meaning higher social isolation is associated with slightly higher PC(36:4) abundance.
- The interaction is significant after FDR correction (`fdr_p_interaction = 0.04`), confirming that the SI-PC(36:4) relationship genuinely differs between sexes.

To verify, check `stats_lipid_female.csv` and `stats_lipid_male.csv` for PC(36:4) and confirm that the sex-stratified coefficients are consistent with the interaction model's estimates.

## Reproducibility

All output files are generated artifacts and are excluded from version control by default. To regenerate the complete results, rerun the pipeline from Step 01 through Step 05 as described in the [Pipeline Overview](index.md). The `results/` directory will be populated with fresh outputs.
