# Development Log: nf-pooled-cellpainting

## Format Guidelines
- Assume familiarity with Nextflow and CellProfiler
- Focus on actionable insights and technical details
- Minimize hierarchical structure and formatting
- No bold text or excessive headings
- Keep entries dated and concise

---

## 2025-09-26: Load Data CSV Generation Architecture

The pipeline generates different CellProfiler load_data.csv files at multiple stages. Currently implemented using Groovy collection operations in two subworkflows: `cellprofiler_load_data_csv` and `cellprofiler_load_data_csv_with_illum`.

Key pattern: Same samplesheet processed multiple times with different grouping keys:
- Stage 1: Group by [batch, plate, channels] → illumination calc CSV
- Stage 2: Group by [batch, plate, well] + illum files → illumination apply CSV
- Stage 3: [NOT IMPLEMENTED] Needs to discover outputs from Stage 2

Current architecture only handles pipelines working directly from samplesheet (1-2, 5-6). Pipelines 3-4, 7-9 require dynamic discovery of outputs from previous stages, which the Groovy implementation cannot handle.

Technical issues identified:

1. Cycle-based CSV generation (lines 97-213 in `cellprofiler_load_data_csv_with_illum/main.nf`) uses deeply nested logic for headers like `FileName_Cycle01_OrigA`. Frame indices vary by cycle, making debugging difficult.

2. Illumination file matching requires parsing filenames to extract cycle/channel (e.g., `Plate_Cycle1_IllumDNA.npy`) with no validation of expected vs actual files.

3. Subdirectory assignment logic for CellProfiler's required `img1/`, `img2/` structure is buried in collection transformations and inconsistent between cycle and non-cycle modes.

Root cause: Groovy/channels designed for static dataflow, but CSV generation needs dynamic file discovery, complex tabular transformations, and cross-stage file matching. Legacy Python implementation handles these naturally with pandas and filesystem scanning.

Proposed solution: Hybrid approach where Nextflow process calls Python script with JSON metadata, Python generates CSV using pandas, returns CSV path to Nextflow channel. This maintains workflow orchestration in Nextflow while leveraging appropriate tools for tabular operations.

Implementation needs:
- Python script per pipeline CSV format
- JSON schema for Nextflow-Python metadata exchange
- Process wrapper for Python execution
- File discovery mechanism for intermediate outputs

Next steps:
- Prototype Pipeline 3 CSV generation in Python
- Define metadata JSON schema
- Test subprocess performance impact

Key files:
- `subworkflows/local/cellprofiler_load_data_csv_with_illum/main.nf:97-213` - Cycle-based CSV logic
- `external/pooled-cell-painting-image-processing/lambda/lambda_functions/create_CSVs.py:97-141` - Pipeline 3 reference
- `subworkflows/local/cellpainting/main.nf:23-66` - Multi-stage CSV pattern

---

## Session Template

## YYYY-MM-DD: Topic

[Problem statement and context]

[Current implementation details]

[Technical issues with specific line references]

[Root cause analysis]

[Proposed solution with trade-offs]

[Implementation requirements]

[Next steps]

[File references]