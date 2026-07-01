# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A customized fork of **laserbeamFoam** (an OpenFOAM-10 VOF solver suite for
laser–substrate interaction: welding, drilling, Laser Powder Bed Fusion) adapted
for NIST AM-Bench melt-track cases. It builds against **OpenFOAM-10** only
(the build scripts warn on any other `WM_PROJECT_VERSION`). C++ with OpenFOAM's
`.C`/`.H` convention where `.H` files included into a solver are code fragments,
not headers.

## Build

OpenFOAM-10 must be sourced first (`WM_PROJECT` env var must be set), then:

```bash
./Allwmake -j          # builds src/ libs then applications/ ; -j uses all cores
./Allwclean            # cleans apps + tutorials + build logs
```

Build order matters: `src/` (the `liblaserHeatSource` library) compiles before
`applications/` (which links against it). Each level has its own `Allwmake`.
Build logs are written to `log.Allwmake` at each level; `Allwmake` greps them
for `" Error "` / `" Stop."` to decide pass/fail. To rebuild a single target,
`cd` into it and run `wmake` (or its local `Allwmake`).

Binaries install to `$FOAM_USER_APPBIN`; libraries to `$FOAM_USER_LIBBIN` /
`$FOAM_LIBBIN`.

## Running a case

There is no test suite — verification is running tutorial cases under
`tutorials/` (`ch_5x5`, `ch_1x5`) via their `./Allrun` / `./Allclean`.

The `Allrun` scripts here are **not** vanilla: they `module purge`, wire up a
**local OpenMPI 5.0.7** (`$HOME/opt/openmpi-5.0.7`) and a local OpenFOAM-10
(`$HOME/OpenFOAM-10`), set `FOAM_SIGFPE=0`, then run the pipeline:

```
cp -r initial 0        # initial fields live in initial/, copied to 0/
blockMesh
setSolidFraction       # seeds alpha.metal from a powder/baseplate locations file
transformPoints "rotate=((0 1 0) (0 0 1))"
decomposePar
mpirun -np 80 laserbeamFoam -parallel
reconstructPar
foamToVTK
```

Adjust `-np` and the MPI/OpenFOAM paths to the local machine before running.

## Architecture

Two build products:

- **`liblaserHeatSource`** (`src/laserHeatSource/`) — the ray-tracing laser heat
  source. `laserHeatSource::updateDeposition(...)` discretises the Gaussian beam
  into rays, tracks them through the domain with Fresnel reflections, and fills
  the `deposition` volume field. **Assumes the laser enters the domain through
  the boundary whose outward normal is `(0 1 0)`** — this is why tutorials rotate
  the mesh with `transformPoints` so the build axis aligns with +y. Rays are
  written to `VTK/rays_<TIME_INDEX>.vtk` for ParaView. Vendors its own
  `interpolationTable` (for `timeVsLaserPosition` / `timeVsLaserPower` series).

- **`laserbeamFoam`** (`applications/solvers/laserbeamFoam/`) — two-phase
  incompressible VOF solver (metal + shielding gas). The top-level
  `laserbeamFoam.C` is a thin driver; the physics lives in the `#include`d `.H`
  fragments run each timestep: `alphaEqnSubCycle.H` (MULES phase fraction),
  `updateProps.H`, then `laser.updateDeposition(...)`, then `UEqn.H` → `TEqn.H`
  → `pEqn.H` inside the PIMPLE loop. `TEqn.H` runs an inner energy/melting
  corrector loop (governed by the `MELTING` sub-dict in `fvSolution`, read in
  `readControls.H`). Bundles two local libs it links: `VoFTurbulenceDamping` and
  `incompressibleInterPhaseTransportModel`.

Plus the **`setSolidFraction`** utility (`applications/utilities/`) — initialises
`alpha.metal` from a `locations` file of particle/baseplate positions + radii
before the solver runs.

### NIST-specific solver modifications

`laserbeamFoam.C` and `createFields.H` carry local changes (marked
`// MODIFICATION:`) that are **not** upstream — preserve them:

- **Track tracking:** `trackDuration` is read from `constant/trackProperties`;
  `currentTrack = floor(time/trackDuration) + 1` gives the active laser-track
  index each step.
- **Melt fields** (written to output, defined in `createFields.H`):
  - `condition` — 1 where a cell is molten this step (`alpha.metal >= 0.5` AND
    `epsilon1 >= 0.5`), else 0.
  - `meltHistory` — cumulative count of steps a cell has been molten.
  - `meltTrackID` — overwritten with `currentTrack` whenever a cell is molten,
    i.e. the last track that melted it.

## Case file layout

Tutorial cases follow OpenFOAM structure with a couple of local conventions:
`initial/` holds the t=0 fields (copied to `0/` by `Allrun`); `constant/` holds
`LaserProperties` (beam vector `V_incident`, `laserRadius`, `wavelength`,
`PowderSim`, and `timeVsLaserPosition`/`timeVsLaserPower` table refs),
`trackProperties` (`trackDuration`), and phase `physicalProperties.{metal,gas}`;
`system/` holds `bedPlateDict` (baseplate bounding box for `setSolidFraction`)
and `setFieldsDict`.
