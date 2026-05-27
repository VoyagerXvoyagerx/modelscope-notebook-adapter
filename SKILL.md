---
name: modelscope-notebook-adapter
description: Adapt and reproduce Jupyter notebooks, Colab notebooks, GitHub tutorial notebooks, or local notebook folders in ModelScope Notebook. Use when Codex must port a tutorial to ModelScope, replace Hugging Face/Google Drive/Colab downloads with ModelScope model or dataset repositories, prepare/upload missing assets with README metadata and license/source information, execute notebooks locally or in ModelScope-compatible environments, debug dependency/data/model issues, and update reports with reproducibility results.
---

# ModelScope Notebook Adapter

## Core Workflow

1. **Inventory the source tutorial**
   - Locate the source notebook(s): local `.ipynb`, GitHub URL/repo, Colab URL, or tutorial docs.
   - Preserve official markdown/text unless the user asks for a shorter adaptation.
   - Record the official data/model sources, expected outputs, runtime, dependency versions, and any optional cells.
   - If the user has not provided enough context, briefly explain the workflow and ask for the missing notebook/source URL, target ModelScope namespace, and whether full execution is required.

2. **Audit dependencies and runtime**
   - Identify Python version, CUDA/PyTorch versions, pip/conda dependencies, CLI tools, and shell commands.
   - Prefer deterministic install cells. Pin versions only when needed for compatibility; use latest official packages when the tutorial explicitly requires the latest model/version.
   - Add early capability checks for fragile version features, e.g. inspect function signatures or enum values before expensive training.
   - Separate warnings from blockers. GPU/CUDA warnings are not model-loading errors; parser/config errors often happen before GPU use.

3. **Map assets to ModelScope**
   - Prefer official ModelScope repositories when available.
   - Otherwise prefer Hugging Face official sources, then `hf-mirror.com` if Hugging Face is inaccessible.
   - If neither local nor ModelScope has the required assets, guide the user to provide access tokens/credentials for the upstream source and for ModelScope upload.
   - Never print tokens. Use environment variables and redact secrets in logs.
   - Check license before mirroring. If the license forbids redistribution or is unclear, stop and tell the user; do not upload. If allowed, carry license/source info into README and metadata.

4. **Create or update ModelScope repositories**
   - Put datasets in dataset repositories and pretrained weights in model repositories. Do not mix all assets into one umbrella repo unless the user explicitly wants that.
   - Each repository must have `README.md` with YAML front matter and body that agree on:
     - `license`
     - source URL/name
     - file list
     - associated `models:` or `datasets:` when relevant
   - For user-owned mirrors, use clear repo IDs such as `namespace/tutorial-dataset-name` and `namespace/model-name`.
   - Keep local smoke checkpoints separate from pretrained weights unless the user explicitly asks to publish them.

5. **Adapt notebook code**
   - Add a ModelScope setup cell with `dataset_snapshot_download` and `snapshot_download`.
   - Use repo-specific local dirs such as `iclr_tutorial_assets/datasets/<repo>` and `iclr_tutorial_assets/models/<repo>`.
   - Replace external downloads (`wget`, `gdown`, `hf_hub_download`, Colab drive mounts) with ModelScope downloads.
   - Prefer system `tar -xf` for large archives with many small files; Python `tarfile.extractall()` can be much slower on notebook cloud storage.
   - For large assets, skip re-download/re-extract when the target directory already exists, but keep a clean path for first runs.
   - Preserve official cells where possible; when skipping optional online-only comparisons, leave an explicit skip cell and explain what asset is missing.

6. **Execute and verify**
   - First run a narrow smoke path: download small files, load model, load one dataset sample, run one forward pass.
   - Then run the full notebook when requested or when outputs must match official tutorials.
   - Save debug images/outputs locally when visual quality matters; inspect them directly.
   - Compare against official outputs and report exact differences, including whether differences are due to data source, model version, random seed, shorter training, or unavailable optional assets.
   - If a full run is blocked by local network/sandbox limits, say exactly which command failed and whether the notebook is expected to run in ModelScope cloud.

7. **Update reports and artifacts**
   - Update report files with:
     - notebook path and executed notebook path
     - data/model repo IDs and source URLs
     - license information
     - local/cloud runtime notes
     - reproduction metrics and consistency with official tutorial
     - known blockers or environment differences
   - Keep generated notebook source and generator scripts in sync when the repo uses a generator.

## Asset and License Rules

- Prefer official ModelScope repo IDs over user mirrors when official repos exist.
- If mirroring from Hugging Face, inspect the model/dataset card and license. If no machine-readable license exists, state the uncertainty and ask before upload.
- Include source provenance in both YAML front matter and README body.
- Use associated metadata:

```yaml
---
license: apache-2.0
repo_type: dataset
tags:
  - modelscope-notebook
source:
  - name: upstream/name
    url: https://...
models:
  - namespace/model-repo
---
```

## ModelScope Code Pattern

Use this pattern and adapt repo IDs:

```python
from pathlib import Path
from modelscope.hub.snapshot_download import dataset_snapshot_download, snapshot_download

ASSET_DIR = Path("./iclr_tutorial_assets")

def _repo_local_dir(kind, repo_id):
    return ASSET_DIR / kind / repo_id.split("/", 1)[1]

def fetch_dataset(repo_id, patterns):
    return Path(dataset_snapshot_download(
        repo_id,
        local_dir=str(_repo_local_dir("datasets", repo_id)),
        allow_patterns=patterns,
        max_workers=4,
    ))

def fetch_model(repo_id, patterns):
    return Path(snapshot_download(
        repo_id=repo_id,
        repo_type="model",
        local_dir=str(_repo_local_dir("models", repo_id)),
        allow_patterns=patterns,
        max_workers=4,
    ))
```

For large tar archives:

```python
import subprocess
from pathlib import Path

Path("data").mkdir(exist_ok=True)
if not Path("data/dataset").is_dir():
    archive = dataset_asset_dir / "dataset.tar"
    subprocess.run(["tar", "-xf", str(archive), "-C", "data"], check=True)
```

## Common Debug Checks

- **Model file exists but loading fails**: print model directory, file list, config keys, package versions, and traceback.
- **State dict shape mismatch**: compare model config features with installed package support. Example: v1.1 models may require newer architecture fields such as linear patch embeddings; old packages can instantiate v1 architecture and fail at load time.
- **Notebook cloud hangs on extraction**: separate download timing from extraction timing; use `tar -xf`, not Python extraction, for many-small-file archives.
- **`np.concat` fails on cloud**: use `np.concatenate`; numpy 1.26 does not provide `np.concat`.
- **Parser errors before training**: inspect YAML/class paths and package enum/API compatibility; these are not GPU problems.
- **ModelScope cache lock or proxy failures**: set writable cache dirs when possible, or fall back to CLI-populated cache only as a local diagnostic path.

## User Guidance

When asking the user for missing information, keep it short and remind them of the pipeline:

- "I will map the notebook assets, check license, mirror/upload missing files to ModelScope if allowed, adapt downloads, run smoke/full execution, and update the report. Please provide the notebook URL/path and target ModelScope namespace."
- If upload is needed: ask for ModelScope token and upstream access token only when required; tell the user the tokens will be used via environment variables and not written into repo files.

## References

For a concise checklist and command snippets, see `references/checklist.md`.
