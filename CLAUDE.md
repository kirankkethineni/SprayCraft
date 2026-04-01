# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

**SprayCraft** is a research implementation (arXiv:2412.12176) of a two-stage pipeline for precision drone spraying in agriculture:

1. **Hotspot detection** — builds a spatial graph of diseased field locations, runs message passing to aggregate neighbor features, and scores nodes by infection probability
2. **Route optimization** — runs the Christofides TSP algorithm over detected hotspots, then generates boustrophedon (serpentine) coverage paths at each stop, with spray intensity modulated by hotspot probability

The entire implementation lives in a single Jupyter notebook: `SprayCraft.ipynb`.

## Running the Notebook

**Install dependencies:**
```bash
pip install pandas matplotlib networkx numpy tensorflow
```

**Launch:**
```bash
jupyter notebook SprayCraft.ipynb
```

**Run all cells in order** — cells are stateful and must execute sequentially.

**Key configurable parameters (set near top of notebook):**
```python
intensity_factor = 4        # Spray intensity scaling
crop_height      = 1        # Crop height in meters
delta_altitude   = 4        # Base drone altitude above crop
VRS_BuiltIn      = True     # True = variable rate sprayer (PWM/PID flow control)
                            # False = constant rate (altitude controls coverage area)
```

## Architecture: Notebook Cell Flow

The 16 code cells follow this pipeline:

| Cells | Stage | What happens |
|-------|-------|-------------|
| 1 | Setup | Imports (pandas, matplotlib, networkx, numpy, TensorFlow, PIL) |
| 2–3 | Path generation | `generate_drone_boustrophedon_path()` and 3D plot helpers |
| 4 | Graph construction | Loads apple leaf disease dataset, builds spatial graph (nodes = disease instances, edges = proximity within dispersal radius, weights = inverse distance) |
| 5–6 | Disease classifier | Trains/loads a TensorFlow/Keras model on leaf disease images (cell 5 is commented out for training; cell 6 loads saved model) |
| 7 | Visualization | Heat map overlay of disease instances on field image |
| 8–10 | Route optimization | Refines graph → computes MST → Christofides TSP to get Hamiltonian circuit |
| 11–14 | Path + visualization | Generates full boustrophedon flight path; renders 3D altitude views from multiple angles |
| 15 | Metrics | Computes total path length |

## Core Algorithms

**Message passing (hotspot scoring):**
```
h_u^(k+1) = UPDATE( h_u^(k), AGG({ h_v^(k) · 1/W(u,v) : v ∈ N(u) }) )
```
Normalized aggregated features become hotspot probability scores.

**Christofides TSP:** MST → odd-degree node matching → Eulerian circuit → Hamiltonian circuit. Guarantees ≤1.5× optimal tour length.

**Variable-rate logic:**
- `VRS_BuiltIn = True`: hotspot probability drives PWM/PID flow rate
- `VRS_BuiltIn = False`: hotspot probability adjusts flight altitude → changes effective spray cone radius

## GitHub Pages Site

The `docs/` directory contains the project website (`index.html`) served at `https://kirankkethineni.github.io/SprayCraft/`. Images used by the site are in `docs/images/`. The site is static HTML — edit `docs/index.html` directly.
