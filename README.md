> **Release status: this repository will be made public after the associated paper is accepted.**
> **本仓库将在论文录用后开源。** Until then it holds only this description; code, models and documentation will be added at acceptance.

# EarthCARE–Himawari collocation and evaluation toolkit

A Python toolkit for collocating EarthCARE active-sensor profiles with geostationary Himawari AHI cloud products, running stratified product evaluations on the collocations, and training pixel-level error models that can be applied to a full-disk scan without active-sensor input.

The repository is a research pipeline, not a library: each stage is a command-line script that reads the previous stage's files and writes Parquet, CSV, Markdown or figure files.

## Pipeline

1. **Catalogue and download.** Query the EarthCARE catalogue for frames crossing a latitude/longitude box and download the Level-2 products; download the matching AHI Level-2 cloud files and reanalysis fields.
2. **Geometry and timing.** Convert between latitude/longitude and the AHI fixed grid, compute viewing and solar angles, and reconstruct the scan-line acquisition time of each image row from the Himawari Standard Data headers.
3. **Collocation.** Attach every EarthCARE profile to the AHI pixel observed closest in time, with a nominal and a parallax-corrected pixel assignment; add a second processing chain, the neighbouring scans and a same-platform imager; remove duplicated profiles in overlapping frames. One Parquet file per frame.
4. **Evaluation.** Detection scores, height errors and phase scores with frame-level bootstrap intervals; sensitivity to collocation and reference choices; diagnostics based on optical depth, layer structure and reanalysis temperature profiles.
5. **Error models.** Gradient-boosted classifiers for mask and height errors and a conformalized quantile regression for height-error intervals, with leave-one-month-out and chronological validation and several feature-set variants.
6. **Full-disk inference.** Build the model features for every pixel of one scan from the operational files and write error-probability and interval-width arrays.
7. **Figures and tables.** One script per figure; a table generator for the manuscript.

## Repository layout

```
code/           processing, experiment and figure scripts, plus bash wrappers that chain them
  ahi_geo.py, ahi_timing.py          fixed-grid geometry, parallax, scan-line timing
  catalog_*.py, dl_*.py              catalogue queries and downloads
  match_ec_*.py, atl_column_detail.py, ebd_tau.py   collocation and column enrichment
  mlib.py                            shared loader (column selection, de-duplication, land mask)
  stats_*.py, exp_*.py, budget_m1.py, cot50*.py, tau_star.py   evaluation and diagnostics
  exp_b1_transfer.py, exp_b2_cqr.py, b1_importance_nogeo.py    error models
  b1_fulldisk.py                     full-disk inference
  fig*.py, make_tables.py            figures and tables
  run_*.sh, build_when_ready.sh      wrappers
docs/           reproduction order, product versions
models/         trained models and land masks
examples/       full-disk inference walk-through
data/           file manifest of the collocation library (the library itself is archived separately)
```

## Requirements

- Python 3.12; packages pinned in `requirements.txt` (NumPy, pandas, PyArrow, SciPy, scikit-learn, XGBoost, xarray, netCDF4, h5py, fsspec, Shapely, Cartopy, Matplotlib).
- A CUDA GPU is used for model training when available; everything else runs on CPU.
- Disk space depends on how much raw input is kept; raw files can be deleted after collocation.

```bash
python3.12 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
```

## Configuration

- All paths are relative to the repository root; the bash wrappers honour `RS4_ROOT` (default `.`).
- Credentials are never stored in the repository. The download scripts read them from the environment or from files under `.secrets/`, which is git-ignored:
  - EarthCARE: `MAAP_TOKEN` (or `.secrets/maap_token`) and `MAAP_CLIENT_SECRET`
  - second processing chain (SFTP): `.secrets/ptree` (two lines: user, password)
  - reanalysis: `.secrets/cds_key`, or use the anonymous mirror script
- Model training options are command-line flags (`--variants`, `--max-train`, `--start`, `--end`); a chronological split is selected with the environment variable `RS4_TEST_FROM=YYYYMM`.

## Usage

Every script documents its arguments in the docstring at the top of the file (`head -20 code/<script>.py`). A typical run for one month:

```bash
python code/catalog_earthcare.py data/catalog --start 2025-06-01 --end 2025-07-01 --bbox 100,20,150,50
python code/dl_earthcare.py data/catalog/frames_202506.csv data/earthcare
python code/match_ec_ahi.py data/matchup --month 202506 --ec data/earthcare
python code/match_ec_ahi_jitter.py data/matchup --month 202506
python code/stats_full.py data/matchup out/stats --start 2025-06-01 --end 2025-07-01
python code/exp_b1_transfer.py data/matchup out/b1 --start 2025-06-01 --end 2026-06-01
python code/b1_fulldisk.py 20260115 0500 --models out/b1 --variant no_geo_no_jx --out out/fulldisk
```

The complete order of commands is in `docs/REPRODUCE.md`.

## Engineering notes

- **I/O.** Scripts read files with a single sequential reader and parallelise only CPU-bound work (bootstrap, fitting). Steps that rewrite Parquet files in place must not run concurrently on the same month.
- **Resumability.** Downloads write to `*.part` and rename on completion; collocation skips frames whose output exists unless `--overwrite` is given.
- **Loader.** `code/mlib.py::load()` is the single entry point for reading the collocation files; it selects columns, applies the time window, removes frame-overlap duplicates and derives the reference columns, so experiment scripts stay short.
- **Outputs.** Each experiment writes `<prefix>_summary.md` plus CSV files; figure scripts read only those files, so figures can be regenerated without the collocation library.
- **Models.** Saved as `joblib` dictionaries containing the fitted model, the ordered feature list and the calibration object.

## Licence

Code: MIT. Other released files: CC BY 4.0. See `LICENSE`.
