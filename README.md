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

## The Problem

Plant diseases cause up to **20% of global produce loss**. Conventional drone spraying applies pesticide uniformly across the entire field — the same amount everywhere, regardless of where disease actually is. This wastes resources, harms the environment, and promotes pesticide resistance.

The challenge is that **disease is never uniform**. It clusters around infection sources and spreads outward. Treating every square meter the same ignores the spatial reality of how disease behaves.

Solving this properly means answering three questions at once:

- **Where to spray?** — Disease clusters spatially. The locations most likely to spread infection to neighbors (hotspots) need to be identified, not just the locations where disease is visible.
- **How much to spray?** — Dosage should be proportional to infection intensity. Hotspots need higher concentration; peripheral low-risk areas need less.
- **How to get there?** — A drone visiting all diseased locations must do so efficiently. This is the Traveling Salesman Problem — solving it poorly wastes battery and time.

Prior approaches treat these questions independently or skip some entirely. SprayCraft is the first unified framework that solves all three using graph theory.

---

## Approach

SprayCraft runs as a two-stage pipeline: first detect where disease is most dangerous, then compute the optimal drone route and adapt spray dosage accordingly.

### Stage 1 — Hotspot Detection via Graph Message Passing

Every diseased instance detected in the field becomes a **node** in a spatial graph. Two nodes are connected by an **edge** if they fall within each other's pathogen dispersal radius — i.e., if one could realistically spread disease to the other. Edge weights are set to the inverse distance between nodes, so closer neighbors exert stronger influence.

Each node starts with a feature value equal to the bounding box area of its disease instance. **Message passing** then runs iteratively: at each step, every node updates its feature by aggregating the features of its neighbors, scaled by how close they are:

```
h_u^(k+1) = UPDATE( h_u^(k),  AGG( { h_v^(k) · 1/W(u,v) : v ∈ N(u) } ) )
```

After several iterations, nodes that sit at the center of dense, heavily-infected clusters accumulate much higher values than isolated peripheral nodes. These high-value nodes are the **hotspots** — the locations most likely to drive further spread across the field.

The final feature values are normalized to produce a **hotspot probability score** between 0 and 1 for each node. This score is passed directly to Stage 2 to control spray dosage.

### Stage 2a — Near-Optimal Route via Christofides TSP

Visiting all diseased locations is the Traveling Salesman Problem. SprayCraft uses the **Christofides algorithm**, which guarantees a tour within **1.5× of optimal** in polynomial time:

1. Compute the **Minimum Spanning Tree (MST)** of the location graph
2. Find all **odd-degree nodes** in the MST
3. Compute **minimum-weight perfect matching** on those odd-degree nodes
4. Merge the MST and matching into an **Eulerian circuit**
5. Shortcut repeated nodes into a **Hamiltonian circuit**

The result is a single drone flight path that visits every diseased location once, with a provable bound on how far it is from the true optimum.

### Stage 2b — Boustrophedon Coverage Paths

At each stop on the tour, the drone doesn't just fly over a point — it executes a **serpentine (boustrophedon) sweep** across the entire diseased area. Parallel passes are spaced at twice the spray radius so every part of the area is covered with no gaps or redundant overlap.

The spray radius at each location comes from that node's hotspot probability score. How this translates to actual dosage depends on the drone hardware:

**Variable Rate Sprayer (`VRS_BuiltIn = True`)** — For drones with PWM/PID flow control, the hotspot score directly modulates the nozzle flow rate. The drone flies at constant altitude; hotspots receive more pesticide, peripheral areas receive less.

**Constant Rate Sprayer (`VRS_BuiltIn = False`)** — For drones with a fixed flow rate, dosage is controlled by adjusting **flight altitude**. A lower altitude concentrates the conical spray cone on a smaller area (denser application); a higher altitude disperses it. At hotspots, the drone descends to increase concentration.

Finally, all meter-based waypoints are converted to GPS lat/long coordinates via the **Haversine formula**, producing a drone-ready flight plan.

---

## Results

Evaluated on a synthetic farmland with **36 diseased locations** in a 250×250 m field using an apple leaf disease dataset (Black Rot, Cedar Apple Rust, Apple Scab).

| Metric | Value |
|---|---|
| Primary hotspots detected | Nodes 14 and 15 (highest aggregated features after message passing) |
| TSP tour length | 1,056.13 m through all 36 locations |
| Total boustrophedon path | ~3,254.7 m |
| Flight altitude range (constant sprayer) | 3.0 m – 4.8 m |
| Spray radius range (variable sprayer) | 1.0 m – 2.0 m |
| TSP approximation bound | ≤ 1.5× optimal |

After two rounds of message passing, nodes 14 and 15 accumulated significantly higher feature values than all other nodes, correctly identifying the densest disease clusters. The Christofides algorithm then produced a tour that naturally clusters spatially adjacent nodes, minimizing unnecessary back-and-forth across the field.

### Comparison with Prior Work

| Approach | Spatial Analysis | Hotspot Detection | Variable Rate | Route Optimization |
|---|---|---|---|---|
| Traditional Uniform Spraying | None | None | Fixed | None |
| Grid-based Coverage | Grid only | None | Fixed | Boustrophedon only |
| ML Disease Severity (no routing) | Local only | Per-image | Limited | None |
| **SprayCraft (Ours)** | **Graph spatial** | **Message passing** | **Probabilistic** | **TSP + Boustrophedon** |

---

## Installation & Usage

```bash
git clone https://github.com/kirankkethineni/SprayCraft.git
cd SprayCraft
pip install pandas matplotlib networkx numpy tensorflow jupyter
jupyter notebook SprayCraft.ipynb
```

Run all cells in order. Key parameters to configure:

```python
intensity_factor = 4        # Spray intensity scaling factor
crop_height      = 1        # Crop height in meters
delta_altitude   = 4        # Base drone altitude above crop top
VRS_BuiltIn      = True     # True  = variable-rate sprayer (PWM/PID)
                            # False = constant-rate sprayer (altitude-controlled)
```

---

## Citation

```bibtex
@article{kethineni2024spraycraft,
  title   = {SprayCraft: Graph-Based Route Optimization for Variable Rate Precision Spraying},
  author  = {Kethineni, Kiran K. and Mohanty, Saraju P. and Kougianos, Elias
             and Bhowmick, Sanjukta and Rachakonda, Laavanya},
  journal = {arXiv preprint arXiv:2412.12176},
  year    = {2024},
  url     = {https://arxiv.org/abs/2412.12176}
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
