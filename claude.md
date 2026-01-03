# Claude Code Context: Hot Star XP Spectral Fitting

## Project Purpose

Determine stellar parameters (Teff, log g, A_V, R_V) for stars by fitting Gaia DR3 BP/RP spectra with synthetic stellar atmosphere models. The fitting jointly constrains spectral shape, extinction, and distance (via parallax). Now also provides stellar Mass, Luminosity, and Radius from isochrone matching.

## Current State

### Git Status
- **Repository**: Initialized, tracking main notebooks and documentation
- **Main branch**: `main` — original working v0 code
- **Feature branch**: `feature/extend-models-add-mass-filter-powr` — current development (5 commits ahead)

### Recent Session (2026-01-02)
Successfully completed:
1. Extended PHOENIX models from 7600K down to 3800K (192 models)
2. Added stellar Mass extraction from Padova isochrones
3. Added PoWR filtering against isochrone coverage (480 models pass)
4. Created model_manifest.csv with all model properties
5. Updated fitting notebook to read manifest and output Mass/L/R
6. Added N_MAX_FIT parameter for test runs (default=50)
7. Fixed verification cell display bug

### Model Library Status
```
Total models: 711
  PHOENIX: 192 models (3800-10000K)
  ATLAS:    39 models (10000-15000K)
  PoWR:    480 models (15000-56000K, filtered to isochrone coverage)

Temperature coverage: 3800-56000K
log g coverage: 2.0-4.5
Mass range: 0.52-197.52 M_sun
```

### Code Structure

```
Three-notebook pipeline:
┌─────────────────────────────────────────────────────────────────┐
│ 1. retrieve_BPRP_spectra-export_2026.v0.ipynb                   │
│    - Input: FITS catalog with source_id                         │
│    - Output: ./BPRP_spectra/<source_id>.fits                    │
│    - Queries Gaia Archive for XP_SAMPLED spectra                │
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
│ 3. fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026.v0.ipynb      │
│    - Loads models from model_manifest.csv (with Mass, L, R)     │
│    - Fits each star: grid search over (Teff, logg, A_V, R_V)    │
│    - Uses Gordon+2023 extinction law                            │
│    - Joint χ² with parallax constraint                          │
│    - N_MAX_FIT parameter for test runs (default=50)             │
│    - Output: CSV with Teff, logg, A_V, Mass, logL, R_Rsun       │
└─────────────────────────────────────────────────────────────────┘
```

### Key Data Files
- `Padova_isochrones.fits` — 13,500 isochrone points for (Teff, logg) → (L, M, R)
- `model_manifest.csv` — 711 models with Teff, logg, logL, Mass, Mass_std, R_Rsun
- `./griddl-gal-ob-vd3-line_calib/` — 244 pre-downloaded PoWR models
- `./stellar_models/` — 243 ATLAS+PHOENIX models normalized to 10pc
- `./BPRP_spectra/` — Downloaded Gaia XP spectra

### Output Format
The fitting output CSV now includes:
```
source_id, Teff, logg, A_V, R_V, distance_pc, gmag, M_G, chi2_red,
model_source, logL, Mass, Mass_std, R_Rsun
```

## Documentation Files
- `Code_Summary.md` — Physics rationale (extinction, isochrones, joint fitting)
- `Code_Details.md` — Pipeline, file formats, function documentation
- `Code_issues_problems_improvements.md` — Known issues and roadmap

## Where We May Want to Go

### Immediate (ready to implement)
- [x] Test fitting with N_MAX_FIT=50 ✓ Working!
- [ ] Run full fitting (set N_MAX_FIT=None)
- [ ] Add caching to skip already-downloaded models
- [ ] Add logg=1.0 to PHOENIX grid (for giants)
- [ ] Add R_std (radius uncertainty) calculation

### Short-term Improvements
- [ ] **Expand R_V grid**: Currently [2.5, 3.1, 3.7] — recommend ~10 values
- [ ] **Add quality flags**: boundary warnings, degeneracy detection

### Medium-term Improvements
- [ ] **Parallelization**: Fitting loop is serial, could use multiprocessing
- [ ] **MCMC uncertainties**: Currently only best-fit, no posteriors
- [ ] **Validation**: Compare to APOGEE/GALAH/benchmark stars

### Longer-term / Optional
- [ ] **Metallicity dimension**: Add [M/H] ≠ 0 model grids
- [ ] **Binary handling**: Fit composite spectra

## Git Workflow

```bash
# See current branch
git branch

# See recent commits
git log --oneline -6

# If changes work, merge to main:
git checkout main
git merge feature/extend-models-add-mass-filter-powr

# Start new feature:
git checkout -b feature/new-feature-name
```

## Testing Protocol

1. Set `N_MAX_FIT = 50` in fitting notebook config
2. Run all cells
3. Check output CSV has Mass, logL, R_Rsun columns
4. Inspect a few fit plots
5. If good, set `N_MAX_FIT = None` for full run

## Session Notes

**2026-01-02**:
- Extended PHOENIX to 3800K - works
- Added Mass/L/R from isochrones - works
- PoWR filtering: 480/~700 models pass isochrone coverage test
- Fitting notebook updated to use manifest - works
- Test run with N=50 successful

---
Last updated: 2026-01-02
Branch: feature/extend-models-add-mass-filter-powr
