# GEV evidence outputs

Run `Rscript fit_surge_gev_v2.R` from the repository root. A successful, reviewed
run writes the following files here:

- `annual_maxima.csv`
- `surge_return_levels.csv`
- `gev_fit_summary.txt`
- `gev_diagnostics.png`

The committed evidence files were generated from a completed run of
`fit_surge_gev_v2.R` on 27 August 2026, using NOAA CO-OPS station 8418150
(Portland, Maine) records for 1980–2025, referenced to the MSL datum. All 46 annual
maxima passed the 80% data-coverage criterion. The maximum-likelihood optimizer
converged, and all four diagnostic panels were inspected. Return-level confidence
intervals were estimated using a 1,000-replicate parametric bootstrap with random seed
42. The parametric bootstrap captures **PARAMETER uncertainty only** and assumes that
the GEV family is correctly specified; consequently, the reported intervals do not
account for model-form uncertainty and may be narrower than the total uncertainty. The
fitted GEV parameters were cross-checked against `extRemes::fevd` and agreed to four
decimal places.
