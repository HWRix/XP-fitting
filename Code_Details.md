# Code Details: Pipeline, Formats, and Functions

## Physics Rationale (Brief)

The pipeline fits Gaia BP/RP spectra with stellar atmosphere models to determine (Teff, log g, A_V, R_V, Mass, L, R). Key physics:

- **Extinction**: Gordon+2023 R(V)-dependent law
- **Plane-parallel model normalization**: Padova isochrones provide L(Teff, logg) → radius via Stefan-Boltzmann
- **Joint constraint**: χ² includes spectral fit, Gaia parallax, and optional priors

See `Code_Summary.md` for full physics discussion.

## Version History

- **v0**: Basic pipeline with spectral + parallax fitting
- **v1**: Adds catalog enrichment with Andrae+2023 stellar parameters and Wang+2025 3D dust map, plus optional A_V and Teff priors
- **v2**: Vectorized chi2 computation across all models (~5-10× speedup), filters unphysical PoWR models (Teff > 40kK & logg ≥ 4.0), preserves all input catalog columns in output
- **v3**: Hierarchical coarse-to-fine search + gradient-based A_V optimization (~50× speedup), filters unphysical PoWR models (Teff > 40kK & logg ≥ 4.0), preserves all input catalog columns in output
- **v4** (current): Polynomial Spectral Model (PSM) refinement based on Rix et al. (2016, ApJL 826, L25). Full brute-force search followed by local 2D quadratic interpolation in (Teff, logg) space for continuous parameter estimation. Also interpolates auxiliary quantities (logL, Mass) via PSM and derives R_Rsun via Stefan-Boltzmann

---

## Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA PREPARATION                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────┐      ┌──────────────────────────────────────┐    │
│  │ Input FITS catalog   │      │ retrieve_BPRP_spectra-export_2026    │    │
│  │ (source_id, Gmag,    │ ───► │                                      │    │
│  │  parallax, ...)      │      │ Queries Gaia Archive, downloads      │    │
│  └──────────────────────┘      │ XP_SAMPLED spectra per source_id     │    │
│                                └──────────────────────────────────────┘    │
│                                              │                              │
│                                              ▼                              │
│                                ┌──────────────────────────────┐            │
│                                │ ./BPRP_spectra/<source_id>.fits │          │
│                                └──────────────────────────────┘            │
│                                                                             │
│  ┌──────────────────────┐      ┌──────────────────────────────────────┐    │
│  │ Padova_isochrones    │      │ download_stellar_models_phoenix_     │    │
│  │ .fits                │ ───► │ atlas_2026                           │    │
│  └──────────────────────┘      │                                      │    │
│                                │ Downloads PHOENIX (3800-10000K) and  │    │
│                                │ ATLAS (10000-15000K), normalizes to  │    │
│                                │ 10pc. Filters PoWR against isochrone │    │
│                                │ coverage. Extracts Mass, L, R.       │    │
│                                └──────────────────────────────────────┘    │
│                                              │                              │
│                                              ▼                              │
│                          ┌──────────────────────────────────────────┐      │
│                          │ ./stellar_models/atlas-or-phoenix-TTT-GG.txt │  │
│                          │ model_manifest.csv (711 models with props)   │  │
│                          └──────────────────────────────────────────┘      │
│                                                                             │
│  ┌──────────────────────┐                                                  │
│  │ PoWR models          │  (Pre-downloaded, already at 10pc)               │
│  │ ./griddl-gal-ob-vd3- │                                                  │
│  │   line_calib/*.txt   │                                                  │
│  └──────────────────────┘                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FITTING                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026                   │      │
│  │                                                                   │      │
│  │  1. Load models from model_manifest.csv (with Mass, L, R)        │      │
│  │  2. Resample models to Gaia wavelength grid                      │      │
│  │  3. Pre-compute extinction table A(λ)/A_V for R_V grid           │      │
│  │  4. For each star (up to N_MAX_FIT for testing):                 │      │
│  │     - Load observed spectrum                                      │      │
│  │     - Grid search over (model, A_V, R_V)                         │      │
│  │     - Compute χ² = χ²_spec + χ²_parallax                         │      │
│  │     - Refine A_V via parabolic interpolation                     │      │
│  │     - Save best fit with Mass/L/R, generate plot                 │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                       │                                     │
│                                       ▼                                     │
│              ┌────────────────────────┴────────────────────────┐           │
│              │                                                  │           │
│              ▼                                                  ▼           │
│  ┌──────────────────────┐                    ┌──────────────────────┐      │
│  │ Zari_G_bright_fits   │                    │ ./fit_plots/         │      │
│  │ .csv                 │                    │ fit_<source_id>.png  │      │
│  └──────────────────────┘                    └──────────────────────┘      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## File Format Specifications

### Input Catalog (FITS)

Any FITS file with columns:
| Column | Type | Description |
|--------|------|-------------|
| `source_id` | int64 | Gaia DR3 source identifier |
| `Gmag` | float | Gaia G-band magnitude |
| `parallax` | float | Parallax in mas |
| `parallax_error` | float | Parallax uncertainty in mas |

Example files: `sample_stars.fits`, `JMH.SDSSV_massive_subsample.fits`, `Zari.G.lt.12.fits`

### Enriched Catalog (FITS) - v1

Output from `retrieve_BPRP_spectra-export_2026.v1.ipynb`:
| Column | Type | Description |
|--------|------|-------------|
| `source_id` | int64 | Gaia DR3 source identifier |
| `ra`, `dec` | float | Coordinates (deg) |
| `parallax` | float | Parallax (mas) |
| `parallax_error` | float | Parallax uncertainty (mas) |
| `Gmag` | float | Gaia G-band magnitude |
| `Teff_A23` | float | Andrae+2023 Teff (K) |
| `logg_A23` | float | Andrae+2023 log g |
| `MH_A23` | float | Andrae+2023 [M/H] |
| `EBV_W25` | float | Wang+2025 E(B-V) at max distance |
| `EBV_W25_err` | float | Uncertainty on E(B-V) |
| `A_V_W25` | float | Derived A_V = R_V × E(B-V) |
| `A_V_W25_err` | float | Uncertainty on A_V |
| `dist_max_W25` | float | Maximum distance in dust map (kpc) |

Example: `SB1Cands.3_enriched.fits`

### Gaia XP Spectra (FITS)

Location: `./BPRP_spectra/<source_id>.fits`

HDU[1] contains:
| Column | Type | Units | Description |
|--------|------|-------|-------------|
| `wavelength` | float[343] | nm | Wavelength grid (336–1020 nm) |
| `flux` | float[343] | W/m²/nm | Flux density |
| `flux_error` | float[343] | W/m²/nm | Flux uncertainty |

### Stellar Models (ASCII)

Location: `./stellar_models/atlas-or-phoenix-TTT-GG.txt`

Format:
```
# ATLAS-OR-PHOENIX: Teff=10000K, logg=4.0, [M/H]=0.0
# Radius from Padova isochrones, normalized to 10pc
# Wavelength (Angstrom), log10(Flux at 10pc in erg/s/cm2/A)
   3310.0000    -8.613757
   3330.0000    -8.623256
   ...
```

Filename encoding: `atlas-or-phoenix-TTT-GG.txt`
- TTT = Teff/100 (e.g., 100 = 10,000 K, 076 = 7,600 K)
- GG = logg×10 (e.g., 40 = 4.0, 25 = 2.5)

### PoWR Models (ASCII)

Location: `./griddl-gal-ob-vd3-line_calib/gal-ob-vd3_TT-GG_line_calib.txt`

Format: Same 2-column format (wavelength Å, log10 flux)

Filename encoding: `gal-ob-vd3_TT-GG_line_calib.txt`
- TT = Teff/1000 (e.g., 19 = 19,000 K)
- GG = logg×10 (e.g., 40 = 4.0)

### Padova Isochrones (FITS)

Location: `Padova_isochrones.fits`

HDU[1] columns used:
| Column | Description |
|--------|-------------|
| `logTe` | log10(Teff/K) |
| `logg` | Surface gravity |
| `logL` | log10(L/L_sun) |
| `Mass` | Stellar mass (M_sun) |

13,500 isochrone points covering the HR diagram.

### Model Manifest (CSV)

Location: `model_manifest.csv`

Generated by download notebook, contains all 711 valid models:
| Column | Type | Description |
|--------|------|-------------|
| `source` | str | Model family: 'phoenix', 'atlas', 'powr' |
| `Teff` | int | Effective temperature (K) |
| `logg` | float | Surface gravity |
| `logL` | float | log10(L/L_sun) from isochrone |
| `Mass` | float | Stellar mass (M_sun) from isochrone |
| `Mass_std` | float | Mass uncertainty from nearby isochrone points |
| `R_Rsun` | float | Stellar radius (R_sun) from Stefan-Boltzmann |
| `filename` | str | Path to spectrum file |

### Output CSV (v1)

Location: `<input_catalog>_fits.csv` (auto-generated from input catalog name)

| Column | Type | Description |
|--------|------|-------------|
| `source_id` | int64 | Gaia source ID |
| `Teff` | int | Best-fit temperature (K) |
| `logg` | float | Best-fit log g |
| `A_V` | float | Best-fit extinction (mag) |
| `R_V` | float | Best-fit R_V |
| `distance_pc` | float | Fitted distance (pc) |
| `gmag` | float | Input G magnitude |
| `M_G` | float | Absolute G magnitude |
| `chi2_red` | float | Reduced chi-square |
| `model_source` | str | Model family: 'PHOENIX', 'ATLAS', 'PoWR' |
| `model_filename` | str | Path to best-fit model file |
| `logL` | float | log10(L/L_sun) from isochrone |
| `Mass` | float | Stellar mass (M_sun) |
| `Mass_std` | float | Mass uncertainty (M_sun) |
| `R_Rsun` | float | Stellar radius (R_sun) |
| `Teff_A23` | float | Andrae+2023 Teff (K), NaN if not available |
| `A_V_W25` | float | Wang+2025 A_V prior value |
| `used_av_prior` | bool | Whether A_V prior was applied |
| `used_teff_prior` | bool | Whether Teff prior was applied |
| `chi2_av_prior` | float | A_V prior chi2 contribution |
| `chi2_teff_prior` | float | Teff prior chi2 contribution |

---

## Key Functions

### Notebook 1: `retrieve_BPRP_spectra-export_2026.v2.ipynb`

#### Key Features (v2)
- **Minimal input**: Only requires `source_id`, `Gmag`, `parallax`, `parallax_error`
- **Auto-fetch coordinates**: If ra/dec not in input, queries Gaia archive automatically (Cell 20)
- **Caching**: Skips sources that already have downloaded spectra (`SKIP_EXISTING_SPECTRA = True`)
- **Catalog enrichment**: Adds Andrae+2023 parameters and Wang+2025 dust map values to output
- **Coordinate column detection**: Searches for common column names (RA_ICRS, ra, RAJ2000, RAdeg, etc.)

#### `check_bprp_availability(source_ids, batch_size=2000)`
Queries Gaia archive to find which source_ids have XP_SAMPLED spectra.
- **Input**: array of source_ids
- **Output**: array of source_ids with available spectra
- **Method**: ADQL query on `gaiadr3.gaia_source.has_xp_sampled`

#### Download loop
Uses `Gaia.load_data(ids, retrieval_type='XP_SAMPLED')` in batches of 50.

#### Catalog Enrichment (v1)
1. **Andrae+2023 parameters**: Cross-matches with HDF5 file containing Teff_A23, logg_A23, MH_A23
   - Source: `table-1.hdf5` (124M rows)
   - XGBoost stellar parameters from Gaia DR3 XP spectra
2. **Wang+2025 3D dust map**: Queries `dustmaps3d` package for E(B-V) along each sightline
   - Converts to A_V using configurable R_V (default 3.1)
   - Returns A_V_W25, A_V_W25_err, EBV_W25, EBV_W25_err
   - Uses maximum available distance (dist_max_W25) or parallax-derived distance

---

### Notebook 2: `download_stellar_models_phoenix_atlas_2026.v0.ipynb`

#### `find_nearest_isochrone(Teff, logg, iso_logTe, iso_logg, iso_logL, iso_Mass)`
Finds closest match in isochrone table using weighted distance metric.
- **Input**: Target Teff, logg; isochrone arrays (including Mass)
- **Output**: dict with keys: `logL`, `Mass`, `Mass_std`, `logL_std`, `matched_Teff`, `matched_logg`, `min_distance`
- **Weights**: dlogTe_norm=0.1, dlogg_norm=0.5
- **Mass_std**: Standard deviation of Mass values for all isochrone points within tolerance

#### `has_isochrone_coverage(Teff, logg, iso_logTe, iso_logg, teff_tol, logg_tol)`
Checks if a (Teff, logg) point is covered by the isochrone table.
- **Input**: Target Teff, logg; isochrone arrays; tolerances
- **Output**: bool (True if covered)
- **Default tolerances**: 5% in Teff, 0.5 dex in logg
- **Purpose**: Filter PoWR models to physically realizable parameter combinations

#### `calculate_radius_from_luminosity(Teff, logL)`
Derives stellar radius from luminosity and Teff via Stefan-Boltzmann.
- **Input**: Teff (K), logL (log10 L/L_sun)
- **Output**: Radius in cm
- **Formula**: R = sqrt(L / 4π σ Teff⁴)

#### `scale_to_10pc(wavelength, flux_surface, stellar_radius_cm)`
Scales surface flux to observed flux at 10 pc.
- **Formula**: F_10pc = F_surf × (R / d_10pc)²

#### `download_phoenix_spectrum(teff, logg)`
Fetches PHOENIX spectrum from Göttingen server.
- **URL pattern**: `phoenix.astro.physik.uni-goettingen.de/...`
- **Output**: wavelength (Å), flux (erg/s/cm²/Å at surface)

#### `download_atlas_fits(teff, metallicity=0.0)`
Downloads ATLAS FITS from STScI archive.
- **URL pattern**: `archive.stsci.edu/hlsps/reference-atlases/...`
- **Caches**: to `./atlas_fits/`

#### `save_model(wave, flux, teff, logg, model_type)`
Writes normalized model to unified format.

---

### Notebook 3: `fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026.v1.ipynb`

#### Key Features (v1)
- **A_V prior**: Optional chi2 term constraining A_V to Wang+2025 dust map value
- **Teff prior**: Optional chi2 term constraining Teff to Andrae+2023 value (for cool stars)
- **Auto output naming**: Output CSV named from input catalog (e.g., `SB1Cands.3_enriched_fits.csv`)
- **Model filename**: Output includes `model_filename` column for traceability
- **Plots show external values**: A_V(W25) and Teff(A23) displayed whenever available

#### `build_extinction_table(wavelength_aa, R_V_grid)`
Pre-computes A(λ)/A_V for all R_V values on the Gaia grid.
- **Uses**: `dust_extinction.parameter_averages.G23`
- **Output**: dict mapping R_V → A(λ)/A_V array
- **Purpose**: 10-20× speedup in fitting loop

#### `apply_reddening(wavelength_aa, flux, A_V, R_V=3.1)`
Applies Gordon+2023 extinction to a spectrum.
- **Formula**: F_red = F × 10^(-0.4 × A_V × A(λ)/A_V)
- **Uses**: Pre-computed lookup table

#### `calculate_A_G(A_V, R_V=3.1)`
Computes Gaia G-band extinction from A_V.
- **Effective wavelength**: 6420 Å (Gaia DR3 G band)

#### `wavelength_weights(wavelength_nm, blue_region, blue_weight)`
Returns fitting weights giving extra weight to blue region.
- **Default blue region**: 340–480 nm
- **Default blue weight**: 2.0×

#### `fit_single_star(sid, plx, plx_err, gmag, Teff_A23, A_V_W25, A_V_W25_err)` (v1)
Main fitting function for one star with optional priors.
- **Algorithm**:
  1. Load observed spectrum
  2. Determine which priors apply (based on availability and config)
  3. Coarse grid search: A_V = [0, 0.5, 1.0, ..., 6.0]
  4. For each A_V: loop over all models and 10 R_V values
  5. Compute scale factor: s = Σ(f_obs × f_mod / σ²) / Σ(f_mod² / σ²)
  6. Derive distance: d = 10 / sqrt(s)
  7. Compute χ² = χ²_spec + χ²_parallax + w_AV × χ²_AV + w_Teff × χ²_Teff
  8. Refine best A_V via parabolic interpolation (3 iterations)
- **Output**: dict with best-fit parameters, prior chi2 components, and model spectrum

#### `plot_fit(sid, fit_params, Teff_A23, A_V_W25)` (v1)
Generates diagnostic plot for one star.
- **Top panel**: Observed vs. model spectrum (log scale)
- **Bottom panel**: Residuals in σ
- **Includes**: Rayleigh-Jeans reference, fit parameters, A_V(W25) and Teff(A23) when available

---

## Configuration Parameters

### Model Grid (Notebook 2)
```python
PHOENIX_TEFF_VALUES = [3800, 4000, ..., 10000]  # 32 values (3800-10000K)
ATLAS_TEFF_VALUES = [10000, 10500, ..., 15000]  # 7 values (overlaps at 10000K)
LOGG_VALUES = [2.0, 2.5, 3.0, 3.5, 4.0, 4.5]   # 6 values
METALLICITY = 0.0  # Solar
LAMBDA_MIN, LAMBDA_MAX = 3300, 10500  # Å (Gaia coverage)

# PoWR filtering tolerances
ISOCHRONE_TEFF_TOLERANCE = 0.05  # 5% in Teff
ISOCHRONE_LOGG_TOLERANCE = 0.5   # 0.5 dex in logg
```

### Fitting Grid (Notebook 3, v1)
```python
A_V_GRID = np.arange(0.0, 7.1, 0.1)  # 71 values (refined search)
R_V_GRID = np.array([2.3, 2.5, 2.8, 3.1, 3.4, 3.7, 4.0, 4.5, 5.0, 5.5])  # 10 values, valid for G23 [2.3-5.6]
WAVELENGTH_FIT_MIN = 340.0   # nm
WAVELENGTH_FIT_MAX = 900.0   # nm
SYSTEMATIC_FLOOR = 0.03      # 3% systematic error floor
BLUE_WEIGHT_REGION = (340.0, 480.0)  # nm
BLUE_WEIGHT_FACTOR = 2.0

# Prior configuration (v1)
USE_AV_PRIOR = True         # Use Wang+2025 A_V as prior
AV_PRIOR_WEIGHT = 10.0      # Weight for A_V prior chi2 term
USE_TEFF_PRIOR = False      # Use Andrae+2023 Teff as prior (for cool stars)
TEFF_PRIOR_WEIGHT = 1.0     # Weight for Teff prior
TEFF_PRIOR_TEFF_MAX = 7500  # Only apply Teff prior if Teff_A23 < this
TEFF_PRIOR_SIGMA = 500      # K, assumed uncertainty for A23 Teff

# Testing/development
N_MAX_FIT = 100      # Set to None to fit all stars, or int for testing
USE_MANIFEST = True  # Load models from model_manifest.csv (includes Mass/L/R)
```

### Chi-squared Formulation (v1+)
```
chi2_total = chi2_spectral + chi2_parallax + w_AV * chi2_AV_prior + w_Teff * chi2_Teff_prior

where:
  chi2_AV_prior = ((A_V_fit - A_V_W25) / A_V_W25_err)^2  [if USE_AV_PRIOR and A_V_W25 available]
  chi2_Teff_prior = ((Teff_model - Teff_A23) / TEFF_PRIOR_SIGMA)^2  [if USE_TEFF_PRIOR and Teff_A23 < 7500K]
```

### Model Filtering (v2, v3)

PoWR models outside Padova isochrone coverage are excluded:
```python
# Exclude unphysical parameter combinations (hot + high gravity)
TEFF_LOGG_CUT_TEFF = 40000  # K
TEFF_LOGG_CUT_LOGG = 4.0    # dex

exclude_mask = (Teff > TEFF_LOGG_CUT_TEFF) & (logg >= TEFF_LOGG_CUT_LOGG)
```

**Rationale**: Very hot stars (Teff > 40kK) with high surface gravity (logg ≥ 4.0) don't exist - real stars at these temperatures are massive supergiants with lower logg. The Padova isochrones confirm no evolutionary tracks pass through this region.

### Polynomial Spectral Model (v4)

Based on Rix et al. (2016, ApJL 826, L25), this approach uses local quadratic interpolation of spectra in parameter space.

**Algorithm:**
1. **Full brute-force search**: Evaluate all ~470 models to find best discrete (Teff*, logg*, A_V*, R_V*)
2. **Build local PSM**: Fit 2D quadratic to intrinsic spectra AND auxiliary quantities (logL, Mass) in 3×3 neighborhood around best model
3. **Continuous optimization**: Minimize χ² over (Teff, logg, A_V, R_V) using PSM + analytic extinction
4. **Interpolate auxiliary quantities**: Use PSM coefficients to get logL and Mass at optimized (Teff, logg), derive R_Rsun via Stefan-Boltzmann

**Quadratic spectral model (per wavelength):**
```python
log(f_intrinsic(λ; T, g)) ≈ a₀ + a₁T̃ + a₂g̃ + a₃T̃² + a₄g̃² + a₅T̃g̃

# where T̃, g̃ are normalized coordinates in [-1, 1] within local patch
# 6 coefficients per wavelength, fit by least-squares to 9 neighbor models
```

**Auxiliary quantity interpolation:**
```python
# Same 2D quadratic form for logL and Mass:
logL(T̃, g̃) ≈ b₀ + b₁T̃ + b₂g̃ + b₃T̃² + b₄g̃² + b₅T̃g̃
Mass(T̃, g̃) ≈ c₀ + c₁T̃ + c₂g̃ + c₃T̃² + c₄g̃² + c₅T̃g̃

# R_Rsun derived via Stefan-Boltzmann (not interpolated directly):
R = sqrt(L / (4π σ Teff⁴))
```

**Key advantage**: No models are skipped in the initial search (unlike v3's coarse pass), while still obtaining continuous parameter estimates. Extinction is applied analytically via Gordon+2023, so only the 2D (Teff, logg) interpolation is needed. All output quantities (Teff, logg, logL, Mass, R_Rsun) vary continuously.

**Implementation details:**
- Neighbors found within the same model source family (PHOENIX, ATLAS, or PoWR), deduplicated by (Teff, logg)
- Edge cases handled: if fewer than 4 neighbors available, PSM refinement is skipped
- L-BFGS-B optimizer with bounds: Teff/logg constrained to local patch, A_V ∈ [0, 6], R_V ∈ [2.3, 5.6]

**Output includes both discrete and refined values:**
- `Teff_fit`, `logg_fit`, `A_V_fit`, `R_V_fit`: PSM-refined (or discrete if PSM failed)
- `logL_fit`, `Mass_fit`, `R_Rsun_fit`: PSM-interpolated auxiliary quantities
- `Teff_grid`, `logg_grid`, `A_V_grid`, `R_V_grid`: Best discrete grid values
- `logL_grid`, `Mass_grid`, `R_Rsun_grid`: Discrete best-model auxiliary quantities
- `psm_refined`: Boolean flag indicating whether PSM optimization succeeded
- `n_psm_neighbors`: Number of models used for PSM (typically 9 for interior, 6 for edges)

### Hierarchical Search (v3)

Two-pass fitting algorithm for ~50× speedup:
```python
# Pass 1: Coarse search
COARSE_MODEL_STEP = 3       # Every 3rd model
COARSE_AV_STEP = 0.5        # A_V step
R_V_GRID_COARSE = [2.5, 3.1, 4.0]  # 3 R_V values
N_TOP_MODELS = 20           # Keep top 20 models

# Pass 2: Fine search on top models
R_V_GRID_FULL = [2.3, 2.5, 2.8, 3.1, 3.4, 3.7, 4.0, 4.5, 5.0, 5.5]  # 10 R_V values
# Use scipy.optimize.minimize_scalar for A_V (gradient-based, ~10 evals instead of 71)
```

---

## Dependencies

```
numpy
astropy
astroquery (Gaia module)
scipy
matplotlib
pandas
tqdm
dust_extinction (>= 1.2, for G23)
h5py              # For reading Andrae+2023 HDF5 file
dustmaps3d        # For Wang+2025 3D dust map queries
```

Install: `pip install numpy astropy astroquery scipy matplotlib pandas tqdm dust_extinction h5py dustmaps3d`

### External Data Files
- `table-1.hdf5`: Andrae+2023 stellar parameters (124M rows, ~3GB)
- Wang+2025 dust map: Downloaded automatically by `dustmaps3d` on first use
