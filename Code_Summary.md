# Code Summary: Stellar Parameter Fitting from Gaia BP/RP Spectra

## Scientific Goal

Determine stellar parameters (Teff, log g, A_V, Mass, L, R) for stars observed by Gaia DR3 by fitting their low-resolution BP/RP spectra (R~50-100) with synthetic stellar atmosphere models. The fitting jointly constrains:

1. **Spectral shape** — sensitive to Teff and log g
2. **Extinction** — modifies the spectral slope, especially in the blue
3. **Distance** — via Gaia parallax, which constrains the absolute flux level
4. **Stellar properties** — Mass, Luminosity, Radius from isochrone matching

This joint spectro-photometric-astrometric approach breaks degeneracies that would plague photometry-only or spectroscopy-only methods.

---

## Physics Rationale

### Why Three Model Grids?

| Model | Type | Teff Range | N_models | Strengths |
|-------|------|------------|----------|-----------|
| **PHOENIX** | Plane-parallel | 3,800–10,000 K | 192 | Molecular opacities for cool stars, extended to K-type |
| **ATLAS** | Plane-parallel, LTE | 10,000–15,000 K | 39 | Well-tested, extensive line lists for B/A stars |
| **PoWR** | Spherical, NLTE, wind | 15,000–56,000 K | 480 | Proper wind treatment, NLTE for O/early-B stars |

**Total: 711 models** covering 3,800–56,000 K.

### The Isochrone Normalization Problem

PoWR models are computed for spherical stellar atmospheres and output flux at a reference distance (10 pc). They self-consistently include the stellar radius.

ATLAS and PHOENIX are **plane-parallel** — they output surface flux (erg/s/cm²/Å) with no information about stellar radius. To compare with observations at a known distance, we need:

```
F_observed = F_surface × (R_star / d)²
```

**Solution**: Use Padova isochrones to map (Teff, log g) → (L, M, R):

```
Isochrone lookup: (Teff, log g) → logL, Mass
Stefan-Boltzmann: L = 4π R² σ T_eff⁴  →  R = √(L / 4π σ T_eff⁴)
```

This assumes the star lies on a theoretical isochrone (valid for single, non-peculiar stars). The isochrone table (`Padova_isochrones.fits`) contains 13,500 points spanning log(Teff) = 3.18–5.35 and log g = -2.2 to 6.2.

### PoWR Model Filtering

Not all PoWR grid points correspond to physical stellar evolution tracks. Very hot temperatures with high log g (e.g., 50,000 K at log g = 4.5) don't exist on isochrones.

**Filter criterion**: Keep only PoWR models where an isochrone point exists within:
- 5% in Teff
- 0.5 dex in log g

Result: 480 of ~700 PoWR models pass the filter.

### Extinction: Gordon et al. (2023)

The code uses the **Gordon et al. (2023)** R(V)-dependent extinction law (ApJ, 950, 86), implemented via the `dust_extinction` Python package. This is the current state-of-the-art, superseding CCM89 and Fitzpatrick99.

Key features:
- Valid for R_V = 2.3–5.6
- Wavelength coverage: 912 Å – 32 μm (fully covers Gaia BP/RP: 330–1050 nm)
- Properly handles the 2175 Å bump and UV rise

The extinction is applied as:
```
F_reddened(λ) = F_intrinsic(λ) × 10^(-0.4 × A_V × A(λ)/A(V))
```

where A(λ)/A(V) depends on R_V and is pre-computed on the Gaia wavelength grid for speed.

**Current R_V grid**: [2.5, 3.1, 3.7] — intentionally coarse for speed; expansion planned.

### Joint Chi-Square with Parallax Constraint

The fitting minimizes:

```
χ² = χ²_spectral + χ²_parallax
```

where:
- **χ²_spectral** = Σ_λ w(λ) × [(F_obs - s×F_model) / σ]²
- **χ²_parallax** = [(1000/d_fit - π_Gaia) / σ_π]²

The wavelength weights w(λ) give 2× weight to the blue region (340–480 nm) where extinction effects are strongest and Teff sensitivity is highest.

**The parallax term is crucial**: it prevents the fitter from finding degenerate solutions where a hotter, more distant star could mimic a cooler, closer one. This joint constraint is a key strength of the method.

---

## Model Coverage Summary

```
Temperature (K):  3,800 -------- 10,000 -------- 15,000 -------- 56,000
                    │               │               │               │
                    │   [PHOENIX]   │    [ATLAS]    │    [PoWR]     │
                    │   192 models  │   39 models   │  480 models   │
                    │   3.8-10 kK   │   10-15 kK    │  15-56 kK     │
                    └───────────────┴───────────────┴───────────────┘
                              Complete coverage: 711 models
```

All models are stored in a unified format:
- Spectra: 2-column ASCII (wavelength Å, log₁₀ flux at 10pc)
- Properties: `model_manifest.csv` with Teff, logg, logL, Mass, Mass_std, R_Rsun

---

## Output Products

1. **Model manifest** (`model_manifest.csv`): Properties of all 711 valid models
   - source, Teff, logg, logL, Mass, Mass_std, R_Rsun, filename

2. **Fit results CSV** (`Zari_G_bright_fits.csv`): Best-fit parameters per star
   - source_id, Teff, logg, A_V, R_V, distance_pc, gmag, M_G, chi2_red
   - model_source, logL, Mass, Mass_std, R_Rsun

3. **Diagnostic plots** (`fit_plots/fit_<source_id>.png`):
   - Top panel: observed vs. model spectrum
   - Bottom panel: residuals in units of σ
   - Rayleigh-Jeans reference line for visual sanity check

---

## References

- Gordon, K. D., et al. 2023, ApJ, 950, 86 (G23 extinction law)
- Padova isochrones: Bressan et al. 2012, MNRAS, 427, 127
- PoWR models: Sander et al. 2015, A&A, 577, A13
- ATLAS9: Castelli & Kurucz 2003
- PHOENIX: Husser et al. 2013, A&A, 553, A6
