# SISTER L2A spectral resample bundle

SISTER (Space-based Imaging Spectroscopy and Thermal PathfindER) was JPL's pathfinder workflow for the Surface Biology and Geology (SBG) mission. Its L2A spectral resample PGE takes the surface reflectance and uncertainty cubes from the SISTER ISOFIT step and resamples them to 10 nm spacing, the band set SBG was planning: bands are first averaged in groups close to the 10 nm target, then a piecewise cubic interpolator puts them on the 10 nm grid (400 to 2500 nm; 400 to 990 nm for DESIS). The output (`L2A_RSRFL`) is the input of SISTER's reflectance correction and L2B steps. The SISTER V004 products made this way are archived at the ORNL DAAC (dataset 2335).

This repository holds the PGE and the hytools release it installs as git submodules, pinned to the commits the PGE's `install.sh` names:

```sh
git clone --recurse-submodules https://github.com/gchang/sister-resample-bundle.git
```

| Submodule | Upstream | Commit | Role |
|---|---|---|---|
| `sister-resample` | https://github.com/sister-jpl/sister-resample | 5fea03771bd81876496b2c2496085e4e250ee199 (tag `3.0.0`) | the PGE: `pge_run.sh`, `generate_runconfig.py`, `spectral_resample.py` |
| `hytools` | https://github.com/EnSpec/hytools | 3bc81b5541cfee1c297cc6f91cfde2c1e69fadfb (tag `1.5.0`) | ENVI reading and writing, installed as a Python package |

Nothing inside the submodules is changed.

## Licenses

Each submodule keeps its own license: `sister-resample` is Apache-2.0 and `hytools` is GPL-3.0. See the `LICENSE` file in each.

## Software

The PGE was built on MAAP from `sister-resample/install.sh`: a conda env named `sister` with Python 3.8, then pip. The same env with conda-forge Python, from the bundle root:

```sh
conda create -n sister -c conda-forge --override-channels python=3.8 pip
conda activate sister
pip install pystac==1.8.4
pip install spectral==0.23.1
pip install ./hytools        # hytools 1.5.0
pip install scikit-image==0.19.3
```

No GDAL is needed for this step. The PGE needs no network.

## Data

- Input: the L2A reflectance and uncertainty datasets from the SISTER ISOFIT step (bundle `sister-isofit-bundle`), or the official ones. Each input is a folder named like the files it holds. The ISOFIT step writes them as `output/EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004/EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004/` and `..._004_UNC/`. For the official example (ORNL DAAC dataset 2335, "SISTER Workflow V004", CMR collection `C3114843226-ORNL_CLOUD`, Earthdata login, files under https://data.ornldaac.earthdata.nasa.gov/protected/sister/SISTER_Workflow_V004/data/), put:
  ```
  EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004/EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004.{bin,hdr}
  EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004_UNC/EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004_UNC.{bin,hdr}
  ```
- Official output to compare with, from the same place: `EXPERIMENTAL-SISTER_AVCL_L2A_RSRFL_20110513T175417_004{.bin,.hdr}`.

The example scene (AVIRIS-Classic `f110513t01p00r03`, Grand Mesa, Colorado, 2011-05-13; 83 by 722 pixels, 224 bands) takes under a minute (most of it Python start-up) and about 0.3 GB of memory.

## Running one scene

From the bundle root, in a new empty work folder that holds the two input folders:

```sh
conda activate sister
mkdir work && cd work
# put the two input folders here (see Data)
../sister-resample/pge_run.sh \
    --reflectance_dataset EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004 \
    --uncertainty_dataset EXPERIMENTAL-SISTER_AVCL_L2A_RFL_20110513T175417_004_UNC \
    --crid 004
```

`--crid 004` gives the official V004 names (default `000`); `--experimental True` (the default) adds the `EXPERIMENTAL-` prefix.

The products land in `output/EXPERIMENTAL-SISTER_AVCL_L2A_RSRFL_20110513T175417_004/`, a STAC catalog (`catalog.json`) with one subfolder for the resampled reflectance (`..._004/`, 211 bands) and one for its uncertainty (`..._004_UNC/`). Each holds the `.bin`, `.hdr`, STAC `.json` and an empty `.met.json`; the reflectance folder also holds the browse `.png`, the `.runconfig.json` and the `.log`. The reflectance subfolder is the `--reflectance_dataset` input of the next step, `sister-reflect_correct` (bundle `sister-reflect-correct-bundle`).

## Known traps

- **`source activate sister`.** `pge_run.sh` starts with that line. Inside the MAAP image, where an env called `sister` exists and conda's `activate` script is on `PATH`, it switches to that env. Anywhere else the line fails with `activate: No such file or directory` and, since the script has no `set -e`, carries on with whatever `python` is on `PATH`. Activate the env yourself before running, and do not keep an unrelated env called `sister` that the line could switch to (the preprocess step's env has the same name and different packages).
- **`numpy==1.19.2` and `gdal_array`.** Applies only if this step shares an env with the preprocess step: the `sister` package pins numpy 1.19.2, which breaks `from osgeo import gdal_array` under a newer conda GDAL. This step needs neither `sister` nor GDAL.
- **`/app` paths and `SISTER_APP_DIR`.** Not needed for this step. They matter only for the ISOFIT step, which hard-codes `/app/6s` and `/app/sRTMnet_v120.h5`.
- **ISOFIT reruns.** If the reflectance input comes from your own ISOFIT run, make sure that run was not an interrupted job rerun in the same folder: ISOFIT then skips the inversion and writes wrong reflectance without an error.
- **Reproducibility.** This step is deterministic. The next step, `sister-reflect_correct`, is not (its BRDF fit draws an unseeded random sample).

## Checked

Checked on 2026-10-25 from a fresh `git clone --recurse-submodules`, with the env built as above (Python 3.8.20, numpy 1.24.4, scipy 1.10.1, scikit-image 0.19.3, hy-tools 1.5.0), on the official V004 L2A RFL and UNC inputs, with no network: the resampled reflectance and uncertainty (`.bin` and `.hdr`) are byte-identical to the official `L2A_RSRFL` files (`cmp`; reflectance sha256 `66bc0fe288f3ad94caedd5c9eb7d4c5308fde0048d6bd089fb5a7a360dd2f478`). The run took 30 s and 0.3 GB.

