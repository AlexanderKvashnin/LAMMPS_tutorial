# Melting and Solidification of Metals with MEAM Potential in LAMMPS

A hands-on tutorial for students: simulating the melting–crystallization hysteresis of aluminum and copper in three geometries — bulk crystal, semi-infinite slab, and nanoparticle — and comparing the results with a Python script.

# 1. Introduction and Physics Background

When a solid is heated, it eventually melts at the melting temperature **T_m**. When the same liquid is cooled, it usually does not crystallize at **T_m** but at a lower temperature **T_c < T_m** — a phenomenon known as **supercooling**. The gap between the melting and crystallization temperatures produces a **hysteresis loop** in the energy–temperature curve.

The magnitude of the hysteresis depends on:

- **Heating/cooling rate** — faster rates shift both transitions away from equilibrium.
- **System geometry** — free surfaces and low dimensionality reduce the melting temperature (surface premelting, Gibbs–Thomson effect).
- **System size** — nanoparticles melt at significantly lower temperatures than the bulk.

In this tutorial we use the **Modified Embedded Atom Method (MEAM)** potential, which captures the directional bonding and surface energetics of metals much better than the simpler pairwise EAM potential. This is important for surface melting and nanoparticle behavior.

The three systems studied:

| System        | Boundary conditions | Physics probed                            |
|---------------|---------------------|-------------------------------------------|
| Bulk crystal  | `p p p`             | Homogeneous melting / volume nucleation   |
| Slab          | `p p f`             | Melting nucleated at free surfaces        |
| Nanoparticle  | `f f f`             | Size-dependent melting (Gibbs–Thomson)    |

---

## 2. Required Files and Setup

You need the following files in your working directory:

- `library.meam` — shared MEAM library of parameters (from `$LAMMPS/potentials/`)
- `Al.meam` — parameters for aluminum
- `Cu.meam` — parameters for copper

Copy them from your LAMMPS installation:

```bash
cp $LAMMPS_DIR/potentials/library.meam .
cp $LAMMPS_DIR/potentials/Al.meam .
cp $LAMMPS_DIR/potentials/Cu.meam .
```

Then run any of the input scripts:

```bash
lmp -in melt_bulk_Al.in
```

Recommended project layout:

```text
project/
├── README.md
├── library.meam
├── Al.meam
├── Cu.meam
├── melt_bulk_Al.in
├── melt_slab_Al.in
├── melt_nano_Al.in
├── melt_bulk_Cu.in
├── melt_slab_Cu.in
├── melt_nano_Cu.in
└── plot_hysteresis.py
```

---

## 3. Case 1 — Bulk Crystal (Al)

**File:** `melt_bulk_Al.in`

### Full listing

```lammps
units           metal
dimension       3
boundary        p p p
atom_style      atomic
timestep        0.002

lattice         fcc 4.05
region          box block 0 8 0 8 0 8
create_box      1 box
create_atoms    1 box

pair_style      meam
pair_coeff      * * library.meam Al Al.meam Al

neighbor        0.3 bin
neigh_modify    delay 10

thermo          100
thermo_style    custom step temp pe ke etotal press vol

compute         mype all pe
compute         mytemp all temp
fix             myout all ave/time 1 1 100 c_mytemp c_mype file melt_bulk_Al.txt

velocity        all create 300.0 12345 dist gaussian
fix             nvt_heat all nvt temp 300.0 2000.0 0.1
run             500000
unfix           nvt_heat

fix             nvt_cool all nvt temp 2000.0 300.0 0.1
run             500000
unfix           nvt_cool

write_data      final_bulk_Al.data
```

### Command-by-command explanation

#### Setup

- **`units metal`** — selects the "metal" unit system: distances in Å, energy in eV, mass in g/mol, time in ps, temperature in K. This is the standard choice for metallic systems.
- **`dimension 3`** — three-dimensional simulation.
- **`boundary p p p`** — periodic boundary conditions (PBC) in all three directions. This mimics an *infinite* crystal: an atom leaving the right face re-enters on the left. No free surfaces exist.
- **`atom_style atomic`** — each atom is a single point particle (no bonds, no charges).
- **`timestep 0.002`** — integration step of 2 fs. This is a typical value for metals in `metal` units; larger values risk numerical instability.

#### Creating the lattice

