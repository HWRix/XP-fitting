# Gaia BP/RP Spectra for IACOB B-stars

This notebook downloads and plots Gaia BP/RP spectra for the IACOB B-stars sample.

## Key Differences from O-stars Notebook

### 1. **Input File**
- File: `IACOB_Bstars_Gaiamatch.fits`
- 149 B-type stars
- Columns: `Name` (HD catalog), `SpType`, `source_id`
- **No Teff or logg** in this dataset

### 2. **Spectral Type Sorting**
Sources are sorted by spectral type (B0 → B9):
- Extracts numeric type from SpType string (e.g., B0Ib → 0.0, B2.5V → 2.5)
- Ignores luminosity class and subtype for sorting
- Files numbered: `001_`, `002_`, etc. (earliest/hottest type first)

### 3. **Plot Annotations**
- Shows **SpType only** (no Teff/logg)
- Text box in upper-right corner
- Format: `SpType: B0Ib`

### 4. **File Naming**
All files prefixed with spectral type rank:
- `001_bstar_bprp_spectrum_2178808979101816192.fits` (earliest type)
- `001_bstar_bprp_spectrum_2178808979101816192.png`
- `002_bstar_bprp_spectrum_4069469560083491712.fits`
- etc.

### 5. **Directories**
- Spectra: `./IACOB_Bstar_BPRP_spectra/`
- Plots: `./IACOB_Bstar_BPRP_plots/`

## Spectral Type Parsing

The sorting function handles various formats:
- `B0V` → 0.0
- `B0.2Ia` → 0.2
- `B1III` → 1.0
- `B2.5V` → 2.5
- `B9Ib` → 9.0

## Expected Spectral Features for B-stars

B-type stars have BP/RP spectra showing:
- **Strong Balmer lines** (H-alpha, H-beta, H-gamma)
- **He I absorption** (especially B0-B2)
- **Blue/UV continuum** (less extreme than O-stars)
- **Temperature sequence**:
  - B0 (~30,000 K): Very strong He I lines
  - B5 (~15,000 K): Maximum Balmer line strength
  - B9 (~10,000 K): Weak He I, strong H lines

## B-star Temperature Range

| SpType | Teff (K) | Notes |
|--------|----------|-------|
| B0 | ~30,000 | Hottest B-stars |
| B2 | ~20,000 | Strong He I |
| B5 | ~15,000 | Maximum H line strength |
| B8 | ~12,000 | Approaching A-type |
| B9 | ~10,000 | Coolest B-stars |

## Output Files

```
./IACOB_Bstar_BPRP_spectra/
    ├── 001_bstar_bprp_spectrum_{source_id}.fits
    ├── 002_bstar_bprp_spectrum_{source_id}.fits
    └── ...

./IACOB_Bstar_BPRP_plots/
    ├── 001_bstar_bprp_spectrum_{source_id}.png
    ├── 002_bstar_bprp_spectrum_{source_id}.png
    └── ...

bstar_bprp_spectrum_summary.csv
```

## Summary CSV Columns

- `source_id`: Gaia DR3 source identifier
- `name`: HD catalog name (e.g., HD205196)
- `sptype`: Spectral type (e.g., B0Ib)
- `has_bprp_spectrum`: TRUE/FALSE
- `rank`: Spectral type order (1 = earliest)
- `spectrum_file`: FITS filename
- `plot_file`: PNG filename

## Usage Notes

1. **High BP/RP coverage**: Almost all B-stars have BP/RP spectra
2. **Sorted by type**: Files are in spectral sequence for easy browsing
3. **HD names**: Star names are HD catalog identifiers
4. **No Teff/logg**: Use published catalogs if you need these parameters

## IACOB Survey

The IACOB (Investigation of Atmospheric properties of Blue supergiant stars) survey provides high-resolution spectroscopy and stellar parameters for massive stars. This dataset represents their B-star sample cross-matched with Gaia DR3.

## References

- [IACOB Survey](http://research.iac.es/proyecto/iacob/)
- [Simón-Díaz et al. (2011) - IACOB I](https://ui.adsabs.harvard.edu/abs/2011A%26A...530A.103S)
- [Holgado et al. (2018) - IACOB IV](https://ui.adsabs.harvard.edu/abs/2018A%26A...613A..65H)
