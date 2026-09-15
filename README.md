# sar-maize-mapper

A **Sentinel-1 SAR preprocessing and time-series feature pipeline** for crop mapping (first target
crop: maize), built on **Google Earth Engine (GEE)**.

Radar sees through clouds. In regions with persistent cloud cover, optical satellites
(Sentinel-2, Landsat) often miss the whole growing season, while Sentinel-1 records the ground
every 6–12 days regardless of weather. This repository turns raw Sentinel-1 GRD scenes into a
**clean, analysis-ready, multi-date VV + VH raster stack** plus a **pixel index raster**, so the
time series of any pixel can be pulled and compared against ground-truth polygons.

> **Current phase:** preprocessing + feature extraction only. No model training yet.

---

## What the pipeline does

```mermaid
flowchart LR
    A[AOI polygon] --> G[1. Grid<br/>master grid, chunks,<br/>pixel index]
    G --> AU[2. Audit<br/>which Sentinel-1 tracks,<br/>dates, gaps, rain]
    AU -->|you choose tracks| R[3. New run]
    R --> E[4. Export<br/>plan → confirm →<br/>GEE preprocessing<br/>per chunk x track]
    E -->|GeoTIFFs on GCS| D[5. Download<br/>+ integrity checks]
    D --> S[6. Stack<br/>per-date VRTs,<br/>QA, dates.csv]
    S --> P[7. Pixel explorer<br/>time series of any pixel]
```

Inside Earth Engine, every image goes through (details in [docs/03_pipeline_overview.md](docs/03_pipeline_overview.md)):

1. optional extra **border-noise** masking (off by default; current GRD data is already cleaned),
2. **radiometric terrain flattening** (σ⁰ → γ⁰, layover/shadow masked),
3. **mosaicking** of slices of the same pass onto the master grid,
4. **speckle filtering** in linear power (multi-temporal, same track),
5. conversion to **dB** and export.

Safety rails built in:

- exports and bulk downloads **never start without explicit confirmation** (`--yes` / `confirmed=True`);
  a dry run only *plans* tasks, and only the confirmed scope is ever submitted;
- missing dates, failed chunks and corrupt files are **reported, never silently skipped**;
- every stage is **crash-safe**: re-running the same command resumes; two processes working on the same
  run cannot corrupt each other's bookkeeping (file locks, append-only journal);
- every run is a new immutable folder; the grid never changes once built;
- CPU, RAM and thread counts are **detected at runtime** (container-aware), nothing is hard-coded.

---

## Quickstart

```bash
# 1. Environment (Python >= 3.10; rasterio wheels include GDAL, no system GDAL needed)
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"

# 2. Credentials: put the service-account JSON key in secrets/ (gitignored)
mkdir -p secrets && cp /path/to/key.json secrets/

# 3. Config: copy the example into config/ and fill every <placeholder>
cp config/pipeline.example.yaml config/my_aoi_season.yaml

# 4. Tests
pytest                                                   # unit tests, no network
SAR_PIPELINE_CONFIG=config/my_aoi_season.yaml pytest -m gee   # live, read-only Earth Engine tests

# 5. Check what the machine offers
python -m sar_pipeline --config config/my_aoi_season.yaml resources
```

### Typical run

```bash
CFG=config/my_aoi_season.yaml
python -m sar_pipeline --config $CFG grid
python -m sar_pipeline --config $CFG audit            # then review the audit and fill s1.tracks
python -m sar_pipeline --config $CFG new-run
python -m sar_pipeline --config $CFG export --pilot chunk_r01c01          # dry run: plans, shows the plan
python -m sar_pipeline --config $CFG export --pilot chunk_r01c01 --yes    # confirms + starts the pilot
python -m sar_pipeline --config $CFG monitor --yes
python -m sar_pipeline --config $CFG download          # shows size + free disk
python -m sar_pipeline --config $CFG download --yes
python -m sar_pipeline --config $CFG stack
python -m sar_pipeline --config $CFG pixel --track RO123_ASC --lon <lon> --lat <lat> --plot logs/px.png
```

