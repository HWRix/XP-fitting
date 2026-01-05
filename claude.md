# Claude Code Context: Hot Star XP Spectral Fitting

## Project Purpose

Determine stellar parameters (Teff, log g, A_V, R_V, Mass, L, R) for stars by fitting Gaia DR3 BP/RP spectra with synthetic stellar atmosphere models. The fitting jointly constrains spectral shape, extinction, and distance (via parallax).

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
│    - Extracts Mass, logL, R for each model                      │
│    - Filters PoWR models against isochrone coverage             │
│    - Output: ./stellar_models/*.txt, model_manifest.csv         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026.v4.ipynb      │
│    - Full brute-force search over all models                    │
│    - PSM refinement: 2D quadratic in (Teff, logg)               │
│    - Interpolates flux, logL, Mass; derives R via Stefan-Boltzmann│
│    - Gordon+2023 extinction applied analytically                │
│    - Output: CSV with *_fit (PSM) and *_grid (discrete) values  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Data Files
- `Padova_isochrones.fits` — 13,500 isochrone points for (Teff, logg) → (L, M, R)
- `model_manifest.csv` — All models with Teff, logg, logL, Mass, R_Rsun
- `./griddl-gal-ob-vd3-line_calib/` — PoWR models (pre-downloaded)
- `./stellar_models/` — ATLAS+PHOENIX models normalized to 10pc
- `./BPRP_spectra/` — Downloaded Gaia XP spectra

### v4 Output Format
```
source_id, Teff_fit, logg_fit, A_V_fit, R_V_fit,      # PSM-refined
           logL_fit, Mass_fit, R_Rsun_fit,            # PSM-interpolated
           Teff_grid, logg_grid, A_V_grid, R_V_grid,  # Discrete best
           logL_grid, Mass_grid, R_Rsun_grid,         # Discrete aux
           distance_pc, chi2_red, psm_refined, n_psm_neighbors, ...
```

## Before Running v4

**Re-run** `download_stellar_models_phoenix_atlas_2026.v0.ipynb` (Cell 14 onwards) to regenerate `model_manifest.csv` without PoWR duplicates.

## Documentation Files
- `Code_Summary.md` — Physics rationale (extinction, isochrones, joint fitting)
- `Code_Details.md` — Pipeline, file formats, function documentation (includes v4 PSM details)
- `Code_issues_problems_improvements.md` — Known issues and roadmap

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
- **2026-01-04**: Added H-alpha emission detection for Teff > 7000K
  - Fits residuals with linear continuum + Gaussian (center=656.3nm, σ=3.4nm)
  - Output columns: `Ha_EW`, `Ha_EW_err`, `Ha_fit_success`
  - Vertical red dashed line at H-alpha in residuals plot

## Potential Next Steps
- [x] Regenerate model_manifest.csv (remove PoWR duplicates)
- [x] Run v4 with N_MAX_FIT=None for full sample
- [x] Add H-alpha emission detection
- [ ] Add logg=1.0 to PHOENIX grid (for giants)
- [ ] Expand R_V grid to ~10 values
- [ ] Add quality flags (boundary, degeneracy)
- [ ] Parallelization for large catalogs
- [ ] Add age from isochrones

---
Last updated: 2026-01-04
