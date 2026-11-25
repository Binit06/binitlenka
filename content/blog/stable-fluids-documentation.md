---
title: Stable Fluids
description: From-scratch Python implementation of Jos Stam's Stable Fluids paper, with video output and handwritten math notes.
date: 2026-11-25
tags:
  - python
  - simulation
  - math
  - research
---
# Stam's Stable Fluid Simulation

This is the implementation that was written in just one day while I was bored and wanted to implement a research paper and freshen up my maths skills, the implementation is rough around the edges but still its deserving of its own article.

**Repository:** [https://github.com/Binit06/stable-fluids](https://github.com/Binit06/stable-fluids)

---
## The Simulation

![[assets/stable-fluids-documentation/fluid_simulation.mp4]]

It's not the prettiest fluid simulation ever made, but it works well enough to see what the Stable Fluids method is doing on a 48×48 grid.

---

## How Things Are Done

Most of the simulation lives in `main.py` in the repository, where each function maps directly to the equations in these notes:

### 1. Navier-Stokes & The Eulerian Grid
Breaking down `∂u/∂t = -u(∇ · u) - (1/ρ)∇p + ν∇²u + f` across grid cells (advection, pressure gradient, viscosity diffusion) alongside the scalar density grid.

![[assets/stable-fluids-documentation/page1.jpeg]]

### 2. Incompressibility & Pressure Correction
Explaining `∇ · u = 0` (`Vin = Vout`), why numerical errors introduce divergence, and deriving the pressure Poisson equation `∇²p = -ρ ∇ · [(u · ∇)u]`.

![[assets/stable-fluids-documentation/page2.jpeg]]

### 3. Fractional Step Method & Grid Iteration
Solving for intermediate velocity `u*` without pressure first, then iterating `∇²p^{n+1} = (ρ / Δt) ∇ · u*` over the grid (Jacobi iterations) to make the final velocity field divergence-free.

![[assets/stable-fluids-documentation/page3.jpeg]]

---

## Notes & Performance

* **References used:** Jos Stam's *Stable Fluids* paper (included in the repo) and [The Visual Room — Poisson equation for pressure](https://thevisualroom.com/poisson_for_pressure.html).
* **Performance:** Right now it runs with plain Python loops over the grid (no NumPy vectorization), so it's a bit clanky. Looking forward to rewriting this in C++ later and profiling the speedup.
