# GEV evidence outputs

Run `Rscript fit_surge_gev_v2.R` from the repository root. A successful, reviewed
run writes the following files here:

- `annual_maxima.csv`
- `surge_return_levels.csv`
- `gev_fit_summary.txt`
- `gev_diagnostics.png`

Before committing them, confirm that the optimizer converged, inspect all four
diagnostic panels, record the number of successful bootstrap fits, and verify that
the fitted parameters reproduce the values reported in `README.md` and `damage.py`.

These generated evidence files are not included in the current commit because the R
runtime and NOAA download were not available in the environment used for this repair.
