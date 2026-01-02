# Hot Star PoWR Fitting with Automatic Gaia Spectrum Download

## Overview

This notebook automatically fits PoWR stellar atmosphere models to hot stars' Gaia BP/RP spectra, with built-in functionality to download missing spectra from the Gaia archive.

## Input Requirements

Your `hot_star_sample.fits` file must contain at least these columns:
- **source_id** (int64): Gaia DR3 source identifier
- **parallax** (float): Parallax in mas (can be NaN)
- **parallax_error** (float): Parallax uncertainty in mas (can be NaN)
- **phot_g_mean_mag** (float): Gaia G magnitude (or 'Gmag')

## Workflow

The notebook performs the following steps for each source:

### 1. Spectrum Acquisition
- **Check local**: Looks for spectrum in `./hot_star_BPRP_spectra/`
- **Auto-download**: If not found, queries Gaia DR3 with your credentials and downloads it
- **Skip**: If spectrum unavailable in Gaia, skips that source

### 2. Model Fitting
- Loads PoWR models from `./griddl-gal-ob-vd3-line_calib/`
- (Optional) Loads ATLAS models from `./atlas9_models/`
- Resamples models to exact Gaia wavelength grid
- Performs grid search over:
  - Teff, log g (from model grid)
  - A_V: 0.0-5.0 mag in 0.1 mag steps
  - R_V: [2.5, 3.1, 4.0, 5.0]
- Fits scaling factor (distance) via weighted least-squares
- Includes parallax constraint in χ² if available

### 3. Output Generation
- **CSV file**: `hot_star_powr_fits.csv` with all fit parameters
- **Diagnostic plots**: `./fit_plots/fit_{source_id}.png` for each star

## Required Directory Structure

```
./
├── hot_star_sample.fits              # Your input catalog
├── griddl-gal-ob-vd3-line_calib/     # PoWR models (required)
├── atlas9_models/                     # ATLAS models (optional)
├── hot_star_BPRP_spectra/            # Downloaded spectra (auto-created)
├── fit_plots/                         # Output plots (auto-created)
└── fit_hot_stars_with_auto_download.ipynb
```

## Output Files

### CSV: `hot_star_powr_fits.csv`
Contains for each successfully fitted star:
- source_id, name (generic)
- Teff, logg (stellar parameters)
- A_V, R_V (extinction)
- distance_pc (spectroscopic distance)
- gmag, M_G (apparent and absolute magnitude)
- chi2_reduced, chi2_spectrum, chi2_parallax (fit quality)
- dof (degrees of freedom)
- model_type (PoWR or ATLAS)

### Plots: `./fit_plots/fit_{source_id}.png`
Each plot shows:
- Top panel: Observed spectrum (black), best-fit model (red), unreddened model (blue dashed)
- Bottom panel: Fit residuals in units of σ
- Title: Best-fit parameters and χ²

## Gaia Credentials

The notebook uses hardcoded Gaia archive credentials:
- Username: `hrix01`
- Password: `Whynot2024?`

These are used to download BP/RP spectra via the Gaia DataLink service.

## Model Grid

The code fits over:
- **~120 PoWR models** (different Teff and log g combinations)
- **51 A_V values** (0.0 to 5.0 in 0.1 mag steps)
- **4 R_V values** (2.5, 3.1, 4.0, 5.0)
- **Total: ~24,000 model evaluations per star**

## Key Features

1. **Automatic spectrum retrieval**: No manual downloads needed
2. **Parallax constraints**: Uses Gaia parallax in fitting when available
3. **CCM89 extinction**: Cardelli, Clayton & Mathis (1989) reddening law
4. **Generic naming**: Stars named as `Source_{source_id}`
5. **Robust error handling**: Skips sources without spectra gracefully

## Execution Time

Approximate timing (depends on your system):
- Model loading: ~30 seconds
- Per-star fitting: ~10-30 seconds
- For 100 stars: ~20-50 minutes

## Notes

- The code assumes all necessary model files are present in the specified directories
- Spectra are downloaded only once and cached locally
- If a source has no BP/RP spectrum in Gaia, it will be skipped
- The code handles NaN values in parallax/magnitude columns gracefully
