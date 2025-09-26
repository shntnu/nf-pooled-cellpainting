# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Nextflow implementation of the PCPIP (Pooled Cell Painting Image Processing) workflow, originally developed as an AWS Lambda-based system for processing optical pooled screening (OPS) data. The pipeline processes thousands of microscopy images from Cell Painting and barcoding experiments to enable large-scale phenotypic screening.

## Commands

### Local execution setup

```bash
# Prerequisites for local execution:
# 1. Install Nextflow (requires Java 11+):
curl -s https://get.nextflow.io | bash
chmod +x nextflow
# Add to PATH or move to system location

# 2. Install Docker Desktop:
# - Mac/Windows: Download from https://www.docker.com/products/docker-desktop
# - Linux: https://docs.docker.com/engine/install/

# 3. Ensure Docker is running:
docker --version
docker ps  # Should not error

# 4. Clone the repository:
git clone https://github.com/seqera-services/nf-pooled-cellpainting.git
cd nf-pooled-cellpainting
```

### Run the pipeline

```bash
# Basic run with test profile (uses small test dataset from S3)
nextflow run main.nf -profile test,docker --outdir ./results

# Full run with custom samplesheet
nextflow run main.nf -profile docker --input samplesheet.csv --barcodes barcodes.csv --outdir ./results

# Alternative: Use Singularity instead of Docker (useful for HPC)
nextflow run main.nf -profile test,singularity --outdir ./results
```

### Testing

```bash
# Run nf-test for specific subworkflows
nf-test test subworkflows/local/cellprofiler_load_data_csv/tests/main.nf.test
nf-test test subworkflows/local/cellprofiler_load_data_csv_with_illum/tests/main.nf.test

# Run with test profile using S3 test data
nextflow run main.nf -profile test,singularity --outdir ./results
```

### Linting and code quality

```bash
# Install and run pre-commit hooks
pip install pre-commit
pre-commit run --all-files

# Run nf-core linting (requires nf-core installation)
pip install nf-core
nf-core pipelines lint
```

### Working with forks and upstream PRs

```bash
# When working on a PR branch from upstream
# Keep tracking upstream for pulling updates:
git pull  # pulls from upstream/feat/apply_illum_barcoding

# Push your changes to your fork:
git push origin feat/apply_illum_barcoding

# To sync with upstream PR changes:
git pull upstream feat/apply_illum_barcoding
git push origin feat/apply_illum_barcoding  # update your fork

# Check branch tracking:
git branch -vv
```

### Troubleshooting

```bash
# Docker issues
# If you see "Cannot connect to the Docker daemon", start Docker:
sudo service docker start
# Add user to docker group if permission denied:
sudo usermod -aG docker $USER && newgrp docker

# Note: Running this pipeline in containerized dev environments (Gitpod, Codespaces)
# is NOT recommended due to Docker-in-Docker limitations and cgroup conflicts.
# Use local execution or cloud compute instead.
```

## Architecture

### Background

This Nextflow pipeline reimplements the legacy PCPIP workflow that consisted of 9 AWS Lambda-orchestrated pipelines:

- **Pipelines 1-4**: Cell Painting track (illumination correction, segmentation, stitching)
- **Pipelines 5-8**: Barcoding track (illumination, alignment, barcode calling, stitching)
- **Pipeline 9**: Combined analysis (alignment, feature extraction, barcode assignment)

The original specification is documented in `external/resources/pcpip-specs.md` and the legacy implementation is in `external/pooled-cell-painting-image-processing/`.

#### External Resources

The `external/pcpip_json_graphs/` directory is a symlink to the starrynight repository's PCPIP CellProfiler pipeline files in JSON format (located at `../starrynight/docs/developer/legacy/pcpip-pipelines/_ref_graph_format/json/`). These files contain the CellProfiler pipeline definitions for each of the 9 PCPIP pipelines.

To access these files, ensure the starrynight repository is cloned alongside this repository:

```bash
# Clone starrynight repo at the specific commit used for reference
cd ..
git clone https://github.com/broadinstitute/starrynight.git
cd starrynight
git checkout 1807217bc28a5c0335bd0bbb67c48de29c19fee5
```

### Current Nextflow Implementation

#### Pipeline Structure

The pipeline processes optical pooled screening (OPS) data through two parallel tracks:

1. **Cell Painting (painting)** - Captures cellular morphology with multiple fluorescent stains
2. **Barcoding** - Identifies genetic perturbations via sequencing-by-synthesis (SBS)

#### Key Components

**Main workflow** (`workflows/nf-pooled-cellpainting.nf`):

- Splits input samplesheet by `arm` field (painting/barcoding)
- Routes to appropriate subworkflow based on arm
- Supports parallel processing of multi-channel images when `multichannel_parallel` is enabled

**Subworkflows** (`subworkflows/local/`):

- `cellpainting/` - Implements Pipelines 1-2 (illumination correction for Cell Painting)
- `barcoding/` - Implements Pipelines 5-6 (illumination correction for barcoding)
- `cellprofiler_load_data_csv/` - Generates CellProfiler LoadData CSV files
- `cellprofiler_load_data_csv_with_illum/` - Generates LoadData CSV with illumination correction files

**CellProfiler Integration**:
Pipeline templates in `assets/cellprofiler/` that support dynamic channel configuration:

- `cp_illumination_calc.cppipe.template` - Cell Painting illumination calculation
- `cp_illumination_apply.cppipe.template` - Cell Painting illumination application with segmentation
- `sbs_illumination_calc.cppipe.template` - Barcoding illumination calculation
- `sbs_illumination_apply.cppipe.template` - Barcoding illumination application with alignment

**Input Requirements**:

- Samplesheet CSV with columns: `arm` (painting/barcoding), `channels`, `paths` to images
- Barcodes CSV for barcoding arm (specified via `--barcodes` parameter)
- Images organized by batch/plate/well structure

### Execution Flow

1. Pipeline initializes and validates input samplesheet
2. Splits data by arm (painting vs barcoding)
3. For multi-channel images, can process channels in parallel if configured
4. Each arm runs:
   - Illumination function calculation per channel
   - Illumination correction application with additional processing
5. Results are organized in output directory by arm and processing stage

### Not Yet Implemented

The following components from the original PCPIP workflow are not yet implemented in Nextflow:

- Pipeline 3: Segmentation quality check (CP-SegmentCheck)
- Pipeline 4: Cell Painting stitching and cropping (CP-Stitching)
- Pipeline 7: Barcoding preprocessing and barcode calling (BC-Preprocess)
- Pipeline 8: Barcoding stitching and cropping (BC-Stitching)
- Pipeline 9: Combined analysis and feature extraction (Analysis)
- FIJI-based stitching operations
- AWS Lambda orchestration layer
- Troubleshooting pipelines (6A, 7A, 8Y, 8Z)

### Configuration Parameters

Key parameters inherited from PCPIP metadata.json:

- Image grid configuration (`painting_rows`, `painting_columns`, `painting_imperwell`)
- Channel dictionary mapping microscope channels to biological stains
- Processing modes (`fast_or_slow_mode`, `one_or_many_files`)
- Stitching parameters (`overlap_pct`, `tileperside`, `final_tile_size`)
- Barcoding cycles count