- **`lattice fcc 4.05`** — defines a face-centered cubic lattice with a = 4.05 Å (the experimental lattice constant of Al). All subsequent `create_atoms` commands use this lattice.
- **`region box block 0 8 0 8 0 8`** — defines a rectangular region 8×8×8 *lattice units* (each lattice unit = one fcc cell = 4.05 Å). So the box is 32.4 Å on a side.
- **`create_box 1 box`** — creates the simulation box with 1 atom type.
- **`create_atoms 1 box`** — fills the box with atoms of type 1 on the fcc lattice. With 8³ fcc cells × 4 atoms/cell = **2048 atoms**.

#### Interatomic potential

- **`pair_style meam`** — use the MEAM potential. Note: in some LAMMPS builds the style is `meam/c` (the C-optimized version). Both have identical syntax.
- **`pair_coeff * * library.meam Al Al.meam Al`** — argument order:
  - `* *` — apply to all atom-type pairs.
  - `library.meam` — global MEAM parameter library.
  - `Al` — element name in the library.
  - `Al.meam` — system-specific parameter file.
  - final `Al` — element assignment for atom type 1.

#### Neighbor list

- **`neighbor 0.3 bin`** — build a neighbor list with a 0.3 Å skin. Atoms within (cutoff + 0.3 Å) are considered neighbors.
- **`neigh_modify delay 10`** — rebuild the neighbor list only every 10 steps, unless atoms have moved more than half the skin.

#### Thermodynamic output

- **`thermo 100`** — print thermodynamic info to the screen every 100 steps.
- **`thermo_style custom step temp pe ke etotal press vol`** — specify what to print: step, temperature, potential energy, kinetic energy, total energy, pressure, volume.

#### Writing data to a file

- **`compute mype all pe`** — define a compute that returns the total potential energy.
- **`compute mytemp all temp`** — a compute returning the instantaneous temperature.
- **`fix myout all ave/time 1 1 100 c_mytemp c_mype file melt_bulk_Al.txt`** — every 100 steps, write the current values of `c_mytemp` and `c_mype` to a file. The three numbers `1 1 100` mean: Nevery=1 (use every step), Nrepeat=1 (one sample per average), Nfreq=100 (output every 100 steps). The output file has 3 columns: **Step, Temperature (K), Potential Energy (eV)**.

#### Heating stage

- **`velocity all create 300.0 12345 dist gaussian`** — assign random velocities drawn from a Gaussian distribution such that the instantaneous temperature is 300 K. The number `12345` is the random seed (must be a positive integer, unique per run).
- **`fix nvt_heat all nvt temp 300.0 2000.0 0.1`** — Nosé–Hoover thermostat that linearly ramps temperature from 300 K to 2000 K over the run. The number `0.1` is the thermostat relaxation time in ps (`Tdamp`); the typical choice is ~100× timestep.
- **`run 500000`** — 500 000 steps × 2 fs = 1 ns. Temperature rises by (2000 − 300) / 1 ns ≈ **1.7 K/ps**.
- **`unfix nvt_heat`** — remove the heating thermostat before starting cooling.

#### Cooling stage

- **`fix nvt_cool all nvt temp 2000.0 300.0 0.1`** — the same thermostat but with the target temperature decreasing from 2000 K to 300 K. This produces the **cooling branch** of the hysteresis loop.
- **`run 500000`** — 1 ns cooling.
- **`unfix nvt_cool`** — clean up.
- **`write_data final_bulk_Al.data`** — save the final atomic configuration (useful for further analysis, e.g., common-neighbor analysis).

### What you should observe

Plotting PE vs T (the Python script does this), you should see:

- **Heating branch:** PE increases linearly (solid expands, anharmonicity), then at ≈ 900–1000 K there is a **jump** to higher PE — the crystal melts. The liquid continues to heat with the same slope but a larger intercept.
- **Cooling branch:** liquid cools, then at some lower temperature (≈ 700–800 K) the PE jumps down — the liquid crystallizes. Because of the finite cooling rate, the crystallization temperature is *lower* than the melting temperature.
- **Hysteresis loop:** the two branches do not coincide.

---

## 4. Case 2 — Semi-Infinite Slab (Al)

**File:** `melt_slab_Al.in`

### Full listing

