# Coastal flood risk assessment for transportation infrastructure

A 2D hydrodynamic inundation model built from the governing equations, coupled to
asset-level depth-damage functions and integrated to expected annual damage.

Storm surge forcing → 2D shallow water hydrodynamics → peak inundation depth over
specific road assets → depth-damage function → expected annual damage → asset ranking.

**Anthony Ariu Abu** · Ibadan, Nigeria · self-directed, 2026

![Model domain](figures/fig1_domain.png)

---

## What is real and what is not

| Component | Status |
|---|---|
| Hydrodynamic solver | Written from the governing equations; verified against four analytical cases |
| Surge distribution | **Fitted** — GEV on 46 annual maxima, NOAA CO-OPS station 8418150 (Portland, Maine), 1980–2025 |
| Terrain and road network | **Synthetic** — geometrically plausible, not a surveyed site |
| Unit costs | **Illustrative** |
| Depth-damage curve | **Illustrative** — shape is plausible, ordinates are not sourced |
| Tide amplitude | **Assumed** (1.40 m) — not taken from the published station datums |

Provenance is printed at runtime. Nothing here should be quoted as a result about
anywhere.

---

## Method

The **local-inertial** approximation of the 2D shallow water equations — advective
acceleration dropped, which is standard for floodplain and coastal overland flow and is
the scheme underlying LISFLOOD-FP.

Momentum per unit width, semi-implicit in friction:

```
                q^n  −  g · h_flow · Δt · ∂(h+z)/∂x
q^(n+1)  =  ────────────────────────────────────────────
             1  +  g · Δt · n² · |q^n| / h_flow^(7/3)
```

Staggered (Arakawa C) grid, adaptive CFL timestep, and a volume-based flux limiter so
a cell can never export more water than it holds.

Two elevation fields are maintained separately: **bed elevation** governs conveyance,
**deck elevation** governs damage. At a bridge these differ by metres. Conflating them
inflated expected annual damage by a factor of 2.7 (see Findings).

---

## Verification

| Test | Expected | Result |
|---|---|---|
| Closed flat basin | Level surface, exact conservation | Volume error 0.0000%, surface flat to 2.6 mm, mean depth exactly as predicted |
| Closed tilted basin | Level surface, exact conservation | Volume error 0.0000% |
| Sloping shore, fixed sea level | Fills to sea level; shoreline where bed = SWL | Converged to 0.6001 m against 0.600 m target; shoreline within one cell |
| Dry-bed dam break | Bounded, monotone, conservative | Volume error 0.0000%; depths stay within bounds |

The dam-break front underpredicts the Ritter analytical solution (570 m vs 688 m at
t = 30 s). Expected — dropping advective acceleration retards the dry-bed front. It
bounds where the scheme applies: floodplain flow, not supercritical or strongly
advective flow.

Mass balance is audited on every production run. Worst error across all 12 return
periods: **0.00000%**, with zero negative-depth clipping correction. The current
implementation does not log how often the conservative flux limiter activates.

Run `python test_solver.py` to reproduce.

---

## Extreme value analysis

The surge distribution was originally invented; it is now fitted to observed water
levels (`fit_surge_gev_v2.R`).

Hourly observed water level minus hourly astronomical prediction gives the residual.
NOAA predictions reference the 1983–2001 tidal epoch, so the residual carries
accumulated sea level rise as well as meteorology — the annual mean residual is
therefore removed year by year, leaving meteorological surge.

**Implementation cross-check and datum-processing plausibility check:**

- The hand-rolled MLE was cross-checked against the `extRemes` package, which returned
  **+0.6900, +0.1353, −0.1319 — identical to four decimal places**.
- Separately, the slope of the *removed* offset provides a plausibility check on the
  datum processing: **+2.83 mm/yr (p < 0.0001)**. It is derived from the same gauge
  record and is therefore not an independent validation of the GEV model.

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

| | Invented (v1) | Fitted (v2) |
|---|---|---|
| Location | 1.05 m | 0.6900 m |
| Scale | 0.38 m | 0.1353 m |
| Shape | 0 (Gumbel assumed) | −0.1319 (bounded tail) |
| 100-yr surge | 2.80 m | **1.16 m** |

No trend in annual maximum surge (+1.09 mm/yr, p = 0.53), so a stationary GEV is
defensible over this window.

---

## Results

**Expected annual damage: $13,210/yr against $6.88 M exposure.**

| Asset | EAD | % of asset value per year |
|---|---|---|
| Access Causeway | $12,388 | 0.77% |
| Coastal Highway | $822 | 0.03% |
| Inland Arterial | $0 | 0.00% |

The causeway carries **94% of the risk on 23% of the exposure**. Ranking by asset value would have prioritised the highway instead it is worth 1.8× the causeway. But the causeway begins taking damage at the 10-year event while the highway stays dry until the 100-year, and expected annual damage is dominated by frequency rather than severity.

