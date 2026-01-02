# Gaia BP/RP (XP_SAMPLED) Spectra Notebook

This notebook downloads and plots Gaia BP/RP spectra for your IACOB O-stars sample.

## Key Features

### 1. **Checks Availability**
- Queries `gaiadr3.gaia_source` for the `has_xp_sampled` flag
- BP/RP spectra have much higher availability (~34 million sources) than RVS (~1 million)

### 2. **Downloads XP_SAMPLED Spectra**
- Uses DataLink protocol with `retrieval_type='XP_SAMPLED'`
- Pre-sampled format (no coefficient conversion needed)
- Saves to `./IACOB_Ostar_BPRP_spectra/`

### 3. **Creates Combined BP+RP Plots**
- BP (blue) and RP (red) on same plot
- Full wavelength range: **336-1020 nm**
- Saves to `./IACOB_Ostar_BPRP_plots/`

## BP/RP Spectrum Details

### Wavelength Coverage
- **BP (Blue Photometer)**: ~336-680 nm
- **RP (Red Photometer)**: ~640-1020 nm  
- **Overlap region**: ~640-680 nm
- **Sampling**: ~2 nm steps (~343 wavelength points total)

### Data Format
The XP_SAMPLED format provides:
- `wavelength`: Wavelength array (nm)
- `flux`: Flux values (W m⁻² nm⁻¹)
- `flux_error`: Flux uncertainties

### Flux Units
BP/RP spectra are **absolute-calibrated** in physical units (W m⁻² nm⁻¹), unlike RVS which is normalized.

## Comparison: RVS vs BP/RP

| Feature | RVS | BP/RP (XP_SAMPLED) |
|---------|-----|-------------------|
| Wavelength range | 846-870 nm | 336-1020 nm |
| Resolution | High (~R=11,500) | Low (~R=50-100) |
| Availability | ~1M sources | ~34M sources |
| Calibration | Normalized | Absolute flux |
| Best for | Radial velocities, Ca II triplet | Full SED, photometry, classification |

## Output Files

```
./IACOB_Ostar_BPRP_spectra/
    ├── bprp_spectrum_2059853915630134912.fits
    └── ...

./IACOB_Ostar_BPRP_plots/
    ├── bprp_spectrum_2059853915630134912.png
    └── ...

bprp_spectrum_summary.csv
```

## Usage Notes

1. **High availability**: Almost all Gaia sources have BP/RP spectra
2. **Low resolution**: Good for overall SED shape, not line profiles
3. **Absolute flux**: Can be used for photometric calibration
4. **Combined plot**: Shows full optical spectrum at a glance

## For Hot Stars (O-type)

BP/RP spectra of O-stars will show:
- Strong UV continuum (if not too reddened)
- Broad Balmer absorption features
- Overall energy distribution
- Effects of interstellar extinction

The low resolution means individual spectral lines are not resolved, but the overall SED shape is very useful!

## References

- [Gaia DR3 BP/RP Documentation](https://gea.esac.esa.int/archive/documentation/GDR3/Data_processing/chap_cu5pho/)
- [De Angeli et al. (2023) - BP/RP Processing](https://www.aanda.org/articles/aa/abs/2023/06/aa45764-22/aa45764-22.html)
- [Montegriffo et al. (2023) - Flux Calibration](https://www.aanda.org/articles/aa/abs/2023/06/aa45916-22/aa45916-22.html)