```lammps
units           metal
dimension       3
boundary        p p f
atom_style      atomic
timestep        0.002

lattice         fcc 4.05
region          box block 0 8 0 8 0 20
create_box      1 box
create_atoms    1 box

pair_style      meam
pair_coeff      * * library.meam Al Al.meam Al

neighbor        0.3 bin
neigh_modify    delay 10

thermo          100
thermo_style    custom step temp pe ke etotal press vol

compute         mype all pe
compute         mytemp all temp
fix             myout all ave/time 1 1 100 c_mytemp c_mype file melt_slab_Al.txt

velocity        all create 300.0 12345 dist gaussian
fix             nvt_heat all nvt temp 300.0 1800.0 0.1
run             500000
unfix           nvt_heat

fix             nvt_cool all nvt temp 1800.0 300.0 0.1
run             500000
unfix           nvt_cool

write_data      final_slab_Al.data
```

### Differences from the bulk case

Only three lines differ from the bulk script:

- **`boundary p p f`** — periodic in x and y, **fixed** (free) in z. Atoms cannot cross the z-boundaries; the top and bottom faces are open surfaces. This creates a **slab** geometry.
- **`region box block 0 8 0 8 0 20`** — the box is now 20 lattice units thick in z (≈ 81 Å), so the slab has a well-defined thickness with two free surfaces.
- **Temperature ranges** are **300 → 1800 K → 300 K** instead of 300 → 2000 K, because surface melting begins earlier.

### Physics of the slab geometry

The two free surfaces act as **nucleation sites** for melting. In a real crystal, melting requires nucleating a liquid droplet inside the solid, which costs surface energy. A free surface removes that barrier: the liquid layer grows inward from the surface as temperature rises. This is called **surface premelting** and is responsible for the fact that a slab appears to melt at a *lower* temperature than the bulk.

During cooling, the two free surfaces again serve as nucleation sites — the liquid crystallizes at the surface and the crystalline front propagates inward. Because nucleation is easier, supercooling is smaller than in the bulk: the hysteresis loop is **narrower**.

### Additional note: atoms near a surface

You can verify this by computing a density profile along z (using `fix ave/chunk`), but that is beyond the scope of this tutorial. For now, the key observation is the *narrower* hysteresis loop compared to the bulk.

---

## 5. Case 3 — Nanoparticle 5 nm (Al)

**File:** `melt_nano_Al.in`

### Full listing

```lammps
units           metal
dimension       3
boundary        f f f
atom_style      atomic
timestep        0.002

lattice         fcc 4.05
region          box block 0 12 0 12 0 12
create_box      1 box
create_atoms    1 box

variable        radius equal 25.0
variable        cx equal 24.3
variable        cy equal 24.3
variable        cz equal 24.3

region          sphere sphere ${cx} ${cy} ${cz} ${radius} units box
group           sphere_atoms region sphere
group           outside_atoms subtract all sphere_atoms
delete_atoms    group outside_atoms

pair_style      meam
pair_coeff      * * library.meam Al Al.meam Al

neighbor        0.3 bin
neigh_modify    delay 10

thermo          100
thermo_style    custom step temp pe ke etotal press vol

compute         mype all pe
compute         mytemp all temp
fix             myout all ave/time 1 1 100 c_mytemp c_mype file melt_nano_Al.txt

velocity        all create 300.0 12345 dist gaussian
fix             nvt_heat all nvt temp 300.0 1200.0 0.1
run             400000
unfix           nvt_heat

fix             nvt_cool all nvt temp 1200.0 300.0 0.1
run             400000
unfix           nvt_cool

write_data      final_nano_Al.data
```

### Command-by-command explanation

#### Setup

- **`boundary f f f`** — no periodicity at all. The system is an isolated cluster in vacuum. This is essential: a periodic image of a nanoparticle would be an artificial crystal of nanoparticles.
- **`region box block 0 12 0 12 0 12`** — a 12×12×12 lattice-unit box (≈ 48.6 Å on a side), large enough to contain a 50 Å sphere with some vacuum padding.

#### Carving out the sphere