The full procedure, including every checkpoint where a human must look before continuing, is in
[docs/04_runbook.md](docs/04_runbook.md). The same steps are available as notebooks in `notebooks/`.

---

## Repository layout

```
config/            pipeline.example.yaml (tracked); your own *.yaml configs (gitignored)
docs/              concepts, setup, runbook, outputs, troubleshooting, scaling, glossary
docs/developer/    interface contract (interfaces.md) and testing guide
notebooks/         01_grid_and_audit, 02_export, 03_download_and_stack, 04_pixel_explorer
src/sar_pipeline/  the Python package
  config.py        config loading + processed/ folder conventions + run versioning
  auth.py          Earth Engine + Cloud Storage authentication
  resources.py     runtime CPU/RAM/disk detection (cgroup aware)
  run_meta.py      crash-safe file writes + run metadata readers
  locking.py       cross-process file locks
  grid.py          master grid + chunks
  index.py         pixel index raster + pixel-id conversions
  audit.py         Sentinel-1 availability, gaps, rain flags, slope statistics
  s1_ard/          Earth Engine preprocessing (border noise, terrain flattening, speckle)
  manifest.py      export state machine (CSV snapshot + append-only journal)
  export.py        chunked exports, monitoring, retry policy
  download.py      GCS download + integrity checks
  stack.py         VRT stacks + QA
  pixel_query.py   pixel time series + plots
  cli.py           command-line interface
tests/             unit tests (no network) and live read-only GEE tests (-m gee)
data/  secrets/  processed/  reference/  logs/     local only, gitignored
```

What each output file contains: [docs/05_outputs_and_folders.md](docs/05_outputs_and_folders.md).

---

## Security (this repository is public)

- **Never commit** credentials, client data, client-specific configs, outputs, logs, figures or vector
  files. `.gitignore` already excludes `secrets/`, `data/`, `processed/`, `reference/`, `logs/`,
  `config/*.yaml` (except the example), rasters, `*.png`, `*.log` and vector formats.
- `tests/test_repo_hygiene.py` fails if any file git would track contains a private value: bucket,
  project id, service-account email, AOI file name (read from your local config and key) or any word
  listed in `secrets/private_terms.txt`. Run `pytest` before every commit.
- Keep the service-account key readable only by you (`chmod 600 secrets/*.json`).
- If a key is ever pushed by mistake, **revoke it in Google Cloud immediately**; deleting the
  commit is not enough.

---

## Documentation

| Doc | Read it when |
|---|---|
| [01 SAR basics](docs/01_sar_basics.md) | you are new to radar or want to know *why* each step exists |
| [02 Setup](docs/02_setup.md) | installing on a laptop or JupyterHub |
| [03 Pipeline overview](docs/03_pipeline_overview.md) | you want to understand the stages and their order |
| [04 Runbook](docs/04_runbook.md) | you are running the pipeline |
| [05 Outputs and folders](docs/05_outputs_and_folders.md) | you are looking for a file or a column |
| [06 Troubleshooting](docs/06_troubleshooting.md) | something failed |
| [07 Scaling to large AOIs](docs/07_scaling_to_large_aois.md) | processing a country-scale AOI |
| [Glossary](docs/glossary.md) | a term is unclear |
| [Developer: interfaces](docs/developer/interfaces.md) | changing code |
| [Developer: testing](docs/developer/testing.md) | writing or running tests |

---

## Credits

Earth Engine preprocessing in `src/sar_pipeline/s1_ard/` is adapted from
[adugnag/gee_s1_ard](https://github.com/adugnag/gee_s1_ard) (MIT licence, © 2021 Adugna Mullissa),
described in Mullissa et al. (2021), *Remote Sensing* 13(10), 1954, and Vollrath et al. (2020),
*Remote Sensing* 12(11), 1867. The terrain-flattening sign convention and the layover/shadow masks
were corrected and validated against real data in this repository (see
[docs/01_sar_basics.md §10](docs/01_sar_basics.md#10-terrain-correction-two-different-things)).
