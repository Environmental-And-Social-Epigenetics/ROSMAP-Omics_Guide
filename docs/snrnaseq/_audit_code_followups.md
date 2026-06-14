# Code Follow-ups (for repository maintainers)

This page records issues in the **Transcriptomics pipeline code** (not the guide) that surfaced while
auditing this documentation against the code. They are recommendations for the code repository; the guide
documents the current behavior and any workaround in the meantime.

## 1. Tsai CellBender → Stage 1 QC filename mismatch (run-blocker)

**Where:** `Processing/Tsai/Pipeline/01_qc_filter.py` (sample discovery at line ~112; input read at
line ~350) vs. the Tsai integrated CellBender output naming in
`Preprocessing/Tsai/02_Cellranger_Counts/Scripts/generate_batch_scripts.py` (lines ~136, ~183).

**Problem.** The Tsai integrated CellBender step writes `cellbender_output.h5` /
`cellbender_output_filtered.h5` into each sample directory under `${TSAI_PREPROCESSED}`. But
`01_qc_filter.py` discovers and reads `processed_feature_bc_matrix_filtered.h5` from that same directory.
Because the names differ, a fresh Tsai run hits a **missing-file failure** at Stage 1: `--list-samples`
reports nothing and the array finds no inputs. (DeJager is unaffected — its CellBender output is already
named `processed_feature_bc_matrix_filtered.h5`.)

**Documented workaround (in the guide).** See
[CellBender → Reconciling the Tsai filename for Stage 1](preprocessing/cellbender.md#reconciling-the-tsai-filename-for-stage-1):
symlink `cellbender_output_filtered.h5` → `processed_feature_bc_matrix_filtered.h5` in each sample dir
before running Stage 1.

**Recommended source fix (pick one):**

1. **Make the QC reader accept both names.** In `01_qc_filter.py`, have `discover_complete_samples()` and
   `run_qc_filter()` look for `processed_feature_bc_matrix_filtered.h5` **or**
   `cellbender_output_filtered.h5` (prefer whichever exists). This is the most backward-compatible fix and
   handles already-processed data without re-running CellBender.
2. **Standardize the writer.** Have the Tsai integrated CellBender step copy/rename its filtered output to
   `processed_feature_bc_matrix_filtered.h5` in `${TSAI_PREPROCESSED}/<projid>/`, matching DeJager and the
   QC reader.

Either fix removes the need for the per-sample symlink documented above.