![Damage and risk](figures/fig5_damage_and_risk.png)

---

## Findings

**1. The damage curve outweighs the hazard model.** Holding hydraulics completely fixed
and changing only the depth-damage curve moved expected annual damage by a factor of
**7.6**, while 100-year damage moved by 3.9. Frequent shallow events dominate the
probability integral, so the least-constrained part of the vulnerability model drives
the headline number. Refining the hydraulics further is misallocated effort while the
damage curve is uncalibrated.

**2. Conflating hydraulic and structural elevation inflated risk ~3× in the historical
v1 configuration.** The first working version read water depth over the channel bed as
water depth on the carriageway, so a bridged crossing appeared severely damaged in
events whose surge never approached its deck. In that obsolete v1 hazard/terrain
configuration, separating the two fields moved EAD from $539,930/yr to $196,827/yr
with no change to the hydraulics. These values are not directly comparable with the
current fitted-hazard EAD of $13,210/yr. The modelling distinction remains a general
trap wherever assets are rasterised onto a DEM.

**3. Tide timing dominates storm size in a macrotidal setting.** Across the entire
1.5-to-500-year range the peak still water level spans only **0.59 m**, because tidal
amplitude (1.40 m) exceeds surge at every return period. Whether an asset floods depends
largely on whether a storm coincides with high water. This model assumes coincidence —
the conservative case — but a defensible assessment would treat tide–surge timing as a
joint probability, and that choice plausibly matters more than any refinement of the
hydraulic scheme.

**Bonus: static and dynamic results diverge non-monotonically with duration and
roughness.** Holding peak water level fixed, dynamic damage ranges from 94.7% of the
static result at 2 h to 110.7% at 6 h and 159.6% at 24 h. The dynamic peak-inundation
area remains below the static area, so the damage exceedance must be diagnosed before
it is interpreted physically; possible contributors include transient water-surface
overshoot, use of a composite cellwise maximum-depth field, numerical sensitivity and
the nonlinear threshold damage curve. Marsh roughness materially changes the dynamic
result, an axis a bathtub calculation cannot represent, but the present sweep is a
diagnostic rather than a calibrated estimate of natural-protection benefit.

![Duration and roughness sensitivity](figures/fig7_duration_sensitivity.png)

---

## Running it

```bash
pip install numpy scipy matplotlib
python test_solver.py       # verification, ~10 s
python analysis.py          # 12-event ensemble, ~5 min
python duration_study.py    # sensitivity studies, ~5 min
```

New runs write figures and tables to the untracked `outputs/` directory. Paths resolve
relative to the scripts, so the model runs from any directory; set `FLOOD_OUT` to
redirect output. The repository's `figures/` and `results/` directories are curated,
tracked snapshots for the README and reproducibility record. They are updated only
after a run has been checked, which prevents an incomplete local run from silently
replacing the published reference outputs.

The ensemble is cached. The cache is fingerprinted against the terrain, roughness, deck
elevations, forcing parameters and return periods — change any of them and it discards
the cache and re-runs rather than silently mixing stale hydraulics with new terrain.

For the extreme value analysis, run `fit_surge_gev_v2.R` in R. It downloads directly
from the NOAA CO-OPS API using base R. Install `extRemes` to reproduce the independent
MLE implementation cross-check. The script writes `annual_maxima.csv`,
`surge_return_levels.csv`, `gev_fit_summary.txt` and `gev_diagnostics.png` under
`results/gev/`. These evidence files should be committed only after all diagnostics
have been inspected.

---

## Files

| File | Contents |
|---|---|
| `solver.py` | Local-inertial 2D shallow water solver with flux limiter |
| `test_solver.py` | Four verification cases with known answers |
| `terrain.py` | Synthetic domain, dual elevation fields, asset inventory |
| `damage.py` | Surge hazard, depth-damage curves, EAD integration |
| `run_event.py` | Single-event driver |
| `analysis.py` | Return-period ensemble, damage curve sensitivity, figures |
| `duration_study.py` | Static vs dynamic, duration and roughness sensitivity |
| `fit_surge_gev_v2.R` | Extreme value analysis of NOAA gauge records |
| `METHODS.md` | Full write-up, including limitations |

---

## Limitations

- Terrain, road network and unit costs are synthetic
- Depth-damage curve ordinates are not sourced, and per Finding 1 this dominates the uncertainty
- Tide amplitude is assumed, not taken from published station datums
- Tide and surge peaks assumed coincident (conservative, not expected)
- Sea level rise is removed from the surge distribution and is **not** added back into still water level anywhere
- Wave setup, wave overtopping, rainfall and groundwater are not modelled
- Local-inertial scheme omits advective acceleration — invalid for supercritical flow
- Direct asset damage only; no traffic disruption or network-detour cost, which for transportation infrastructure is usually the larger share of total economic impact
- One deterministic run per return period; no uncertainty propagation through the damage function
