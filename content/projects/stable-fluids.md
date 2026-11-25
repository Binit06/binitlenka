---
title: "Stable Fluids"
description: "From-scratch Python implementation of Jos Stam's landmark SIGGRAPH paper on unconditionally stable fluid simulation."
date: "2024-11-01"
tags: ["python", "graphics", "math", "simulation"]
---

# Stable Fluids: Navier-Stokes Simulation

An implementation of Jos Stam's landmark 1999 paper *Stable Fluids* from first mathematical principles.

### Key Highlights
- **Unconditional Stability**: Uses semi-Lagrangian advection (backtracing along velocity streamlines) to ensure the simulation never explodes, regardless of the time step size (Δt).
- **Pressure Projection**: Enforces incompressibility (∇ · u = 0) using Helmholtz-Hodge vector field decomposition.
- **Tech Stack**: Python, NumPy.
- **Repository**: [github.com/Binit06/stable-fluids](https://github.com/Binit06/stable-fluids)

### Engineering Deep Dives:
- [[blog/stable-fluids-documentation|Simulation video & handwritten math walkthrough]]