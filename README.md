# Victorian Transmission System Dynamic Model

A dynamic model of the Victorian transmission system (Australia) for the [STEPSS](https://stepss.sps-lab.org/) platform: 125 buses at 500, 330, 220, 132, 66 and 33 kV plus generator terminals at 16 and 22 kV, 126 lines and 51 transformers, with 11 synchronous generating units (8 thermal, 3 hydro), the interconnectors to New South Wales and South Australia modelled as synchronous machines, and 23 converter-interfaced plants (17 wind, 6 solar) on the `WT4` model. It was built to study fast frequency response in a system with high renewable and electrified-heat shares, and comes with scripts that generate operating points for any half-hour of five study days and three electrification levels. This was an academic effort to model the system, and there may be substantial differences from the real one.

The development and application of the case are described in:

- J. Vorwerk, *Integration of Fast Demand Response into Renewable Power Systems*, PhD thesis, ETH Zurich, [doi:10.3929/ethz-b-000643071](https://doi.org/10.3929/ethz-b-000643071).
- J. Vorwerk, I. Saedi, P. Mancarella, G. Hug, "Electrifying thermal loads vs. installing batteries: A case study on fast frequency resource potentials of the Victorian power system", *Electric Power Systems Research*, vol. 235, 110622, 2024, [doi:10.1016/j.epsr.2024.110622](https://doi.org/10.1016/j.epsr.2024.110622).

## Repository Files

| File | Description |
|------|-------------|
| `pyRdat_ss.dat` | Network and operating point: buses, loads, lines, transformers and the solved voltages (`LFRESV`). Day 12_22, time step 24, no electrification |
| `pyRdat_dyn.dat` | Dynamic models for the same operating point: synchronous machines with `ENTSOE_simp` or `ST1A` exciters and `TGOV1` or `HYGOV` governors, interconnectors, and the `WT4` wind and solar plants |
| `SOLVEROPTIONS.dat` | Solver settings (centre-of-inertia reference frame) |
| `nothing.dst` | No disturbance, 20 s; holds commented example events |
| `trip_SM35G1.dst` | Loss of SM35G1, a 500 MW unit at bus 35G, at t = 0.5 s; 20 s |
| `obs.dat` | Observables: every bus, synchronous machine and injector |
| `cmd.txt` | RAMSES command file: the three data files with `nothing.dst` |
| `VICSystem.cfg` | STEPSS scenario file: opens the SM35G1 trip in the STEPSS GUI with **File > Load configuration**. Paths are relative to this folder |
| `TNSingleSim.py` | Example study: runs `cmd.txt`, trips SM35G1 at t = 0.5 s, plots generators, reserve activation and wind farms (`Generators.png`, `Reserve.png`, `WindFarms.png`) and estimates the kinetic energy supplied by generators and interconnectors |
| `GenerateOPs/TNLoadFlow.py` | Builds and solves the power flow of every half-hour of a chosen day and electrification level with HELIOS, into `GenerateOPs/LFData/` |
| `GenerateOPs/TNDynamicData.py` | Turns one of those power-flow solutions into `pyRdat_ss.dat` and `pyRdat_dyn.dat` |
| `GenerateOPs/GridData/` | Network, load, generator, exciter and governor data the two scripts read |
| `GenerateOPs/Scenarios/` | DC-OPF dispatch (`Output_elec_*`) and demand (`demand_*`) for days 5_15, 7_17, 8_9, 10_21 and 12_22 at electrification 0, 50 and 100 % |

## Quick Start

With [stepss](https://stepss.sps-lab.org/python/), which bundles the engine. The network has fewer than 1000 buses, so no licence is needed.

```python
import stepss
case = stepss.cfg('cmd.txt')          # pyRdat_ss.dat, pyRdat_dyn.dat, SOLVEROPTIONS.dat, nothing.dst
case.clearDst(); case.addDst('trip_SM35G1.dst')
case.addTrj('out.trj')
stepss.sim().execSim(case)
ext = stepss.extractor('out.trj')
ext.getSync('SM35G2').S.plot()
```

`python TNSingleSim.py` runs the full example study from this folder; it needs `pandas` and `matplotlib`. In the STEPSS GUI, open `VICSystem.cfg`.

## Generating other operating points

The two scripts in `GenerateOPs/` are run from inside that folder and need `pandas`, `numpy`, `openpyxl`, `matplotlib` and `seaborn`.

1. In `TNLoadFlow.py`, choose `date` (one of `5_15`, `7_17`, `8_9`, `10_21`, `12_22`), `electrification` (`0_0`, `50_50` or `100_100`, the share of gas-based residential heating that is electrified) and the power factor, then run it. It solves the 48 half-hours of that day with HELIOS, stores each solution in `LFData/`, and plots the DC-OPF dispatch against the full power-flow solution.
2. In `TNDynamicData.py`, choose the same `date` and `electrification` and a `time` from 1 to 48, then run it. It **overwrites** `pyRdat_ss.dat` and `pyRdat_dyn.dat` in this folder with that operating point.

The slack is the NSW interconnector at bus 681. Its row in `SteadyStateData.xlsx` carries no voltage set-point, which the original power-flow program accepted and HELIOS does not, so `TNLoadFlow.py` sets it to 1.003986 pu; with that value the chain reproduces the shipped operating point to within 0.005 Mvar.

## Status

**Runs.** With stepss 3.84 (RAMSES 3.84, HELIOS 1.5.0), on 2026-11-20:

- with `nothing.dst` every bus holds its power-flow voltage to 8e-6 pu over 20 s;
- with `trip_SM35G1.dst` frequency falls to a nadir of 49.55 Hz at about 3 s and settles near 49.82 Hz, with every voltage above 0.955 pu;
- `TNSingleSim.py` runs end to end;
- `TNLoadFlow.py` converges at all 48 half-hours of day 12_22 without electrification, and `TNDynamicData.py` then regenerates the two data files to within power-flow tolerance.

The case was first built against a custom RAMSES build, and four things had to change for the current engine: the exciter `exc_ENTSOE` is the registered `ENTSOE_simp`, the governor `TGOV1D` is registered as `TGOV1` and takes `VMAX` before `VMIN`, and the hydro governor `Hydro` is `HYGOV`. The spreadsheets in `GenerateOPs/GridData/` carry the same corrections, so regenerated files keep them. The full list is in [SPS-L/stepss-test-systems#4](https://github.com/SPS-L/stepss-test-systems/issues/4).

## Documentation

The data formats, models and solver settings are documented in the STEPSS user guide at [stepss.sps-lab.org](https://stepss.sps-lab.org/).

## License

This repository carries no licence file. Check with the author before redistributing the data.

## Authors

Model and scripts by J. Vorwerk. Maintained within the [STEPSS](https://stepss.sps-lab.org/) test systems by the [Sustainable Power Systems Laboratory (SPS-L)](https://sps-lab.org/), Cyprus University of Technology.
