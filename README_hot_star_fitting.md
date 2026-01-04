# Hot Star Fitting: Gaia BP/RP Spectra with Stellar Atmosphere Models

## Overview

This pipeline fits stellar atmosphere models to Gaia DR3 BP/RP spectra to determine stellar parameters (Teff, log g, A_V, R_V, Mass, L, R). The fitting jointly constrains spectral shape, extinction, and distance via Gaia parallax.

## Quick Start

### 1. Prepare Input Catalog
Minimal FITS file with columns: `source_id`, `Gmag`, `parallax`, `parallax_error`

### 2. Download Spectra & Enrich Catalog
Run `retrieve_BPRP_spectra-export_2026.v2.ipynb`:
- Auto-fetches ra/dec from Gaia if not in input
- Downloads XP spectra to `./BPRP_spectra/`
- Enriches with Andrae+2023 (Teff, logg) and Wang+2025 (A_V)
- Outputs `*_enriched.fits`

### 3. Prepare Model Library (one-time)
Run `download_stellar_models_phoenix_atlas_2026.v0.ipynb` to:
- Download PHOENIX (3800-10000K) and ATLAS (10000-15000K) models
- Normalize to 10pc flux using Padova isochrones
- Filter PoWR models (15000-56000K) against isochrone coverage
- Generate `model_manifest.csv` with all model properties

### 4. Run Fitting
Run `fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026.v4.ipynb`:
- Full brute-force search over all ~470 models
- PSM (Polynomial Spectral Model) refinement for continuous parameters
- Gordon+2023 R(V)-dependent extinction law
- Joint χ² with parallax constraint

## Model Grid

| Model | Type | Teff Range | N_models |
|-------|------|------------|----------|
| **PHOENIX** | Plane-parallel | 3,800–10,000 K | 192 |
| **ATLAS** | Plane-parallel, LTE | 10,000–15,000 K | 39 |
| **PoWR** | Spherical, NLTE, wind | 15,000–56,000 K | 240 |

**Total: ~471 models** covering 3,800–56,000 K with log g = 2.0–4.5.

## Fitting Algorithm (v4)

1. **Brute-force search**: Evaluate all models to find best discrete (Teff, logg, A_V, R_V)
2. **PSM refinement**: Build local 2D quadratic model in 3×3 neighborhood
3. **Continuous optimization**: L-BFGS-B over (Teff, logg, A_V, R_V)
4. **Auxiliary interpolation**: logL and Mass via PSM, R via Stefan-Boltzmann

Based on Rix et al. (2016, ApJL 826, L25).

## Output

CSV with columns:
- `Teff_fit`, `logg_fit`, `A_V_fit`, `R_V_fit`: PSM-refined parameters
- `logL_fit`, `Mass_fit`, `R_Rsun_fit`: PSM-interpolated stellar properties
- `*_grid`: Discrete best-model values for comparison
- `distance_pc`, `chi2_red`, `psm_refined`, `n_psm_neighbors`

Diagnostic plots in `./fit_plots/`.

## Directory Structure

```
./
├── retrieve_BPRP_spectra-export_2026.v2.ipynb   # Step 2: Download + enrich
├── download_stellar_models_phoenix_atlas_2026.v0.ipynb  # Step 3: Model prep
├── fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026.v4.ipynb  # Step 4: Fit
├── model_manifest.csv        # Model properties (Teff, logg, Mass, L, R)
├── Padova_isochrones.fits    # For model normalization
├── stellar_models/           # PHOENIX + ATLAS models
├── griddl-gal-ob-vd3-line_calib/  # PoWR models
├── BPRP_spectra/             # Downloaded Gaia XP spectra
└── fit_plots/                # Output diagnostic plots
```

## References

- Gordon, K. D., et al. 2023, ApJ, 950, 86 (G23 extinction law)
- Rix, H.-W., et al. 2016, ApJL, 826, L25 (Polynomial Spectral Models)
- Padova isochrones: Bressan et al. 2012, MNRAS, 427, 127
- PoWR: Sander et al. 2015, A&A, 577, A13
- ATLAS9: Castelli & Kurucz 2003
- PHOENIX: Husser et al. 2013, A&A, 553, A6
