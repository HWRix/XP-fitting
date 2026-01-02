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

### 3. Inconsistent Model Naming

The download notebook was modified to use unified naming (`atlas-or-phoenix-TTT-GG.txt`), but:
- The `save_model()` function still accepts a `model_type` parameter
- The verification cell counts "phoenix" and "atlas" prefixes separately (finds 0 phoenix files)
- Actual files all have `atlas-or-phoenix-` prefix

**Impact**: Cosmetic — the fitting notebook handles this correctly.

**Fix**: Clean up `save_model()` to always use unified naming, update verification logic.

---

### 4. Model Type Not Tracked in Output

The output CSV doesn't record which model type (PoWR vs ATLAS/PHOENIX) provided the best fit.

**Impact**: Loses information about whether the fit came from spherical (PoWR) or plane-parallel models.

**Fix**: Add `model_type` column to output.

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

### Temperature Coverage Gap at Cool End

**Current**: 7,600–50,000 K (PHOENIX + ATLAS + PoWR)

**Goal**: 3,800–50,000 K

**Missing**: PHOENIX models for 3,800–7,600 K need to be added. The Göttingen server has these; the download code just needs extended `PHOENIX_TEFF_VALUES`.

**Caution**: Below ~5,000 K, molecular bands (TiO, H₂O) become important. BP/RP resolution may be insufficient to constrain Teff precisely.

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

1. **Extend PHOENIX to lower Teff**
   - Add models from 3,800–7,600 K
   - Test fitting on known cool standards
   - Effort: Low (code structure already supports this)

2. **Expand R_V grid**
   - Change to ~10 values from 2.3–5.0
   - Pre-compute extinction table for all values
   - Effort: Trivial code change, ~3× slower fitting

3. **Add model_type to output**
   - Track which model family (PoWR/ATLAS/PHOENIX) gave best fit
   - Useful for validation and understanding systematics
   - Effort: ~10 lines of code

4. **Add quality flags**
   - `flag_av_boundary`: A_V at grid edge
   - `flag_degenerate`: multiple models with Δχ² < threshold
   - `flag_poor_spec_fit`: χ²_spec/dof > threshold
   - Effort: Moderate

### Medium Priority

5. **Parallelization**
   - Current: Serial loop over stars
   - Could use `multiprocessing` or `joblib` for ~10× speedup
   - Effort: Moderate (need to handle file I/O carefully)

6. **Caching/checkpointing**
   - Save intermediate results to allow restart after failure
   - Currently must restart from beginning if notebook crashes
   - Effort: Moderate

7. **MCMC or nested sampling for uncertainties**
   - Current: Only reports best-fit, no formal uncertainties
   - Could use `emcee` or `dynesty` for posterior distributions
   - Effort: Significant refactoring

8. **Better A_V optimization**
   - Current: Grid + parabolic refinement
   - Could use `scipy.optimize.minimize_scalar` for more robust convergence
   - Effort: Low

### Lower Priority

9. **Add metallicity dimension**
   - Download [M/H] = -1.0, -0.5, +0.3 grids
   - Add [M/H] to fitting grid
   - Effort: High (significantly more models, slower fitting)

10. **Handle binaries**
    - Fit composite spectra with two components
    - Effort: High (doubles parameter space)

11. **Web interface / CLI tool**
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

1. **Immediate**: Fix R_V grid (expand to ~10 values)
2. **Short-term**: Add PHOENIX models down to 3,800 K
3. **Short-term**: Add quality flags and model_type to output
4. **Medium-term**: Add parallelization for large catalogs
5. **Medium-term**: Validate against APOGEE/GALAH/benchmark stars
