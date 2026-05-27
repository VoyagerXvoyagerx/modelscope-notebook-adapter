# ModelScope Notebook Adaptation Checklist

## Intake

- Source notebook or repo URL/path
- Target ModelScope namespace
- Required execution level: smoke, full, or report-only
- Expected official outputs and metrics
- Required secrets: upstream token, ModelScope token, GitHub token if publishing code

## Source Audit

- List notebook files and generator scripts.
- Search for external downloads:
  - `hf_hub_download`
  - `snapshot_download`
  - `wget`
  - `curl`
  - `gdown`
  - `drive.mount`
  - hard-coded URLs
- Identify file names, sizes, checksums when available, and official licenses.

## ModelScope Upload Plan

- Use official ModelScope repo if available.
- Otherwise create one dataset repo per dataset and one model repo per pretrained model.
- Add README metadata:
  - license
  - source
  - tags
  - associated models/datasets
  - file list
- Upload with `HubApi.upload_file` or CLI, using tokens from environment variables.

## Notebook Adaptation

- Add setup cell with repo constants and `fetch_dataset`/`fetch_model`.
- Replace external downloads with ModelScope downloads.
- Use `tar -xf` for large tar archives.
- Add existence checks for already extracted data.
- Add version/capability checks near imports for fragile APIs.
- Clear outputs from modified cells unless rerunning immediately.

## Validation

- Smoke:
  - import dependencies
  - download README/small file
  - locate model/data files
  - load model
  - load one sample
  - run one forward pass
- Full:
  - execute notebook with requested settings
  - save executed notebook
  - save debug figures when visual outputs matter
  - compare metrics/visuals to official tutorial

## Report

- Record:
  - environment and package versions
  - source URLs
  - ModelScope repo IDs
  - license
  - runtime
  - metrics
  - known differences
  - unresolved blockers
