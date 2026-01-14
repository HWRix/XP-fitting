# Claude Code Context: Hot Star XP Spectral Fitting

## Project Purpose

Determine stellar parameters (Teff, log g, A_V, R_V, Mass, L, R, Age) for stars by fitting Gaia DR3 BP/RP spectra with synthetic stellar atmosphere models. The fitting jointly constrains spectral shape, extinction, and distance (via parallax).

## Current State

### Fitting Versions
- **v0-v2**: Grid search with varying optimizations
- **v3**: Hierarchical coarse-to-fine search (~50× speedup)
- **v4** (current): Polynomial Spectral Model (PSM) based on Rix et al. (2016). Full brute-force followed by local 2D quadratic interpolation for continuous parameter estimation

### Model Library
```
Total models: ~471 (after deduplication)
  PHOENIX: 192 models (3800-10000K)
  ATLAS:    39 models (10000-15000K)
  PoWR:    240 models (15000-56000K, filtered to isochrone coverage)

Temperature coverage: 3800-56000K
log g coverage: 2.0-4.5
```

### Code Structure

```
Three-notebook pipeline:
┌─────────────────────────────────────────────────────────────────┐
│ 1. retrieve_BPRP_spectra-export_2026.v2.ipynb                   │
│    - Input: FITS with source_id, Gmag, parallax, parallax_error │
│    - Auto-fetches ra/dec from Gaia if not in input (Cell 20)    │
│    - Output: ./BPRP_spectra/*.fits + *_enriched.fits            │
│    - Enriches catalog with Andrae+2023 params & Wang+2025 dust  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. download_stellar_models_phoenix_atlas_2026.v0.ipynb          │
│    - Downloads PHOENIX (3800-10000K) and ATLAS (10000-15000K)   │
│    - Normalizes to 10pc using Padova isochrones                 │
│    - Extracts Mass, logL, R, logAge for each model              │
│    - Filters PoWR models against isochrone coverage             │
│    - Output: ./stellar_models/*.txt, model_manifest.csv         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026.v4.ipynb      │
│    - Full brute-force search over all models                    │
│    - PSM refinement: 2D quadratic in (Teff, logg)               │
│    - Interpolates flux, logL, Mass, logAge; derives R via S-B    │
│    - Gordon+2023 extinction applied analytically                │
│    - Output: CSV with *_fit (PSM) and *_grid (discrete) values  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Data Files
- `Padova_isochrones.fits` — 13,500 isochrone points for (Teff, logg) → (L, M, R, Age)
- `model_manifest.csv` — All models with Teff, logg, logL, Mass, R_Rsun, logAge
- `./griddl-gal-ob-vd3-line_calib/` — PoWR models (pre-downloaded)
- `./stellar_models/` — ATLAS+PHOENIX models normalized to 10pc
- `./BPRP_spectra/` — Downloaded Gaia XP spectra

### Sample Datasets
- **CarOB1** (Carina OB1 association):
  - `CarOB1_OBcluster.fits` — Source catalog with Gaia source_id, ra, dec, parallax, Gmag
  - `CarOB1_OBcluster_Sptypes.fits` — Spectral types (SpT-GES, SpT-GOSSS) and luminosity classes (LC-GES, LC-GOSSS)
  - `CarOB1_prep.ipynb` — Prepares input for the pipeline (merges catalogs, outputs `CarOB1_sample.fits`)
  - `CarOB1_validation_plots.ipynb` — Validates Teff_fit and M_G_fit against SpT+LC calibrations

### v4 Output Format
```
source_id, Teff_fit, logg_fit, A_V_fit, R_V_fit,      # PSM-refined
           logL_fit, Mass_fit, logAge_fit, R_Rsun_fit,# PSM-interpolated
           Teff_grid, logg_grid, A_V_grid, R_V_grid,  # Discrete best
           logL_grid, Mass_grid, logAge_grid, R_Rsun_grid, # Discrete aux
           distance_pc, chi2_red, psm_refined, n_psm_neighbors,
           parallax_zp_offset_mas, parallax_corrected_mas,  # Parallax correction
           Ha_EW, Ha_EW_err, Ha_fit_success, ...      # H-alpha emission