- **`variable radius equal 25.0`** — radius 25 Å = **diameter 5 nm**, as requested.
- **`variable cx/cy/cz equal 24.3`** — coordinates of the sphere center. Since the box is 48.6 Å on a side, the center is at (24.3, 24.3, 24.3) Å.
- **`region sphere sphere ${cx} ${cy} ${cz} ${radius} units box`** — a spherical region in absolute (Å) coordinates (the `units box` keyword means the coordinates are in Å, not in lattice units).
- **`group sphere_atoms region sphere`** — select all atoms inside the sphere.
- **`group outside_atoms subtract all sphere_atoms`** — define the complement: all atoms *outside* the sphere.
- **`delete_atoms group outside_atoms`** — remove atoms outside the sphere. What remains is a single fcc nanoparticle with free surfaces everywhere.

After this, the system contains **~4000 atoms** forming an fcc nanoparticle with a free surface everywhere.

#### Heating / cooling range

- **Temperature range 300 → 1200 → 300 K** — nanoparticles melt at much lower temperatures than the bulk, because a large fraction of atoms is on the surface. The **Gibbs–Thomson relation** predicts:

  ```
  T_m(d) = T_m_bulk · (1 − 4 σ_sl / (d · ρ_s · ΔH_f))
  ```

  where *d* is the particle diameter, σ_sl the solid–liquid interface energy, ρ_s the solid density, and ΔH_f the enthalpy of fusion. For a 5 nm Al particle, T_m is roughly 100–200 K below the bulk value.

#### Output

- **`write_data final_nano_Al.data`** — save the final configuration, useful for visualization (OVITO, VMD) or common-neighbor analysis to check crystallinity.

### Physics of the nanoparticle geometry

- **Large surface-to-volume ratio.** In a 5 nm particle, a substantial fraction of atoms lies within 1–2 atomic layers of the surface. These atoms have fewer neighbors and lower coordination, which lowers the effective cohesive energy of the particle.
- **Premelting of the surface layer.** As temperature rises, the outermost shell of the particle loses crystalline order first, forming a quasi-liquid skin. The liquid–solid interface then moves inward.
- **Gibbs–Thomson depression.** The melting temperature scales linearly with the inverse diameter. Smaller particles melt at lower temperatures.
- **Narrower hysteresis.** Because the liquid–solid interface is already present (the whole particle is essentially an interface), nucleation barriers are small — supercooling is modest.

---

## 6. Case 4 — Copper Variants

All three copper scripts are identical to the aluminum ones **except**:

1. Replace `Al` → `Cu` in `pair_coeff`.
2. Replace the lattice constant: **a(Cu) = 3.615 Å** (instead of 4.05 Å).
3. Replace the sphere center coordinates in the nanoparticle script: the box is 12 lattice units of 3.615 Å ≈ 43.4 Å on a side; the center is at **(21.69, 21.69, 21.69) Å**.
4. Adjust temperature ranges:
   - Bulk Cu: 300 → 2200 → 300 K
   - Slab Cu: 300 → 2000 → 300 K
   - Nano Cu: 300 → 1500 → 300 K
5. Replace the output filenames: `melt_bulk_Cu.txt`, `melt_slab_Cu.txt`, `melt_nano_Cu.txt`.

### 6.1. Bulk Cu — `melt_bulk_Cu.in`

```lammps
units           metal
dimension       3
boundary        p p p
atom_style      atomic
timestep        0.002

lattice         fcc 3.615
region          box block 0 8 0 8 0 8
create_box      1 box
create_atoms    1 box

pair_style      meam
pair_coeff      * * library.meam Cu Cu.meam Cu

neighbor        0.3 bin
neigh_modify    delay 10

thermo          100
thermo_style    custom step temp pe ke etotal press vol

compute         mype all pe
compute         mytemp all temp
fix             myout all ave/time 1 1 100 c_mytemp c_mype file melt_bulk_Cu.txt

velocity        all create 300.0 12345 dist gaussian
fix             nvt_heat all nvt temp 300.0 2200.0 0.1
run             500000
unfix           nvt_heat

fix             nvt_cool all nvt temp 2200.0 300.0 0.1
run             500000
unfix           nvt_cool

write_data      final_bulk_Cu.data
```

### 6.2. Slab Cu — `melt_slab_Cu.in`

