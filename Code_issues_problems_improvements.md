# Code Issues, Problems, and Improvements

## Known Issues

### 1. Failed ATLAS Models at Low log g

Three ATLAS models failed during download/processing:
- Teff=12000 K, logg=2.0
- Teff=13000 K, logg=2.0
- Teff=15000 K, logg=2.0

**Cause**: ATLAS grid doesn't include these combinations (evolved supergiants at high Teff). The STScI files exist but contain zero flux values for these log g columns.

**Impact**: Low — these parameter combinations are rare and better served by PoWR models anyway (which do cover this regime with proper wind treatment).

**Fix**: Could interpolate from neighboring grid points, or simply rely on PoWR for logg < 2.5 at Teff > 10,000 K.

---

### 2. Hardcoded Gaia Archive Credentials

In `retrieve_BPRP_spectra-export_2026.v0.ipynb`, cell 4:
```python
username = '<name>'
password = '<passwd>'
```

**Impact**: Login fails (401 error in output), but code continues with anonymous access. Works fine for public DR3 data.

**Fix**: Remove login attempt or use `getpass.getpass()` interactively.

---

### 3. ~~Inconsistent Model Naming~~ [FIXED]

~~The download notebook was modified to use unified naming (`atlas-or-phoenix-TTT-GG.txt`), but the verification cell counts "phoenix" and "atlas" prefixes separately (finds 0 phoenix files).~~

**Status**: Fixed — verification now counts by Teff range (< 10000K = PHOENIX).

---

### 4. ~~Model Type Not Tracked in Output~~ [FIXED]

~~The output CSV doesn't record which model type (PoWR vs ATLAS/PHOENIX) provided the best fit.~~

**Status**: Fixed — output now includes `model_source` column ('phoenix', 'atlas', or 'powr').

---

### 5. No Convergence Flag

The fitting doesn't flag cases where:
- χ² is at the boundary of the A_V grid (0 or 6.0)
- Multiple models have similar χ² (degenerate solutions)
- Parallax constraint dominates (poor spectral fit accepted)

**Fix**: Add quality flags to output.

---

## Limitations

### R_V Grid is Coarse

**Current**: R_V ∈ {2.5, 3.1, 3.7} — only 3 values

**Problem**: R_V varies significantly along different sightlines:
- Diffuse ISM: R_V ~ 3.1
- Dense clouds: R_V ~ 5.0+
- Some sightlines: R_V ~ 2.5

With only 3 grid points, R_V errors of +/-0.3 are possible, propagating to A_V errors.

**Recommendation**: Expand to approximately 10 values:

```
R_V = [2.3, 2.5, 2.7, 2.9, 3.1, 3.3, 3.5, 3.7, 4.0, 4.5, 5.0]
```

The G23 extinction law supports R_V = 2.3 to 5.6.

---

### ~~Temperature Coverage Gap at Cool End~~ [FIXED]

~~**Current**: 7,600–50,000 K (PHOENIX + ATLAS + PoWR)~~

**Status**: Fixed — PHOENIX extended to 3,800K. Current coverage: 3,800–56,000 K.

**Note**: Below ~5,000 K, molecular bands (TiO, H₂O) become important. BP/RP resolution may be insufficient to constrain Teff precisely for very cool stars.

---

### No Metallicity Variation

All models use [M/H] = 0.0 (solar metallicity).

**Impact**:
- Minimal for hot stars (continuum dominated by H, He)
- Significant for cool stars where metal line blanketing matters
- Could bias results for halo stars or LMC/SMC targets

**Fix**: Would require downloading additional model grids at different [M/H] and adding metallicity to the fitting grid.

---

### Single-Star Assumption

The isochrone-based radius normalization assumes:
- Single star (no unresolved binaries)
- Star lies on theoretical isochrone (no peculiar abundances, rotation effects)

**Impact**: Binary companions will bias distance estimates. Rapid rotators may have modified Teff-logg-L relations.


## Potential Improvements

### High Priority

1. ~~**Extend PHOENIX to lower Teff**~~ [DONE]
   - ~~Add models from 3,800–7,600 K~~
   - **Status**: Implemented. 192 PHOENIX models now cover 3,800–10,000 K.

2. **Expand R_V grid**
   - Change to ~10 values from 2.3–5.0
   - Pre-compute extinction table for all values
   - Effort: Trivial code change, ~3× slower fitting

3. ~~**Add model_type to output**~~ [DONE]
   - ~~Track which model family (PoWR/ATLAS/PHOENIX) gave best fit~~
   - **Status**: Implemented as `model_source` column.

