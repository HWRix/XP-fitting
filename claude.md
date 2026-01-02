# Claude Code Context: Hot Star XP Spectral Fitting

## Project Purpose

Determine stellar parameters (Teff, log g, A_V, R_V) for stars by fitting Gaia DR3 BP/RP spectra with synthetic stellar atmosphere models. The fitting jointly constrains spectral shape, extinction, and distance (via parallax).

## Current State

### Git Status
- **Repository**: Initialized, tracking main notebooks and documentation
- **Main branch**: `main` — original working v0 code
- **Feature branch**: `feature/extend-models-add-mass-filter-powr` — current development

### Recent Changes (on feature branch)
1. **Extended PHOENIX models**: 3800-10000K (was 7600-10000K)
2. **Added stellar mass**: Extracted from Padova isochrones alongside L and R
3. **PoWR filtering**: Removes models without isochrone coverage (5% Teff, 0.5 dex logg tolerance)
4. **Model manifest**: New CSV output with all valid models and their properties

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
│    - Filters PoWR models against isochrone coverage             │
│    - Output: ./stellar_models/*.txt, model_manifest.csv         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026.v0.ipynb      │
│    - Loads all models (PoWR + ATLAS + PHOENIX)                  │
│    - Fits each star: grid search over (Teff, logg, A_V, R_V)    │
│    - Uses Gordon+2023 extinction law                            │
│    - Joint χ² with parallax constraint                          │
│    - Output: CSV results, diagnostic plots                      │
└─────────────────────────────────────────────────────────────────┘
```

### Key Data Files
- `Padova_isochrones.fits` — 13,500 isochrone points for (Teff, logg) → (L, M, R)
- `./griddl-gal-ob-vd3-line_calib/` — 244 pre-downloaded PoWR models (15-50 kK)
- `./stellar_models/` — ATLAS+PHOENIX models normalized to 10pc
- `./BPRP_spectra/` — Downloaded Gaia XP spectra

### Model Coverage (after current changes)
| Source | Teff Range | Notes |
|--------|------------|-------|
| PHOENIX | 3800-10000K | Plane-parallel, normalized via isochrones |
| ATLAS | 10000-15000K | Plane-parallel, normalized via isochrones |
| PoWR | 15000-50000K | Spherical, filtered to isochrone coverage |

## Documentation Files
- `Code_Summary.md` — Physics rationale (extinction, isochrones, joint fitting)
- `Code_Details.md` — Pipeline, file formats, function documentation
- `Code_issues_problems_improvements.md` — Known issues and roadmap

## Where We May Want to Go

### Immediate (this session or next)
- [ ] Test the modified download notebook
- [ ] Verify PHOENIX downloads work for 3800-7600K range
- [ ] Check PoWR filtering results (how many rejected?)
- [ ] Add R_std (radius uncertainty) from logL_std

### Short-term Improvements
- [ ] **Expand R_V grid**: Currently [2.5, 3.1, 3.7] — too coarse
  - Recommend: [2.3, 2.5, 2.7, 2.9, 3.1, 3.3, 3.5, 3.7, 4.0, 4.5, 5.0]
- [ ] **Add model_type to fitting output**: Track which model family gave best fit
- [ ] **Add quality flags**: boundary warnings, degeneracy detection
- [ ] **Update fitting notebook**: Load model_manifest.csv instead of glob patterns

### Medium-term Improvements
- [ ] **Parallelization**: Fitting loop is serial, could use multiprocessing
- [ ] **MCMC uncertainties**: Currently only best-fit, no posteriors
- [ ] **Validation**: Compare to APOGEE/GALAH/benchmark stars

### Longer-term / Optional
- [ ] **Metallicity dimension**: Add [M/H] ≠ 0 model grids
- [ ] **Binary handling**: Fit composite spectra
- [ ] **CLI tool**: Convert notebooks to standalone scripts

## Git Workflow

```bash
# See current branch
git branch

# See recent commits
git log --oneline -5

# Compare to main
git diff main..HEAD --stat

# If changes work, merge to main:
git checkout main
git merge feature/extend-models-add-mass-filter-powr

# If changes break, discard and return to main:
git checkout main

# Start new feature:
git checkout -b feature/new-feature-name
```

## Testing Protocol

1. Run notebook cells in order
2. Check for errors in PHOENIX downloads (network dependent)
3. Verify `model_manifest.csv` is created with expected columns
4. Spot-check a few models: do Teff/logg/Mass values make sense?
5. Report results back to Claude for iteration

## Session Notes

*Add notes here during development sessions*

---
Last updated: 2026-01-02
Branch: feature/extend-models-add-mass-filter-powr
