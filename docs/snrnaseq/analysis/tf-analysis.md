# Transcription Factor Activity Analysis

Transcription factor (TF) activity analysis infers the activity of transcription factors from the expression levels of their known target genes. Rather than measuring TF expression directly (which is often low and noisy in single-cell data), this approach uses the collective expression of a TF's downstream targets as a proxy for its regulatory activity.

## Method: DoRothEA

The pipeline uses DoRothEA, a curated resource of TF-target interactions compiled from multiple evidence sources (ChIP-seq, TF binding motifs, gene expression, and literature). Each TF-target interaction is assigned a confidence level from A (highest) to E (lowest), based on the number and type of supporting evidence.

For each cell, DoRothEA computes a TF activity score by aggregating the expression of the TF's target genes using a statistical test (typically VIPER or a weighted mean). The result is a cells-by-TFs activity matrix that can be analyzed in the same way as a gene expression matrix.

## Relationship to DEG and SCENIC

TF analysis is complementary to the other two analysis types:

- **DEG** identifies which genes change expression, but does not explain why.
- **TF analysis** identifies which transcription factors are likely driving those expression changes, based on curated prior knowledge.
- **SCENIC** discovers regulatory networks de novo from the data, without relying on prior knowledge databases.

TF analysis is faster and more interpretable than SCENIC, but is limited to transcription factors and interactions present in the DoRothEA database.

## Running the Analysis

Navigate to the appropriate phenotype and dataset directory:

```bash
cd Analysis/<Phenotype>/TF/<Dataset>/
```

Follow the phenotype-specific README for the exact commands and configuration.

## Inputs

| File | Source |
|------|--------|
| Annotated AnnData (`.h5ad`) | `Processing/<Dataset>/Pipeline/` output from Stage 3 |
| DoRothEA regulon database | Included in the `dorothea` Python or R package |
| Clinical phenotype CSV | `Data/Phenotypes/` |

## Outputs

| Output | Description |
|--------|-------------|
| TF activity matrix | Per-cell activity scores for each transcription factor |
| Differential TF activity | TFs with significantly different activity between phenotype groups, per cell type |
| TF-target networks | Visualization of which target genes contribute to each TF's activity score |

## Resource Requirements

| Parameter | Value |
|-----------|-------|
| Cores | 8 |
| Memory | 64 GB |
| Time | 2 to 4 hours |