```lammps
units           metal
dimension       3
boundary        p p f
atom_style      atomic
timestep        0.002

lattice         fcc 3.615
region          box block 0 8 0 8 0 20
create_box      1 box
create_atoms    1 box

pair_style      meam
pair_coeff      * * library.meam Cu Cu.meam Cu

neighbor        0.3 bin
neigh_modify    delay 10

thermo          100
thermo_style    custom step temp pe ke etotal press vol

compute         mype all pe
compute         mytemp all temp
fix             myout all ave/time 1 1 100 c_mytemp c_mype file melt_slab_Cu.txt

velocity        all create 300.0 12345 dist gaussian
fix             nvt_heat all nvt temp 300.0 2000.0 0.1
run             500000
unfix           nvt_heat

fix             nvt_cool all nvt temp 2000.0 300.0 0.1
run             500000
unfix           nvt_cool

write_data      final_slab_Cu.data
```

### 6.3. Nanoparticle Cu — `melt_nano_Cu.in`

```lammps
units           metal
dimension       3
boundary        f f f
atom_style      atomic
timestep        0.002

lattice         fcc 3.615
region          box block 0 12 0 12 0 12
create_box      1 box
create_atoms    1 box

variable        radius equal 25.0
variable        cx equal 21.69
variable        cy equal 21.69
variable        cz equal 21.69

region          sphere sphere ${cx} ${cy} ${cz} ${radius} units box
group           sphere_atoms region sphere
group           outside_atoms subtract all sphere_atoms
delete_atoms    group outside_atoms

pair_style      meam
pair_coeff      * * library.meam Cu Cu.meam Cu

neighbor        0.3 bin
neigh_modify    delay 10

thermo          100
thermo_style    custom step temp pe ke etotal press vol

compute         mype all pe
compute         mytemp all temp
fix             myout all ave/time 1 1 100 c_mytemp c_mype file melt_nano_Cu.txt

velocity        all create 300.0 12345 dist gaussian
fix             nvt_heat all nvt temp 300.0 1500.0 0.1
run             400000
unfix           nvt_heat

fix             nvt_cool all nvt temp 1500.0 300.0 0.1
run             400000
unfix           nvt_cool

write_data      final_nano_Cu.data
```

### Expected behavior

The resulting plots should show the same qualitative features as for Al, but shifted to higher temperatures (Cu melts at 1358 K in experiment, Al at 933 K). The hysteresis widths and relative ordering (bulk > slab > nano) should be preserved.

---

## 7. Python Visualization Script

**File:** `plot_hysteresis.py`

```python
#!/usr/bin/env python3
"""
plot_hysteresis.py — Plot melting hysteresis curves for Al or Cu
in three geometries: bulk, slab, nanoparticle.

Usage:
    python plot_hysteresis.py            # Al by default
    python plot_hysteresis.py Cu         # Cu
"""

import sys
import numpy as np
import matplotlib.pyplot as plt


def read_lammps_output(filename):
    """
    Read the file produced by `fix ave/time`.
    Columns: step, temperature, potential energy.
    Returns three numpy arrays.
    """
    data = np.loadtxt(filename, comments='#')
    step = data[:, 0]
    temp = data[:, 1]
    pe   = data[:, 2]
    return step, temp, pe


def main():
    metal = sys.argv[1] if len(sys.argv) > 1 else 'Al'

    step_bulk, T_bulk, PE_bulk = read_lammps_output(f'melt_bulk_{metal}.txt')
    step_slab, T_slab, PE_slab = read_lammps_output(f'melt_slab_{metal}.txt')
    step_nano, T_nano, PE_nano = read_lammps_output(f'melt_nano_{metal}.txt')

    # Optional: normalize energy per atom by editing N here.
    # N_bulk, N_slab, N_nano = 2048, 5120, 4000
    # PE_bulk /= N_bulk
    # PE_slab /= N_slab
    # PE_nano /= N_nano

    fig, ax = plt.subplots(figsize=(8, 6))

    ax.plot(T_bulk, PE_bulk, 'o-', ms=3, lw=1.2,
            color='tab:blue',   label=f'Bulk {metal}')
    ax.plot(T_slab, PE_slab, 's-', ms=3, lw=1.2,
            color='tab:orange', label=f'Slab {metal}')
    ax.plot(T_nano, PE_nano, '^-', ms=3, lw=1.2,
            color='tab:green',  label=f'Nanoparticle {metal} (5 nm)')

    ax.set_xlabel('Temperature (K)', fontsize=13)
    ax.set_ylabel('Potential energy (eV)', fontsize=13)
    ax.set_title(f'Melting Hysteresis — MEAM {metal}', fontsize=14)
    ax.legend(fontsize=11)
    ax.grid(True, alpha=0.3)

    plt.tight_layout()
    plt.savefig(f'hysteresis_{metal}.png', dpi=300)
    plt.show()


if __name__ == '__main__':
    main()
```

