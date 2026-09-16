# Turbulent Flow in an Annular Fuel-Pin Coolant Channel — Liquid Sodium CFD
A CFD study built in ANSYS Fluent, modelling turbulent forced convection through the annular gap between a nuclear fuel pin and its coolant channel — a simplified stand-in for a fast-reactor fuel subassembly — validated against the Seban-Shimazaki liquid-metal correlation.

## Table of Contents
- [What Was Actually Done](#what-was-actually-done)
- [Setup / Software Used](#setup--software-used)
- [Reproducing the Results](#reproducing-the-results)
- [Results](#results)
- [Method, Briefly](#method-briefly)
- [Limitations](#limitations)
- [Repository Structure](#repository-structure)
- [Author](#author)

## What was actually done

- Modelled a 2D axisymmetric annulus (r₁ = 5 mm to r₂ = 6.5 mm, L = 0.7 m) representing the fuel pin / coolant channel gap, solved in ANSYS Fluent using k-ω SST turbulence and the energy equation
- Created a custom constant-property liquid sodium material (ρ = 856 kg/m³, Cp = 1274 J/kg·K, k = 71 W/m·K, μ = 2.8×10⁻⁴ kg/m·s at 673 K), since sodium isn't in Fluent's default database
- Set inlet velocity to 5.5 m/s to target Re ≈ 50,000, and applied a constant heat flux of 2.5 MW/m² on the pin wall to represent fuel-pin surface heating, with the outer channel wall adiabatic
- Confirmed mass conservation (inlet/outlet mass flow match to within 10⁻¹² kg/s) and energy conservation (converged heat transfer rate at the pin wall matches the hand-calculated value to within 0.04%) before trusting any downstream result
- Diagnosed and fixed a real convergence problem: the first attempt diverged outright (continuity blew up past 10¹⁹), and after fixing that with lower under-relaxation, the wall temperature was still climbing after 5,000 iterations with no sign of flattening — traced to the domain's extreme length-to-gap ratio (~470:1) meaning the thermal field takes far longer to propagate through the full length than the momentum field does. Fixed by patching the temperature field closer to its expected value and switching to the Coupled scheme
- Learned the hard way that a flat convergence plot isn't proof of a *correct* answer — one "converged" run settled cleanly at a wall temperature of 674.8 K, which a quick hand energy-balance immediately flagged as physically wrong (expected 770–850 K); the actual issue turned out to be worth re-checking the boundary condition rather than just running longer
- Extracted local wall and bulk (mass-weighted average) temperatures at four downstream stations (x = 0.1, 0.3, 0.5, 0.6 m) using Line/Rake surfaces, rather than relying on a single whole-wall average, and computed local Nusselt number at each
- Validated against the Seban-Shimazaki correlation (Nu = 5.0 + 0.025·Pe⁰·⁸ ≈ 7.1 for this Peclet number) instead of Dittus-Boelter, since sodium's Prandtl number (~0.005) puts it well outside Dittus-Boelter's valid range

## Setup / Software Used

* ANSYS DesignModeler (geometry — single 2D axisymmetric rectangle)
* ANSYS Meshing — Mapped Face Meshing, wall-biased structured quad mesh
* ANSYS Fluent 2026 R1, Student version (solver)
* Fluent's native Surface Integrals, XY-Plot, and Flux Report tools for post-processing and validation

## Reproducing the Results

1. Open ANSYS Workbench → Fluid Flow (Fluent)
2. Build the geometry as a single flat rectangle in DesignModeler's XY plane — bottom edge at Y = 5 mm, top edge at Y = 6.5 mm, spanning X = 0 to X = 700 mm — and convert to a surface body (Concept → Surfaces From Sketches). Name the four edges `inlet`, `outlet`, `pin_wall`, `outer_wall`
3. Mesh with Mapped Face Meshing; size only the inlet edge (biased toward both ends, ~40 divisions) and pin_wall edge (uniform, ~150–200 divisions) — the opposite edges inherit their node counts automatically
4. In Fluent: set 2D Space to Axisymmetric, Viscous Model to k-ω SST, Energy on
5. Create a custom fluid material named for liquid sodium with the four constant properties listed above (evaluated at your chosen reference temperature), and assign it to the fluid cell zone — double-check this, since Fluent defaults new zones to `air` and won't warn you if you forget
6. Boundary conditions: `inlet` — velocity inlet (5.5 m/s, 5% turbulence intensity, hydraulic diameter 0.003 m, 673 K); `outlet` — pressure outlet (0 Pa gauge); `pin_wall` — wall, heat flux 2.5×10⁶ W/m²; `outer_wall` — wall, adiabatic
7. Solution Methods: Coupled scheme, PRESTO! pressure, second-order upwind for momentum/turbulence/energy
8. If continuity diverges on the first run, lower under-relaxation (momentum ≈ 0.3, pressure ≈ 0.2, energy ≈ 0.5) and re-initialize before retrying
9. If wall temperature is still drifting after several thousand iterations even though momentum/mass residuals look converged, patch the temperature field (Solution Initialization → Patch) to a value closer to the expected converged range rather than just running more iterations from a uniform inlet-temperature start
10. Before trusting the result, sanity-check it: compare `Reports → Fluxes → Total Heat Transfer Rate` at pin_wall against a hand calculation of Q = q″ × A, and confirm inlet/outlet mass flow rates match via `Reports → Fluxes → Mass Flow Rate`
11. Extract local Nu using Line/Rake surfaces at several downstream x-stations, with `Results → Reports → Surface Integrals → Mass-Weighted Average` for bulk temperature and the exported `.xy` wall-temperature data for local wall temperature

## Results

**Reynolds and Peclet numbers** — Re ≈ 50,400, Pr ≈ 0.005, Pe = Re·Pr ≈ 253

**Mass and energy conservation** — inlet/outlet mass flow match to 2.17×10⁻¹² kg/s; converged pin-wall heat transfer rate (54,977.87 W) matches the hand-calculated ~55,000 W to within 0.04%

**Local Nusselt number vs. Seban-Shimazaki (Nu ≈ 7.1):**

| Station x (m) | T_wall (K) | T_bulk (K) | Nu (CFD) | Deviation |
|---|---|---|---|---|
| 0.1 | 710.70 | 697.36 | 7.92 | +11.5% |
| 0.3 | 759.02 | 745.53 | 7.83 | +10.3% |
| 0.5 | 807.35 | 793.56 | 7.66 | +7.9% |
| 0.6 | 831.51 | 817.88 | 7.75 | +9.2% |

Nu is highest close to the inlet and gradually settles toward a slightly lower, roughly constant value further downstream — the expected thermal entry-length trend, though modest in magnitude here because sodium's high thermal diffusivity gives it an unusually short thermal entry length compared to ordinary fluids. All four points land within about 8–12% of Seban-Shimazaki, which is a solid, defensible result given the correlation's own quoted uncertainty is in a similar range.

Full report plots (residuals, wall shear, wall temperature, mass flow) and raw exported `.xy` data for the axial wall-temperature profile and each station are in `results/`.

## Method, briefly

- **Geometry:** single 2D axisymmetric rectangle, r₁ = 5 mm to r₂ = 6.5 mm, L = 0.7 m — no 3D revolve, Fluent handles the axisymmetry internally once `2D Space` is set to Axisymmetric.
- **Mesh:** Mapped Face Meshing for a structured all-quad grid, with the radial edge biased toward both walls to resolve the boundary layer at both pin_wall and outer_wall.
- **Fluid:** custom constant-property liquid sodium at 673 K, chosen so the CFD result compares cleanly against a correlation that itself assumes constant properties at a single reference temperature — using Fluent's temperature-varying property option would have muddied that comparison.
- **Convergence:** SIMPLE was enough for momentum and mass, but the energy equation needed help — the domain's ~470:1 length-to-gap ratio meant the thermal front had to propagate the full axial length before the wall temperature genuinely stopped changing, which took far longer than the residual plots alone suggested. Patching the temperature field and switching to the Coupled scheme resolved this.
- **Validation:** rather than trusting the residual plots alone, every result was cross-checked against something independent — mass balance against the flux report, heat transfer rate against a hand calculation, and one "converged" run's wall temperature against an expected range from the energy balance, which is what caught a genuinely wrong result before it went into a report.
- **Nu extraction:** local wall and mass-weighted bulk temperatures at four downstream stations, rather than a single whole-wall average, to check whether the flow had reached a fully-developed thermal state and to show the entry-length trend directly.

## Limitations

Seban-Shimazaki was developed for fully-developed flow in a circular pipe, not an annulus — the geometry mismatch is a real, if modest, source of the deviation seen at every station. Data was pulled at four discrete axial stations rather than as a continuous profile, so the entry-length decay in Nu is inferred from four points, not fully mapped. A formal three-level Grid Convergence Index (GCI) study via Richardson extrapolation was planned but not completed in this version — the production mesh was checked for orthogonal quality and aspect ratio rather than run through the full GCI procedure. Wall shear stress values were confirmed converged from the report plots but weren't pulled as precise numeric outputs in this pass, so a friction-factor comparison against the Moody chart is a natural next addition rather than something already included here.

## Repository structure

```
├── README.md
├── report/
│   └── report.docx   → full write-up with methodology, all convergence plots, and the Nu validation table
├── geometry/
│   └── Geometry
│   └── geometry.png
├── mesh/
│   └── CFD_mesh.msh
│   └── mesh.png
├── results/
│   ├── residuals/            → scaled residuals, the converged final run
│   ├── report-plots/         → wall shear, wall temperature, and mass flow convergence history
│   ├── wall_temp_axial.xy    → exported axial wall-temperature profile used for local Nu extraction
│   └── station_data/         → mass-weighted average bulk temperature at each of the four stations
└── case_files/
    └── turbulent-flow-in-annular-pipe
    └── turbulent-flow-in-annular-pipe_files
```

## Author

**Pratyush Dash**

B.Tech Chemical Engineering, KIIT University, Bhubaneswar
