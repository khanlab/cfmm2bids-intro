# cfmm2bids overview

- overview of cfmm2bids
   https://github.com/khanlab/cfmm2bids

- what is cfmm2bids?
  - snakemake workflow for reproducible, automated DICOM → BIDS conversion
  - driven by a single YAML config file per study
  - covers: query, filter, download, convert, fix, (optional) gradcorrect

- workflow stages
  1. query — search CFMM DICOM server for matching studies → studies.tsv
     - uses cfmm2tar to query by study_description, study_date, patient_name
     - query caching: skip if parameters unchanged
  2. filter — include/exclude rules, session remapping by date
     - pandas query syntax
     - remap_sessions_by_date: auto-label ses-06m, ses-12m, etc.
  3. download — download DICOM tar files via cfmm2tar
     - centralized download cache (by StudyInstanceUID)
     - --skip-derived to skip scanner-reconstructed images
  4. convert — heudiconv DICOM → BIDS
     - study-specific heuristic (trident_15T.py, ki3_15T.py, etc.)
     - QC reports: series.svg, unmapped.svg per subject/session
     - aggregate_report.html
  5. fix — post-conversion patches
     - update_json: add metadata fields (e.g. Units: rad for phase images)
     - fix_orientation_quadruped: reorient for mouse/rat data
     - split_multiecho_nifti: split multi-echo data
     - remove: delete unwanted files

- trident study examples (config/trident/)
  - vaccine.yml: premap raw patient names, constant session label, session remapping
  - ki3.yml: custom session labels (11m, 14m), split multi-echo, ki3_15T heuristic
  - lecanemab_early.yml: multiple search specs, subject ID map, session remapping
  - lecanemab_late.yml: null mapping to exclude specific subjects
  - mouse_app_mapt_apoe.yml: format string for subject IDs (AA{value})

- running the workflow
  - pixi run snakemake --configfile config/trident/vaccine.yml --cores all
  - dry-run, head=1 for testing
  - --executor slurm for cluster
  - force_requery=true when new scans acquired

- outputs
  - bids/ — final validated BIDS dataset
  - qc/aggregate_report.html — per-session series tables, fix provenance
