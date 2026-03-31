---
marp: true
theme: default
paginate: true
header: "cfmm2bids — Reproducible & Automated BIDS Conversion"
footer: "Khan Lab | https://github.com/khanlab/cfmm2bids"
style: |
  section {
    font-size: 1.4rem;
  }
  section.title {
    text-align: center;
    justify-content: center;
  }
  code {
    font-size: 0.85rem;
  }
  h1 { color: #2c5f8a; }
  h2 { color: #3a7abf; }
  a { color: #2c7fb8; }
---

<!-- _class: title -->

# cfmm2bids

### Reproducible & Automated BIDS Conversion for CFMM MRI Studies

**Khan Lab**
https://github.com/khanlab/cfmm2bids

---

## The Problem

MRI data at CFMM is stored in DICOM format on the PACS server.

**Before analysis, we need to:**
1. Find which DICOM studies belong to our project
2. Download the relevant data
3. Convert DICOMs → NIfTI in [BIDS format](https://bids-specification.readthedocs.io)
4. Apply post-conversion fixes (metadata, orientation)
5. Validate the resulting BIDS dataset

> Doing this manually is time-consuming, error-prone, and hard to reproduce.

### Solution: **cfmm2bids**

A **Snakemake workflow** that automates the entire pipeline — from querying the DICOM server to a validated BIDS dataset — driven by a single YAML config file.

---

## Workflow Overview

```
 config/trident/vaccine.yml
           │
           ▼
 ┌─────────────────┐
 │  1. Query       │  Search CFMM DICOM server → studies.tsv
 └────────┬────────┘
          ▼
 ┌─────────────────┐
 │  2. Filter      │  Include/exclude rules → studies_filtered.tsv
 └────────┬────────┘
          ▼
 ┌─────────────────┐
 │  3. Download    │  cfmm2tar → dicoms/sub-*/ses-*/
 └────────┬────────┘
          ▼
 ┌─────────────────┐
 │  4. Convert     │  heudiconv → bids-staging/ + QC reports
 └────────┬────────┘
          ▼
 ┌─────────────────┐
 │  5. Fix         │  Post-conversion fixes → bids/
 └────────┬────────┘
          ▼
      bids/  ✓ validated BIDS dataset
```

---

## Stage 1 — Query

Searches the CFMM DICOM server for matching studies.

**Configured via `search_specs` in the YAML config.**

Each specification defines:
- A DICOM query (study description, date range, patient name pattern)
- Metadata mappings to extract `subject` and `session` IDs

Output: `results/<study>/0_query/studies.tsv`

> Queries are **cached** by hash — re-running skips the query if parameters haven't changed.
> Use `--config force_requery=true` when new scans have been acquired.

Multiple `search_specs` can be combined into one BIDS dataset — useful for studies that span different scanner protocols or date ranges.

---

## Stage 1 — Query: Trident Example (vaccine study)

```yaml
search_specs:
  - dicom_query:
      study_description: "*VaccIBV"
      study_date: 20250405-          # All scans from April 5, 2025 onward
    metadata_mappings:
      subject:
        source: PatientName
        pattern: '(AS[a-z0-9A-Z]+)$' # Extract subject ID (e.g. AS177M2)
        sanitize: true               # Strip non-alphanumeric characters
      session:
        source: StudyDate            # Use scan date as session ID (remapped later)

  - dicom_query:
      study_description: "*VaccIBV"
      study_date: 20250401-20250404  # Earlier scans started at session 12m
    metadata_mappings:
      subject:
        source: PatientName
        pattern: '(AS[a-z0-9A-Z]+)$'
        sanitize: true
        premap:                      # Rename raw PatientName values
          177M2: AS177M2
      session:
        constant: '12m'              # All these scans are the 12-month timepoint
```

---

## Stage 2 — Filter

Post-filters the queried studies before download.

**Configured via `study_filter_specs`.**

- `include` / `exclude`: pandas query syntax
- `remap_sessions_by_date`: automatically rename sessions by time since first scan (or age at scan)

**Trident vaccine example:**
```yaml
study_filter_specs:
  include:
  exclude:
  remap_sessions_by_date:
    enable: true
    units: 'months'
    round_step: 3
    time_to_label:
      0: '06m'
      3: '09m'
      6: '12m'
```

> Study dates become meaningful session labels: `ses-06m`, `ses-09m`, `ses-12m`

---

## Stage 2 — Filter: Subject Mapping Examples

### Excluding subjects with null mapping (lecanemab_late study)
```yaml
subject:
  source: PatientName
  pattern: '_(AS[0-9a-zA-Z]+[MF][0-9])$'
  map:
    'AS160M1': null   # Map to null → excluded from dataset
    'AS160M2': null
```

### Renaming subject IDs (lecanemab_early study)
```yaml
subject:
  source: PatientName
  pattern: '4_([0-9]+-[0-9][MF][0-9])$'
  map:
    '64-5F2': AS303F2   # Rename to canonical subject ID
    '71-2M1': AS305M1
```

### Reformatting extracted IDs (mouse_app_mapt_apoe study)
```yaml
subject:
  source: PatientName
  pattern: 'APOE\*?[34]-([0-9\-]+[MF]#[0-9]+)$'
  sanitize: true
  format: "AA{value}"  # Prepend "AA" → e.g. AA12M1
```

---

## Stage 3 — Download

Downloads DICOM data from CFMM using `cfmm2tar`.

**Key features:**
- **Centralized download cache** indexed by `StudyInstanceUID`
  - Avoids re-downloading the same study when subject/session mappings change
  - Subject/session directories contain symlinks to cached tar files
- `--skip-derived` option (used in all Trident studies) skips scanner-derived images
- `merge_duplicate_studies: true` merges multiple same-day scans into one session

**Trident configuration (shared across studies):**
```yaml
cfmm2tar_download_options: '--skip-derived'
merge_duplicate_studies: false
download_cache: "/nfs/trident3/mri/cfmm2bids/results/download_cache"
```

Output:
```
results/download_cache/{StudyInstanceUID}/   ← cached tar archives
dicoms/sub-*/ses-*/                          ← symlinks to cache
```

---

## Stage 4 — Convert

Converts DICOMs to BIDS using **heudiconv** with a study-specific heuristic.

**The heuristic maps DICOM series → BIDS filenames.**

| Heuristic | Used by |
|-----------|---------|
| `heuristics/trident_15T.py` | vaccine, lecanemab_early, lecanemab_late |
| `heuristics/ki3_15T.py` | ki3 |
| `heuristics/Schmitz^MouseAD_heuristic.py` | mouse_app_mapt_apoe |
| `heuristics/cfmm_base.py` | General CFMM studies |

**Trident convert config:**
```yaml
heudiconv_options: '--minmeta --use-enhanced-dicom --no-sanitize-jsons'
heuristic: heuristics/trident_15T.py
dcmconfig_json: resources/dcm2niix_config.json
```

**Outputs per subject/session:**
- `bids-staging/sub-*/ses-*/` — BIDS-formatted NIfTI + JSON files
- `qc/sub-*/ses-*/series.svg` — Series list QC report
- `qc/sub-*/ses-*/unmapped.svg` — Unmapped series summary

---

## Stage 4 — QC Reports

Two QC reports are generated automatically for each subject/session.

### Series list (`series.svg`)
Shows every DICOM series with:
- Series description & protocol name
- Image dimensions, TR, TE
- Mapped BIDS filename (or "NOT MAPPED")

### Unmapped summary (`unmapped.svg`)
Lists series that were **not** mapped to BIDS — useful for spotting missing heuristic rules or unexpected acquisitions.

### Aggregate HTML report
After the fix stage, a single report consolidates all subjects:
```
results/4_fix/qc/aggregate_report.html
```
Includes validation results, fix provenance, and per-session series tables.

---

## Stage 5 — Fix

Applies post-conversion patches to make the dataset BIDS-compliant.

**Configured via `post_convert_fixes` — a list of named fix actions.**

### Fix actions available

| Action | Description |
|--------|-------------|
| `remove` | Delete files matching a glob pattern |
| `update_json` | Add/update fields in JSON sidecar files |
| `fix_orientation` | Reorient NIfTI to canonical RAS+ |
| `fix_orientation_quadruped` | Reorient for quadruped (mouse) data |
| `split_multiecho_nifti` | Split multi-echo NIfTI into separate files |
| `remove_duplicate_niftis` | Remove duplicate NIfTIs (keeps first alphabetically) |
| `intended_for` | Set `IntendedFor` in fieldmap JSON files |

Output: `bids/` — the final, validated BIDS dataset

---

## Stage 5 — Fix: Trident Examples

### Common fixes used across all Trident studies

```yaml
post_convert_fixes:
  # Add Units field to phase images (required for BIDS compliance)
  - name: phase_units
    pattern: "*/*_part-phase_*.json"
    action: update_json
    updates:
      Units: rad

  # Reorient all NIfTI files for quadruped (mouse/rat) anatomy
  - name: reset_orientation
    pattern: "*/*.nii.gz"
    action: fix_orientation_quadruped
```

### KI3-specific fix (split multi-echo data)

```yaml
  # Split multi-echo T2* images into individual echo files
  - name: split_multiecho_t2starw
    pattern: "anat/*T2starw.nii.gz"
    action: split_multiecho_nifti

  # Remove auto-generated scans.tsv (not needed)
  - name: remove_scans_tsv
    pattern: "*scans.tsv"
    action: remove
```

---

## Multi-Study Configuration

Each Trident study has its own config file under `config/trident/`:

```
config/trident/
├── ki3.yml               # KI3 study — ki3_15T heuristic, custom session labels
├── lecanemab_early.yml   # Lecanemab early — 4 search specs, subject remapping
├── lecanemab_late.yml    # Lecanemab late — null mapping to exclude subjects
├── mouse_app_mapt_apoe.yml  # Mouse AD study — format strings for IDs
└── vaccine.yml           # Vaccine study — premap + constant session
```

**All studies share a single download cache**, avoiding redundant downloads when the same scan appears in multiple studies.

```yaml
# Per-study stage directories (all studies share one cache)
stages:
  query:    "results/vaccine/0_query"
  filter:   "results/vaccine/1_filter"
  download: "results/vaccine/2_download"
  convert:  "results/vaccine/3_convert"
  fix:      "results/vaccine/4_fix"

download_cache: "/nfs/trident3/mri/cfmm2bids/results/download_cache"
```

---

## Running the Workflow

### Prerequisites

```bash
git clone https://github.com/khanlab/cfmm2bids.git
cd cfmm2bids
pixi install
```

### Basic commands

```bash
# Dry run — preview all steps without executing
pixi run snakemake --configfile config/trident/vaccine.yml --dry-run

# Process first subject only (for testing)
pixi run snakemake --configfile config/trident/vaccine.yml \
  --config head=1 --cores all

# Full workflow
pixi run snakemake --configfile config/trident/vaccine.yml --cores all

# Run a specific stage only
pixi run snakemake convert --configfile config/trident/vaccine.yml --cores all

# Force re-query (when new scans acquired)
pixi run snakemake --configfile config/trident/vaccine.yml \
  --config force_requery=true --cores all
```

---

## Running on a SLURM Cluster

```bash
pixi run snakemake --configfile config/trident/vaccine.yml \
  --executor slurm \
  --jobs 10 \
  --cores all
```

> Each download and conversion job runs as a separate SLURM job.
> Query caching prevents multiple jobs from querying the DICOM server simultaneously.

### Useful debug options

```
--dry-run / -n         Preview without executing
-R rule_name           Re-run a specific rule
--until rule_name      Run only up to (and including) a rule
--report               Generate an HTML Snakemake workflow report
--notemp               Keep all intermediate files
--rerun-incomplete     Re-run any incomplete jobs from a previous run
```

---

## Output Directory Structure

```
bids/                          ← Final validated BIDS dataset
├── dataset_description.json
├── participants.tsv
└── sub-AS177M2/
    └── ses-12m/
        └── anat/
            ├── sub-AS177M2_ses-12m_T2w.nii.gz
            └── sub-AS177M2_ses-12m_T2w.json

results/vaccine/
├── 0_query/studies.tsv        ← All matched DICOM studies
├── 1_filter/studies_filtered.tsv
├── 2_download/dicoms/         ← Symlinks to cached tar files
├── 3_convert/
│   ├── bids-staging/          ← Per-session BIDS data
│   └── qc/sub-*/ses-*/        ← series.svg, unmapped.svg
└── 4_fix/
    ├── bids-staging/          ← Fixed per-session BIDS data
    └── qc/aggregate_report.html
```

---

## Summary

| Stage | Config key | Output |
|-------|-----------|--------|
| Query | `search_specs` | `studies.tsv` |
| Filter | `study_filter_specs` | `studies_filtered.tsv` |
| Download | `cfmm2tar_download_options` | `dicoms/sub-*/ses-*/` |
| Convert | `heuristic`, `heudiconv_options` | `bids-staging/`, QC reports |
| Fix | `post_convert_fixes` | `bids/` (validated) |

**One config file per study → fully reproducible, automated BIDS conversion.**

Key features:
- 🔁 **Query caching** — skip re-query when nothing has changed
- 💾 **Shared download cache** — avoid re-downloading across studies
- 🗺️ **Flexible ID mapping** — regex, sanitize, premap, constant, format
- 📅 **Session remapping by date** — automatic longitudinal session labels
- 🔍 **QC reports** — per-session series tables + aggregate HTML report

---

<!-- _class: title -->

## Questions?

💻 GitHub: https://github.com/khanlab/cfmm2bids
📧 Contact: Ali Khan — alik@robarts.ca

---

## Appendix — Metadata Mapping Options

| Option | Description | Example |
|--------|-------------|---------|
| `source` | DICOM field to extract from | `PatientName`, `StudyDate` |
| `pattern` | Regex with capture group | `'(AS[a-z0-9A-Z]+)$'` |
| `sanitize` | Remove non-alphanumeric chars | `true` |
| `premap` | Rename raw DICOM values before extraction | `177M2: AS177M2` |
| `map` | Rename extracted values | `'64-5F2': AS303F2` |
| `constant` | Fixed value for all subjects/sessions | `'15T'` |
| `format` | Reformat extracted value with a template | `"AA{value}"` |
| `fillna` | Default when extraction returns empty | `'01'` |

---

## Appendix — Session Remapping by Date

Converts raw `StudyDate` values to interpretable longitudinal labels.

```yaml
remap_sessions_by_date:
  enable: true
  units: 'months'    # days, months, or years
  round_step: 3      # Round to nearest 3 months
  time_to_label:     # Custom label mapping
    0: '06m'
    3: '09m'
    6: '12m'
```

**How it works:**
1. For each subject, the earliest study date is treated as the baseline
2. Time since baseline is computed and rounded to the nearest `round_step`
3. Labels from `time_to_label` are applied (or auto-generated as `0m`, `3m`, etc.)

Optional: use `reference_col: PatientBirthDate` to compute age at scan instead.

---

## Appendix — Gradcorrect Stage (optional)

Applies gradient nonlinearity correction using the [gradcorrect BIDS app](https://github.com/khanlab/gradcorrect).

```yaml
gradcorrect:
  enable: true
  grad_coeff_file: "path/to/gradient_coeff.grad"
  create_bids_uncorr: true   # Also keep the uncorrected dataset
```

Run with:
```bash
pixi run snakemake --configfile config/myconfig.yml \
  --use-singularity --cores all
```

**Outputs:**
- `bids/` — Gradient-corrected BIDS dataset
- `bids_uncorr/` — Uncorrected dataset (when `create_bids_uncorr: true`)

> The corrected dataset has been resampled; keep the uncorrected version for analyses sensitive to resampling.
