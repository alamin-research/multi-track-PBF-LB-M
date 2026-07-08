# LPBF Multi-Track Melt Pool — Simulation vs Experiment

Melt-pool cross-section geometry for laser powder-bed fusion (LPBF) multi-track
scans. Each measured quantity has a **simulated** value and an **experimental**
value (mean ± standard deviation). All lengths in **micrometers (µm)**, areas in
**µm²**.

## Provenance

These are the melt-pool metrics reported in:

> Kumar, B., **Kanak, R. I.**, Sultana, N., Guo, J., Schrader, A., Liu, W. K.,
> Al Amin, A. (2026). *Laser Powder Bed Fusion Melt Pool Dynamics for Different
> Geometric Variations and Powder Layer Heights: High-Fidelity Multiphysics
> Modeling vs 2025 NIST Experiments.* arXiv:2604.07359.

- **`_sim`** = high-fidelity ray-tracing VOF solver (`laserbeamFoam`, OpenFOAM-10;
  fork at `multi-track-PBF-LB-M`). Physics: fusion + latent heat, Marangoni
  surface tension, recoil pressure, Boussinesq buoyancy, mush damping.
- **`_exp`** = NIST AM-Bench 2025 experiments, challenge
  **CHAL-AMB2025-06-PMPG** (Pad Melt Pool Geometry) on IN718.

### Process conditions (all cases)

| Parameter | Value |
|---|---|
| Laser power | 285 W |
| Scan speed | 960 mm/s |
| Hatch spacing | 0.11 mm |
| Beam radius (Gaussian) | 36 µm |
| Incidence angle | 5° |
| Track turnaround | 0.75 ms |
| Material | IN718 |

Study varies **two factors**: pad geometry (`5x5` = 5 mm × 5 mm, `1x5` = 1 mm ×
5 mm) and powder layer height (bare plate `0`, `80` µm, `160` µm). A "pad" = array
of overlapping serpentine laser tracks; the full NIST pad is 45 tracks / 5 mm.

### Track count and averaging

Ray-tracing VOF is expensive, so the full 45-track pad was **not** simulated
directly. A reduced number of tracks was simulated, then per-track metrics were
**extrapolated to the full 45 tracks and averaged**.

The per-track values in `trackwise` are the **directly simulated** tracks (file
exports tracks 1–10). The `pad_average` file holds the **45-track extrapolated +
averaged** result — the number that goes head-to-head with the NIST pad average.

## Files

| File | What it holds |
|------|---------------|
| `trackwise_sim_vs_exp.csv` | Per-track metrics (tracks 1–10), one row per (track × layer × quantity) |
| `pad_average_geometry_sim_vs_exp.csv` | Pad-averaged geometry (45-track) + solid/dilution areas, per case |

Two datasets: **per-track** (`trackwise`) and its **pad average** (`pad_average`).
The pad_average is the higher-level summary and additionally carries area
quantities the track file omits.

## Experiment factors (shared across files)

- **pad_size** — scan-pad footprint. `1x5` = 1 mm × 5 mm, `5x5` = 5 mm × 5 mm.
  (In `pad_average` written as `1mm_5mm` / `5mm_5mm`.)
- **position** — where the section was cut on the pad: `start` or `middle`
  (blank for `1x5`, which has no start/middle split).
- **layer_thickness_um** — powder layer thickness: `0`, `80`, or `160` µm.
  `0` = bare-plate / no powder (remelt) baseline.
- **track_no** — scan track index `1`–`10` (track-wise files only).
- **measurement_pos_mm** — axial cut location along the track (pad_average only).

## Measured quantities

| Quantity | Meaning |
|----------|---------|
| `bead_height` | Cap height above the plate surface (can be **negative** = depression below surface, e.g. bare-plate remelt) |
| `depth` | Melt-pool penetration below the plate surface |
| `overlap_depth` | Remelt depth in the track-to-track overlap region (inter-track fusion) |
| `width` | Melt-pool width at the surface |
| `solid_area` | Re-solidified (added bead) cross-section area, µm² (pad_average only) |
| `dilution_area` | Remelted substrate cross-section area, µm² (pad_average only) |

## Column suffix convention

- `_sim` — simulation prediction
- `_exp_mean` — experimental mean
- `_exp_std_dev` / `_stdv_exp` — experimental standard deviation

---

## `pad_average_geometry_sim_vs_exp.csv`

Wide, one row per case = (pad, layer thickness, axial position). 9 rows.
Values are the **45-track extrapolated + averaged** pad metrics (see *Track count
and averaging* above), matched against the NIST pad average.

Columns:
`Case`, `layer_thickness_um`, `measurement_pos_mm`, then for each of
{bead_height, depth, overlap_depth, width, solid_area, dilution_area}:
`*_sim`, `*_mean_exp`, `*_stdv_exp`.

Notes:
- `Case` values: `5mm_5mm`, `1mm_5mm`.
- Some `1mm_5mm` width cells are `NA` (experimental width not measured).
- Area values use scientific notation (e.g. `7.24E+04`).

## `trackwise_sim_vs_exp.csv`

One row per (track × layer × quantity). Columns:
`pad_size`, `position`, `track_no`, `layer_thickness_um`, `measurement_type`,
`sim_value`, `exp_mean_value`, `exp_std_dev`.

- `measurement_type` ∈ {bead_height, depth, overlap_depth, width}.
- Row groups: `1x5` (position blank), `5x5 start`, `5x5 middle`.
- Filter on `measurement_type` + `layer_thickness_um` to get one sim-vs-exp series
  across the 10 tracks.

## Data caveats

- **`not measured`** — where an experiment was not taken, the exp cells hold the
  literal string `not measured` (was blank / `NA`). Affects `1x5` width tracks
  6–10 (trackwise) and `1mm_5mm` width (pad_average). `sim` values still present.
- **Empty `sim` cells** — a few `5x5` `track_10` rows have a blank `sim_value`
  (simulation did not report that track); the exp columns are still populated.
  These are left blank, not marked `not measured`.
- **Negative bead_height** is physical (surface depression), not an error —
  common at layer thickness `0` (bare plate).
- Numeric precision varies (some values full-precision, some rounded); parse as
  float, treat empty strings as missing and `not measured` as a text sentinel.