### What the script does

1. **`read_lammps_output`** — loads the three columns written by `fix ave/time`.
2. **Three curves on one plot** — bulk, slab, nanoparticle.
3. **Saves a PNG** — `hysteresis_Al.png` or `hysteresis_Cu.png`, ready for a report or a paper.

### Why it is useful

- It allows you to **directly compare** the three geometries.
- You can **measure T_m and T_c** from the graph as the points of the discontinuous jumps in PE.
- You can **quantify the hysteresis width** ΔT = T_m − T_c for each case.
- The CLI argument lets you reuse the same script for both metals without editing.

### Optional refinement — energy per atom

For a more physical plot, divide PE by the number of atoms and plot **energy per atom (eV/atom)**. You can obtain N from the LAMMPS log file (`grep "atoms" log.lammps`) or by adding a compute:

```lammps
variable natoms equal count(all)
```

Uncomment the corresponding block in the Python script to normalize.

---

## 8. Control Questions

Answer these after running the simulations and producing the plots.

1. **Estimate T_m and T_c** for each of the three Al systems (bulk, slab, nano) from the energy–temperature curves. Report the numbers in a table.

2. **Compare the hysteresis widths** ΔT = T_m − T_c for bulk, slab, and nanoparticle. Which system has the widest loop? Which the narrowest? Explain this in terms of nucleation barriers.

3. **Why does the melting temperature decrease** when going from bulk → slab → nanoparticle? Which physical effect dominates in each case (surface premelting, Gibbs–Thomson, curvature-driven melting)?

4. **Estimate the number of atoms in a 5 nm Al nanoparticle** from the geometry (volume × atomic density). Compare with the actual number of atoms in your simulation.

5. **How does the heating rate affect** T_m and T_c? Predict what happens if you increase the number of steps from 500 000 to 2 000 000 (i.e., slow down the ramp). Test your prediction if you have time.

6. **What would change if the thermostat had a very small Tdamp** (e.g., 0.01 ps)? Very large (e.g., 1 ps)? How would the energy–temperature curve look?

7. **What role does the MEAM potential play** compared to a simple Lennard-Jones potential? Which physical properties of Al would be wrong with LJ?

8. **Compare Al and Cu.** Which metal has the higher melting temperature in the simulation? Does it match the experimental values (933 K vs. 1358 K)?

9. **Examine the shape of the energy–temperature curve** near the transition. Why is the transition *sharp* in a perfect bulk crystal but *smoother* in a nanoparticle? (Hint: think about the distribution of local environments.)

10. **What is the finite-size effect** on melting temperature? If you simulated a 10 nm particle instead of 5 nm, would T_m be higher or lower? Use the Gibbs–Thomson relation to make a quantitative prediction.

---

## 9. Further Reading

- Foiles, S. M., Baskes, M. I., & Daw, M. S. (1986). *Embedded-atom-method functions for the fcc metals Cu, Ag, Au, Ni, Pd, Pt, and their alloys.* Phys. Rev. B, 33, 7983.
- Baskes, M. I. (1992). *Modified embedded-atom potentials for cubic materials and impurities.* Phys. Rev. B, 46, 2727.
- LAMMPS documentation:
  - [pair_style meam](https://docs.lammps.org/pair_meam.html)
  - [fix nvt](https://docs.lammps.org/fix_nh.html)
  - [fix ave/time](https://docs.lammps.org/fix_ave_time.html)
- Frenkel, D., & Smit, B. (2002). *Understanding Molecular Simulation.* Academic Press. Chapter 6 (melting and freezing), Chapter 7 (free-energy calculations).
- Buffat, P., & Borel, J.-P. (1976). *Size effect on the melting temperature of gold particles.* Phys. Rev. A, 13, 2287. — classic experimental paper on nanoparticle melting.

---

*Author: [Your Name]*  
*License: MIT*
