# LPBF Melt Pool Dynamics — Simulation Code

[![arXiv](https://img.shields.io/badge/arXiv-2604.07359-b31b1b.svg)](https://arxiv.org/abs/2604.07359)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

This repository contains the OpenFOAM-10 solver and simulation cases used in:

> **Laser Powder Bed Fusion Melt Pool Dynamics for Different Geometric Variations
> and Powder Layer Heights: High-Fidelity Multiphysics Modeling vs 2025 NIST
> Experiments**
> Badhon Kumar, Rakibul Islam Kanak, Nishat Sultana, Jiachen Guo, Andrew
> Schrader, Wing Kam Liu, Abdullah Al Amin<sup>†</sup>.
> arXiv:[2604.07359](https://arxiv.org/abs/2604.07359).
>
> <sup>†</sup> Corresponding author.

The code is a fork of the [`laserbeamFoam`](https://github.com/micmog/LaserbeamFoam)
solver suite, built against **OpenFOAM-10**, extended to reproduce the NIST
AM-Bench 2025 laser powder bed fusion pad experiments studied in the paper.

![Melt-track animation: 5 mm pad (top) and 1 mm pad (bottom), 80 µm powder](media/demo.gif)

*Top: `ch_5x5` (5 mm × 5 mm pad). Bottom: `ch_1x5` (1 mm × 5 mm pad). Both with an 80 µm powder layer.*

## What `laserbeamFoam` does

`laserbeamFoam` is a volume-of-fluid (VOF) solver for high energy density
laser–material interaction (welding, drilling, powder bed fusion). It treats the
metallic substrate and the shielding gas as two incompressible phases and
captures:

- fusion/melting state transition with latent heat,
- surface tension and its temperature dependence (Marangoni flow),
- a phenomenological recoil pressure for vaporisation,
- buoyancy (Boussinesq) and momentum damping in the solidifying mush.

The heat source is a **ray-tracing** model: the incident Gaussian beam is
discretised into rays that are tracked through the domain, depositing energy at
each surface reflection via the Fresnel equations.

## About this study

The paper studies how **powder layer height** (bare plate, 80 µm, 160 µm) and
**pad geometry** affect melt pool dynamics in laser powder bed fusion, and
compares the predicted melt pool metrics (depth, width, solidified and dilution
areas) against the NIST AM-Bench 2025 experiments.

The cases here target the NIST **CHAL-AMB2025-06-PMPG** challenge (Pad Melt Pool
Geometry): arrays of overlapping laser tracks ("pads") on IN718, measuring
per-track bead height, depth, width, overlap, and solidified / dilution areas.

The full NIST pad is 45 tracks over a 5 mm width. **These cases use at most 15
tracks** — the ray-tracing VOF solver is very expensive, so the reduced pads are
run on an HPC cluster to keep cost manageable while reproducing the AM-Bench
process conditions:

| Parameter | Value |
|---|---|
| Laser power | 285 W |
| Scan speed | 960 mm/s |
| Hatch spacing | 0.11 mm |
| Beam radius (Gaussian) | 36 µm |
| Incidence angle | 5° (`V_incident = (0.087 0.996 0)`) |
| Track turnaround | 0.75 ms |
| Material | IN718 |

Beyond the upstream solver, this fork adds per-step **melt tracking** fields
(`condition`, `meltHistory`, `meltTrackID`) and reads a per-track duration from
`constant/trackProperties`; see [CLAUDE.md](CLAUDE.md) for details.

## Installation

OpenFOAM-10 must be available and sourced (the `WM_PROJECT` environment variable
set) before building. Clone this repository, then use one of the routes below: a
native OpenFOAM-10 install, a Docker container (workstation), or an Apptainer
container (HPC). The container routes use the `openfoam/openfoam10-paraview510`
image.

### Option A — Native OpenFOAM-10

Install OpenFOAM-10 from the
[openfoam.org packages](https://openfoam.org/download/10-ubuntu/) or by compiling
the source, source its environment, then build this fork:

```bash
source /opt/openfoam10/etc/bashrc      # path depends on your install
./Allwmake -j                          # -j uses all cores
```

`./Allwclean` cleans the build.

### Option B — Docker

A convenience function that launches the container with your home directory
mounted and the user build paths preset (add it to your `~/.bashrc`):

```bash
of10 () {
    local of10_home="$HOME/.of10-home"
    mkdir -p "$of10_home"
    docker rm -f of10 >/dev/null 2>&1
    docker run -it --name of10 --hostname of10 --user root \
        -e HOME="$of10_home" -e USER=root -e LOGNAME=root \
        -e FOAM_USER_APPBIN="$HOME/platforms/linux64GccDPInt32Opt/bin" \
        -e FOAM_USER_LIBBIN="$HOME/platforms/linux64GccDPInt32Opt/lib" \
        -v "$HOME":"$HOME" -w "$PWD" \
        openfoam/openfoam10-paraview510 bash
}
```

Then, from the repository directory:

```bash
of10                                   # drops into the container at the repo
source /opt/openfoam10/etc/bashrc      # load the OpenFOAM environment
./Allwmake -j                          # -j uses all cores
```

`./Allwclean` cleans the build.

### Option C — Apptainer / Singularity (HPC)

On clusters without Docker, the build runs inside an OpenFOAM-10 `.sif` image.
If you do not have one, build it from the OpenFOAM-10 Docker image:

```bash
apptainer build ~/openfoam10.sif docker://openfoam/openfoam10-paraview510
```

A convenience function (add to `~/.bashrc`; replace the `.sif` path with your
image) that opens an interactive OpenFOAM shell, or runs a command directly:

```bash
of10 () {
    if [ "$#" -eq 0 ]; then
        apptainer exec --cleanenv --env USER="$USER" ~/openfoam10.sif \
            bash --noprofile --norc -lc "source /opt/openfoam10/etc/bashrc && exec bash --noprofile --norc -i"
    else
        apptainer exec --cleanenv --env USER="$USER" ~/openfoam10.sif \
            bash --noprofile --norc -lc "source /opt/openfoam10/etc/bashrc && $*"
    fi
}
```

Build the solver from the repository directory:

```bash
of10 ./Allwmake -j       # or: of10   (interactive), then ./Allwmake -j
```

Binaries install to `$FOAM_USER_APPBIN`; libraries to `$FOAM_USER_LIBBIN` /
`$FOAM_LIBBIN`.

### Optional — LIGGGHTS® (DEM) for powder beds

The powder-bed tutorial can regenerate its powder layer with the
[LIGGGHTS®](https://github.com/CFDEMproject/LIGGGHTS-PUBLIC) discrete element
solver. It is only needed if you want to rebuild the powder bed — the case ships
with a pre-generated one.

```bash
# Linux
sudo apt update
sudo apt install -y build-essential cmake gfortran git \
  libfftw3-dev libjpeg-dev libpng-dev libvtk6-dev libopenmpi-dev openmpi-bin

git clone https://github.com/CFDEMproject/LIGGGHTS-PUBLIC.git
cd LIGGGHTS-PUBLIC/src
make auto            # produces the `liggghts` executable
```

Add `liggghts` to your `PATH` (e.g. in `~/.bashrc`):

```bash
export PATH="$HOME/LIGGGHTS-PUBLIC/src:$PATH"
```

## Tutorial cases

Two representative cases live under `tutorials/`, one for each NIST AM-Bench pad
geometry. Both use an 80 µm powder layer on an IN718 plate and the base process
conditions above with a 0.75 ms track turnaround.

The paper covers six cases in total: the two pad geometries (5 mm × 5 mm and
1 mm × 5 mm), each at three powder layer thicknesses — bare plate, 80 µm, and
160 µm. The two cases provided here (both 80 µm) are examples; other conditions
are obtained by changing the powder layer in the case setup.

### `ch_5x5` — 5 mm × 5 mm pad

- Domain 1.94 × 5.14 × 0.5 mm, mesh 194 × 514 × 50 (~5.0 M cells, ~6 µm).
- 15 serpentine tracks (of the 45 in the full pad), 4.84 mm laser-on per track.
- End time 86.88 ms.
- The powder layer and baseplate are seeded into `alpha.metal` by
  `setSolidFraction` from `constant/location`.

### `ch_1x5` — 1 mm × 5 mm pad

- Domain 1.84 × 1.14 × 0.6 mm, mesh 184 × 114 × 60 (~1.26 M cells, ~6 µm).
- 15 serpentine tracks, end time 24.375 ms.
- The powder layer is generated with LIGGGHTS in `DEM_small/` and seeded into
  `alpha.metal` by `setSolidFraction` from `constant/location`.

### Running a case

Each case has an `Allrun` (prepare + solve) and an `Allclean` (reset) script:

```bash
cd tutorials/ch_5x5
./Allrun
```

`Allrun` performs, in order: copy `initial/` to `0/`, `blockMesh`,
`setSolidFraction`, `transformPoints` (rotates the mesh so the build axis aligns
with the laser), `decomposePar`, the parallel solve, `reconstructPar`, and
`foamToVTK`.

> **Note:** the `Allrun` scripts hard-code a local OpenMPI 5.0.7 and
> OpenFOAM-10 path and `mpirun -np 80`. Edit the MPI/OpenFOAM paths and the core
> count near the top/bottom of each `Allrun` to match your machine before
> running.

To (optionally) regenerate the `ch_1x5` powder bed first:

```bash
cd tutorials/ch_1x5/DEM_small
./Allrun        # runs liggghts, copies post/location to ../constant/
```

### Running on an HPC cluster (SLURM)

Each case also ships a `job.sh` SLURM batch script for cluster runs. It performs
the same setup and solve as `Allrun`, with two conveniences:

- **Restartable** — if `processor0/` already exists it skips meshing and
  decomposition and continues from the latest time, so a job stopped by the
  walltime limit can simply be resubmitted.
- **Auto post-processing** — once the run reaches `endTime` it runs
  `reconstructPar` and `foamToVTK` automatically.

Edit the `#SBATCH` directives (account, nodes/tasks, walltime, email) and the
OpenFOAM / MPI paths at the top to match your cluster, then submit:

```bash
cd tutorials/ch_5x5
sbatch job.sh
```

## Contact

For questions about the paper or this code, contact the corresponding author:

**Abdullah Al Amin** — Assistant Professor, Department of Mechanical and
Aerospace Engineering, University of Dayton
Email: <aamin1@udayton.edu>
Lab: [SMALT — Smart Manufacturing Advancement and Logistics Technology](http://smalt.dev/)

## License

`laserbeamFoam`, and by extension this fork, is licensed under the
[GNU General Public License version 3](https://www.gnu.org/licenses/gpl-3.0.en.html),
following OpenFOAM.

## Citing

If you use this code, please cite our paper:

```bibtex
@article{Kumar2026LPBFMeltPool,
  title   = {Laser Powder Bed Fusion Melt Pool Dynamics for Different Geometric
             Variations and Powder Layer Heights: High-Fidelity Multiphysics
             Modeling vs 2025 NIST Experiments},
  author  = {Kumar, Badhon and Kanak, Rakibul Islam and Sultana, Nishat and
             Guo, Jiachen and Schrader, Andrew and Liu, Wing Kam and
             Al Amin, Abdullah},
  journal = {arXiv preprint arXiv:2604.07359},
  year    = {2026},
  doi     = {10.48550/arXiv.2604.07359}
}
```

Please also cite the upstream `laserbeamFoam` solver:

```bibtex
@article{Flint2023laserbeamFoam,
  title   = {laserbeamFoam: Laser ray-tracing and thermally induced state
             transition simulation toolkit},
  author  = {Flint, T. F. and Robson, J. D. and Parivendhan, G. and Cardiff, P.},
  journal = {SoftwareX},
  volume  = {21},
  pages   = {101299},
  year    = {2023},
  doi     = {10.1016/j.softx.2022.101299}
}
```

## References

- Kumar, B., Kanak, R. I., Sultana, N., Guo, J., Schrader, A., Liu, W. K., &
  Al Amin, A. (2026). *Laser Powder Bed Fusion Melt Pool Dynamics for Different
  Geometric Variations and Powder Layer Heights: High-Fidelity Multiphysics
  Modeling vs 2025 NIST Experiments.* arXiv:2604.07359.
  https://arxiv.org/abs/2604.07359
- Flint, T. F., Robson, J. D., Parivendhan, G., & Cardiff, P. (2023).
  *laserbeamFoam: Laser ray-tracing and thermally induced state transition
  simulation toolkit.* SoftwareX, 21, 101299.
  https://doi.org/10.1016/j.softx.2022.101299
- Parivendhan, G., Cardiff, P., Flint, T., et al. (2023). *A numerical study of
  processing parameters and their effect on the melt-track profile in Laser
  Powder Bed Fusion processes.* Additive Manufacturing, 67, 103482.
  https://doi.org/10.1016/j.addma.2023.103482
- NIST AM-Bench 2025. *AMB2025-06 and AMB2025-07 Benchmark Measurements and
  Challenge Problems.* Calibration dataset: https://doi.org/10.18434/mds2-3707
- Deisenroth, D., Mekhontsev, S., & Grantham, S. (2025). *Design and Calibration
  of the Fundamentals of Laser-Material Interaction (FLaMI) Powder Bed Fusion
  Testbed at NIST.* NIST AMS 100-66. https://doi.org/10.6028/NIST.AMS.100-66
- Kloss, C., Goniva, C., Hager, A., Amberger, S., & Pirker, S. (2012). *Models,
  algorithms and validation for opensource DEM and CFD–DEM.* Progress in
  Computational Fluid Dynamics, 12(2/3), 140–152.
  https://doi.org/10.1504/PCFD.2012.047457

## Acknowledgement

This offering is not approved or endorsed by OpenCFD Limited, producer and
distributor of the OpenFOAM software, and owner of the OPENFOAM® and OpenCFD®
trade marks. OPENFOAM® is a registered trademark of OpenCFD Limited.