```

### Parallax Zero-Point Correction

Gaia DR3 parallaxes have a systematic negative bias (measured parallaxes too small).
We apply a correction: `parallax_corrected = parallax_input + offset`

**Literature values for bright stars (G < 11 mag):**
| Source | Zero-Point | Reference |
|--------|-----------|-----------|
| Quasars (global) | -17 to -21 μas | Lindegren+2021 |
| Bright stars | -30 to -40 μas | Groenewegen 2021, 2023 |
| VLBI comparison | -38 ± 11 μas | Recent VLBI studies |

**Current setting:** `PARALLAX_ZEROPOINT_OFFSET = +0.04 mas` (40 μas)

This makes parallaxes larger → distances smaller (corrects for stars appearing too far).

## Before Running v4

**Re-run** `download_stellar_models_phoenix_atlas_2026.v0.ipynb` (Cell 14 onwards) to regenerate `model_manifest.csv` without PoWR duplicates.

## Documentation Files
- `Code_Summary.md` — Physics rationale (extinction, isochrones, joint fitting)
- `Code_Details.md` — Pipeline, file formats, function documentation (includes v4 PSM details)
- `Code_issues_problems_improvements.md` — Known issues and roadmap

### SpT-Teff Calibration (Mamajek 2019)
Used in validation: https://www.pas.rochester.edu/~emamajek/EEM_dwarf_UBVIJHK_colors_Teff.txt

| SpT | Teff (K) | SpT | Teff (K) | SpT | Teff (K) |
|-----|----------|-----|----------|-----|----------|
| O2 | 54000 | O7 | 37100 | B1 | 26000 |
| O3 | 44900 | O8 | 35100 | B2 | 20600 |
| O4 | 42900 | O9 | 33300 | B3 | 17000 |
| O5 | 41400 | B0 | 31400 | B5 | 15700 |
| O6 | 39500 | B0.5 | 29000 | B9 | 10700 |

## GitHub Repository

**Remote:** `git@github.com:HWRix/XP-fitting.git`

**Branch:** `feature/extend-models-add-mass-filter-powr`

### Commit Protocol
After making significant changes to notebooks or code files:
```bash
git add <modified-files>
git commit -m "Description of changes"
git push origin feature/extend-models-add-mass-filter-powr
```

Claude should commit and push changes at the end of each working session or after completing major features.

## Recent Changes
- **2026-01-13**: Added CarOB1 validation against spectral types
  - `CarOB1_validation_plots.ipynb` — Compares Teff_fit to Teff from spectral types
  - Uses Mamajek (2019) SpT-Teff calibration for O2-B9 dwarfs
  - Parses GOSSS/GES spectral types (O9.7, B2.5, etc.)
  - Analyzes A_V as potential source of systematics
  - Compares M_G_fit to M_G from SpT + luminosity class (LC V/IV/III/II/I)
  - Outputs: `CarOB1_Teff_validation.png`, `CarOB1_AV_systematics.png`, `CarOB1_MG_validation.png`
- **2026-01-13**: Added CarOB1 (Carina OB1) sample preparation
  - `CarOB1_prep.ipynb` — Merges cluster catalog with spectral types
  - Outputs `CarOB1_sample.fits` ready for retrieve + fitting pipeline
- **2026-01-04**: Added stellar age (logAge) from isochrones
  - `find_nearest_isochrone()` now returns logAge and logAge_std
  - model_manifest.csv includes logAge column
  - PSM interpolates logAge alongside logL and Mass
  - Output columns: `logAge_fit`, `logAge_grid`, `logAge_std_grid`
- **2026-01-04**: Added H-alpha emission detection for Teff > 4000K
  - Fits residuals with linear continuum + Gaussian (center=656.3nm, σ=3.4nm)
  - Output columns: `Ha_EW`, `Ha_EW_err`, `Ha_fit_success`
  - Vertical red dashed line at H-alpha in residuals plot

## Potential Next Steps
- [x] Regenerate model_manifest.csv (remove PoWR duplicates)
- [x] Run v4 with N_MAX_FIT=None for full sample
- [x] Add H-alpha emission detection
- [x] Add age from isochrones
- [x] Validate Teff against spectral types (CarOB1_validation_plots.ipynb)
- [ ] Add logg=1.0 to PHOENIX grid (for giants)
- [ ] Expand R_V grid to ~10 values
- [ ] Add quality flags (boundary, degeneracy)
- [ ] Parallelization for large catalogs

---
Last updated: 2026-01-13