4. **Add quality flags**
   - `flag_av_boundary`: A_V at grid edge
   - `flag_degenerate`: multiple models with Δχ² < threshold
   - `flag_poor_spec_fit`: χ²_spec/dof > threshold
   - Effort: Moderate

5. **Add logg=1.0 to PHOENIX grid** [NEW]
   - Currently logg=[2.0, 2.5, 3.0, 3.5, 4.0, 4.5]
   - logg=1.0 needed for red giants
   - Effort: Low

6. ~~**Add R_std uncertainty calculation**~~ [DONE in v4]
   - v4 PSM interpolates logL continuously, derives R_Rsun via Stefan-Boltzmann
   - R_Rsun_fit varies continuously with Teff_fit and logL_fit

7. **Add model caching** [NEW]
   - Skip already-downloaded models during re-runs
   - Check if output file exists before download
   - Effort: Low

### Medium Priority

8. **Parallelization**
   - Current: Serial loop over stars
   - Could use `multiprocessing` or `joblib` for ~10× speedup
   - Effort: Moderate (need to handle file I/O carefully)

9. **Fitting checkpointing**
   - Save intermediate results to allow restart after failure
   - Currently must restart from beginning if notebook crashes
   - Effort: Moderate

10. **MCMC or nested sampling for uncertainties**
    - Current: Only reports best-fit, no formal uncertainties
    - Could use `emcee` or `dynesty` for posterior distributions
    - Effort: Significant refactoring

11. **Better A_V optimization**
    - Current: Grid + parabolic refinement
    - Could use `scipy.optimize.minimize_scalar` for more robust convergence
    - Effort: Low

### Lower Priority

12. **Add metallicity dimension**
    - Download [M/H] = -1.0, -0.5, +0.3 grids
    - Add [M/H] to fitting grid
    - Effort: High (significantly more models, slower fitting)

13. **Handle binaries**
    - Fit composite spectra with two components
    - Effort: High (doubles parameter space)

14. **Web interface / CLI tool**
    - Convert notebooks to standalone scripts
    - Add command-line interface for batch processing
    - Effort: Moderate

---

## Code Quality Issues

### Error Handling

- Download failures are caught but not logged persistently
- No retry logic for transient network failures
- Some exceptions print warnings but continue silently

### Documentation

- Inline comments are sparse in fitting loop
- No docstrings on helper functions
- Magic numbers (e.g., `0.03` systematic floor) not explained in code

### Testing

- No unit tests
- No validation against known standards
- Output not compared to literature values


## Recommended Next Steps

1. **Immediate**: Regenerate model_manifest.csv (re-run download notebook to remove PoWR duplicates)
2. **Immediate**: Run v4 fitting with N_MAX_FIT=None
3. **Short-term**: Add model caching to skip existing downloads
4. **Short-term**: Add logg=1.0 to PHOENIX grid for red giants
5. **Short-term**: Expand R_V grid to ~10 values
6. **Medium-term**: Add quality flags (boundary, degeneracy, poor fit)
7. **Medium-term**: Add parallelization for large catalogs
8. **Medium-term**: Validate against APOGEE/GALAH/benchmark stars

---

## Scaling Roadmap

### Current Workflow
```
source_ids.fits → [v2: enrich + download spectra] → sample_enriched.fits
                                                            ↓
                                    [v4: fit] → sample_enriched_fits.csv
```
- **Minimal input**: Only `source_id`, `Gmag`, `parallax`, `parallax_error` required
- **v2 auto-fetches** ra/dec from Gaia archive if not present (Cell 20)
- Output filenames trace back to input (e.g., `my_stars.fits` → `my_stars_enriched_fits.csv`)

### Phase 1: Current (up to ~1k stars)
- Keep current notebooks
- Add checkpointing to v4:
  - Output CSV appended incrementally
  - Skip source_ids already in output
  - Can restart after interruption

### Phase 2: Unified Script (5k+ stars)
- Single `fit_pipeline.py` with CLI
- Handles download + enrich + fit in one command
- Parallelized fitting loop with `multiprocessing`
- Example: `python fit_pipeline.py --input ids.csv --n-workers 8`

### Phase 3: Production Scale (50k-500k stars)
- Convert spectra storage to HDF5 (single file, ~10× faster I/O)
  - 100k spectra: ~2-3 GB compressed
  - 500k spectra: ~10-15 GB compressed
- Pre-compute A23/W25 lookup table for full sample
- Run on cluster or multi-core machine

### Key Design Principles
1. **Model prep stays separate** — truly one-time task
2. **Cache everything** — spectra, enrichment, fit results
3. **Checkpoint frequently** — enable restart after interruption
4. **Parallelize fitting** — main computational bottleneck

---
*Last updated: 2026-01-04*
