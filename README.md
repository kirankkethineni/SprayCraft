# SprayCraft: Graph-Based Route Optimization for Variable Rate Precision Spraying

<div align="center">

[![arXiv](https://img.shields.io/badge/arXiv-2412.12176-b31b1b.svg)](https://arxiv.org/abs/2412.12176)
[![Project Page](https://img.shields.io/badge/Project-Page-blue)](https://kirankkethineni.github.io/SprayCraft/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)](https://python.org)

**[Project Page](https://kirankkethineni.github.io/SprayCraft/) | [Paper (arXiv)](https://arxiv.org/abs/2412.12176) | [Notebook](SprayCraft.ipynb)**

*Kiran K. Kethineni, Saraju P. Mohanty, Elias Kougianos, Sanjukta Bhowmick, Laavanya Rachakonda*  
University of North Texas &nbsp;|&nbsp; University of North Carolina Wilmington

</div>

---

## Overview

**SprayCraft** is an Agriculture Cyber-Physical System (A-CPS) framework for precision disease management. It combines **graph-based spatial analysis** with **TSP route optimization** to enable drones to spray pesticides at exactly the right dosage, in exactly the right places, via the shortest possible path.

Plant diseases cause up to **20% of global produce loss**. Traditional uniform spraying wastes pesticides, harms the environment, and promotes resistance. SprayCraft solves this with a two-stage pipeline:

1. **Spatial Analysis** — models diseased field locations as a graph; uses message passing to detect infection hotspots and compute variable spray dosages
2. **Route Optimization** — generates near-optimal drone flight paths via Traveling Salesman Problem (TSP) with dense boustrophedon (serpentine) local coverage patterns

---

## The Problem

Precision agriculture requires answering three questions simultaneously:

| Question | Challenge |
|---|---|
| **Where to spray?** | Disease is spatially heterogeneous — hotspots cluster near infection sources |
| **How much to spray?** | Dosage must be proportional to infection intensity, not uniform |
| **How to get there?** | Drone paths must minimize travel while ensuring full coverage |

Classical approaches treat these independently or assume uniform fields. SprayCraft is the first unified framework that solves all three using graph theory.

---

## Approach

### Stage 1 — Hotspot Detection via Graph Message Passing

Each diseased location in the field becomes a **graph node**. Nodes within a distance threshold (based on pathogen dispersal radius) are connected with **weighted edges** (inverse distance).

```
Graph G = (V, E, F)
  V: diseased locations (nodes)
  E: edges between nodes within dispersal radius
  F: node feature = area of diseased instance
```

**Message passing** is applied iteratively to update each node's features using its neighbors':

```
h_u^(k+1) = UPDATE( h_u^(k), AGG({ h_v^(k) · 1/W(u,v) : v ∈ N(u) }) )
```

After convergence, nodes with higher aggregated features are **hotspots** — they are more likely to spread infection to their neighbors. Normalized feature values serve as **hotspot probability scores**.

### Stage 2a — TSP Route Computation (Christofides Algorithm)

The Christofides algorithm is applied over the graph of diseased locations to find a near-optimal Hamiltonian circuit:

1. Compute Minimum Spanning Tree (MST)
2. Find odd-degree nodes in MST
3. Compute minimum-weight perfect matching on odd-degree nodes
4. Form Eulerian circuit → convert to Hamiltonian circuit

This guarantees a route within **1.5× of optimal** — balancing accuracy and computation for real-time field deployment.

### Stage 2b — Boustrophedon Coverage Paths

At each diseased location, a **serpentine (boustrophedon) path** ensures 100% area coverage with no gaps or overlaps. Path spacing is set to twice the spray coverage radius.

**Variable Rate Sprayer (VRS):** The hotspot probability score modulates flow rate in real time — hotspots receive higher dosage.

**Constant Rate Sprayer:** Flight altitude is adjusted to control the effective cone coverage area — lower altitude over hotspots for denser application.

```
Coverage Radius  = f(hotspot_probability)
Spray Area       = π × R²
Flight Altitude  = g(Coverage Radius, Crop Height)
```

---

## Results

Evaluated on a synthetic farmland with **36 diseased locations** in a 250×250 m area (apple leaf disease dataset).

| Metric | Value |
|---|---|
| Detected hotspot nodes | Nodes 14 and 15 (highest aggregated features) |
| TSP tour cost | **1,056.13 meters** |
| Total boustrophedon path length | **~3,254.7 meters** |
| Coverage altitude range (constant sprayer) | 3.0 m – 4.8 m |
| Coverage radius range (variable sprayer) | 1.0 m – 2.0 m |

SprayCraft's variable-rate mode concentrates pesticide application in high-probability hotspot zones while reducing dosage in peripheral low-risk areas — cutting waste while protecting yield.

---

## Repository Structure

```
SprayCraft/
├── SprayCraft.ipynb      # Full implementation notebook
└── README.md             # This file
```

### Notebook Sections

| Section | Description |
|---|---|
| Data Loading | Load annotated apple leaf disease dataset with bounding boxes |
| Graph Construction | Build spatial graph with nodes = diseased instances, edges = proximity |
| Message Passing | Iterative neighbor aggregation for hotspot detection |
| Route Optimization | Christofides TSP for near-optimal tour |
| Boustrophedon Path | Serpentine coverage path per diseased location |
| Variable Rate Logic | Altitude / flow-rate adaptation by hotspot probability |
| GPS Conversion | Haversine-based pixel → GPS waypoint conversion |
| Visualization | 3D flight altitude maps, spray heat maps, path overlays |

---

## Installation & Usage

### Requirements

```bash
pip install pandas matplotlib networkx numpy tensorflow
```

### Run the Notebook

```bash
git clone https://github.com/kirankkethineni/SprayCraft.git
cd SprayCraft
jupyter notebook SprayCraft.ipynb
```

### Key Parameters

```python
intensity_factor = 4        # Controls spray intensity scaling
crop_height      = 1        # Crop height in meters
delta_altitude   = 4        # Base flight altitude above crop
VRS_BuiltIn      = True     # True = variable rate sprayer, False = constant rate
```

Set `VRS_BuiltIn = True` for drones with PWM/PID flow control. Set to `False` for drones that control dosage through altitude adjustment.

---

## Key Contributions

- **Graph-based hotspot detection:** Novel spatial analysis using message passing to identify disease clusters and estimate infection intensity
- **Probabilistic variable-rate spraying:** Hotspot probability directly drives pesticide dosage — first graph-theoretic approach to variable-rate A-CPS
- **Dual sprayer support:** Works with both variable-rate (PWM/PID) and constant-rate spraying systems
- **Near-optimal routing:** Christofides TSP integration guarantees 1.5× approximation with boustrophedon local coverage

---

## Citation

If you use SprayCraft in your research, please cite:

```bibtex
@article{kethineni2024spraycraft,
  title     = {SprayCraft: Graph-Based Route Optimization for Variable Rate Precision Spraying},
  author    = {Kethineni, Kiran K. and Mohanty, Saraju P. and Kougianos, Elias and Bhowmick, Sanjukta and Rachakonda, Laavanya},
  journal   = {arXiv preprint arXiv:2412.12176},
  year      = {2024},
  url       = {https://arxiv.org/abs/2412.12176}
}
```

---

## Authors

| Author | Affiliation |
|---|---|
| Kiran K. Kethineni | University of North Texas |
| Saraju P. Mohanty | University of North Texas |
| Elias Kougianos | University of North Texas |
| Sanjukta Bhowmick | University of North Texas |
| Laavanya Rachakonda | University of North Carolina Wilmington |

---

<div align="center">
  <a href="https://kirankkethineni.github.io/SprayCraft/">Project Page</a> &nbsp;|&nbsp;
  <a href="https://arxiv.org/abs/2412.12176">Paper</a> &nbsp;|&nbsp;
  <a href="SprayCraft.ipynb">Code Notebook</a>
</div>
