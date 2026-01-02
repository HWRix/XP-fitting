# Code Details: Pipeline, Formats, and Functions

## Physics Rationale (Brief)

The pipeline fits Gaia BP/RP spectra with stellar atmosphere models to determine (Teff, log g, A_V, R_V). Key physics:

- **Extinction**: Gordon+2023 R(V)-dependent law
- **Plane-parallel model normalization**: Padova isochrones provide L(Teff, logg) → radius via Stefan-Boltzmann
- **Joint constraint**: χ² includes both spectral fit and Gaia parallax

See `Code_Summary.md` for full physics discussion.

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
│                                │ Downloads PHOENIX (Göttingen) and    │    │
│                                │ ATLAS (STScI), normalizes to 10pc    │    │
│                                └──────────────────────────────────────┘    │
│                                              │                              │
│                                              ▼                              │
│                                ┌──────────────────────────────┐            │
│                                │ ./stellar_models/atlas-or-   │            │
│                                │    phoenix-TTT-GG.txt        │            │
│                                └──────────────────────────────┘            │
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
│  │  1. Load all models (PoWR + ATLAS/PHOENIX)                       │      │
│  │  2. Resample models to Gaia wavelength grid                      │      │
│  │  3. Pre-compute extinction table A(λ)/A_V for R_V grid           │      │
│  │  4. For each star:                                               │      │
│  │     - Load observed spectrum                                      │      │
│  │     - Grid search over (model, A_V, R_V)                         │      │
│  │     - Compute χ² = χ²_spec + χ²_parallax                         │      │
│  │     - Refine A_V via parabolic interpolation                     │      │
│  │     - Save best fit, generate plot                               │      │
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

13,500 isochrone points covering the HR diagram.

### Output CSV

Location: `Zari_G_bright_fits.csv`

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

---

## Key Functions

### Notebook 1: `retrieve_BPRP_spectra-export_2026.v0.ipynb`

#### `check_bprp_availability(source_ids, batch_size=2000)`
Queries Gaia archive to find which source_ids have XP_SAMPLED spectra.
- **Input**: array of source_ids
- **Output**: array of source_ids with available spectra
- **Method**: ADQL query on `gaiadr3.gaia_source.has_xp_sampled`

#### Download loop
Uses `Gaia.load_data(ids, retrieval_type='XP_SAMPLED')` in batches of 50.

---

### Notebook 2: `download_stellar_models_phoenix_atlas_2026.v0.ipynb`

#### `find_nearest_isochrone(Teff, logg, iso_logTe, iso_logg, iso_logL)`
Finds closest match in isochrone table using weighted distance metric.
- **Input**: Target Teff, logg; isochrone arrays
- **Output**: (logL, matched_Teff, matched_logg)
- **Weights**: dlogTe_norm=0.1, dlogg_norm=0.5

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

### Notebook 3: `fit_PoWRmodels_to_BPRP_hot_stars_Gordon24_2026.v0.ipynb`

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

#### `fit_single_star(sid, plx, plx_err, gmag)`
Main fitting function for one star.
- **Algorithm**:
  1. Load observed spectrum
  2. Coarse grid search: A_V = [0, 0.5, 1.0, ..., 6.0]
  3. For each A_V: loop over all models and R_V values
  4. Compute scale factor: s = Σ(f_obs × f_mod / σ²) / Σ(f_mod² / σ²)
  5. Derive distance: d = 10 / sqrt(s)
  6. Compute χ² = χ²_spec + χ²_parallax
  7. Refine best A_V via parabolic interpolation (3 iterations)
- **Output**: dict with best-fit parameters and model spectrum

#### `plot_fit(sid, fit_params)`
Generates diagnostic plot for one star.
- **Top panel**: Observed vs. model spectrum (log scale)
- **Bottom panel**: Residuals in σ
- **Includes**: Rayleigh-Jeans reference, fit parameters in legend

---

## Configuration Parameters

### Model Grid (Notebook 2)
```python
PHOENIX_TEFF_VALUES = [7600, 7800, ..., 10000]  # 13 values
ATLAS_TEFF_VALUES = [10000, 10500, ..., 15000]  # 7 values
LOGG_VALUES = [2.0, 2.5, 3.0, 3.5, 4.0, 4.5]   # 6 values
METALLICITY = 0.0  # Solar
LAMBDA_MIN, LAMBDA_MAX = 3300, 10500  # Å (Gaia coverage)
```

### Fitting Grid (Notebook 3)
```python
A_V_GRID = np.arange(0.0, 7.1, 0.1)  # 71 values (refined search)
R_V_GRID = np.array([2.5, 3.1, 3.7]) # 3 values (coarse)
WAVELENGTH_FIT_MIN = 340.0   # nm
WAVELENGTH_FIT_MAX = 900.0   # nm
SYSTEMATIC_FLOOR = 0.03      # 3% systematic error floor
BLUE_WEIGHT_REGION = (340.0, 480.0)  # nm
BLUE_WEIGHT_FACTOR = 2.0
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
```

Install: `pip install numpy astropy astroquery scipy matplotlib pandas tqdm dust_extinction`
