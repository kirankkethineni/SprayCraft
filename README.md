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

Plant diseases cause up to **20% of global produce loss** every year. The standard response — sending a drone to spray pesticide uniformly across the entire field — is blunt and wasteful. It applies the same dose to a healthy corner of the field as it does to a heavily infected cluster. It visits locations in whatever order happens to be convenient. And it treats every infection site as equally urgent, even though some are far more dangerous than others.

The reason this matters goes beyond waste. Uniform over-spraying promotes pesticide resistance, contaminates soil and water, and increases cost. But the reason precision spraying is hard is that fixing it properly requires solving three problems that are deeply entangled with each other:

**Where is the real danger?** Not every diseased location is equally threatening. A single infected plant in an isolated corner of the field is very different from a cluster of infected plants at the center, surrounded by more infection. The isolated case might resolve on its own; the cluster is actively spreading. Any system that can't distinguish these two situations will misallocate resources — under-treating the dangerous zones and over-treating the safe ones.

**How much pesticide should each location get?** Once you know which locations are hotspots, dosage should scale accordingly. A hotspot at the center of a dense cluster warrants a higher concentration than a peripheral case at the edge of the disease boundary. Getting this right requires a continuous, spatially-grounded estimate of infection intensity — not a binary sick/healthy label.

**What order should the drone visit them in?** The drone has limited battery and time. Visiting 36 diseased locations in the wrong order could mean flying back and forth across the field repeatedly, doubling or tripling the total flight distance. Finding an efficient route through all locations is the Traveling Salesman Problem — one of the most studied hard problems in computer science.

The critical insight behind SprayCraft is that these three problems cannot be solved well in isolation. The route depends on which locations need treatment. The dosage depends on a spatial analysis that requires knowing each location's relationship to its neighbors. Solving them sequentially with independent methods throws away the information each one needs from the others. SprayCraft solves them as a unified pipeline.

---

## Why Model Disease as a Graph?

The foundation of SprayCraft is representing the diseased field as a **spatial graph** — and this choice is not arbitrary. It reflects something true about how disease actually works.

Disease spreads through physical proximity. Fungal spores travel on wind. Bacterial infections spread through water splash and insect movement. A diseased plant doesn't exist in isolation — it exists in a neighborhood of other plants, and whether it becomes a major outbreak depends heavily on what surrounds it. Two disease instances with identical individual severity can have completely different epidemiological importance depending on their spatial context.

A graph naturally encodes this. Each diseased instance becomes a **node**. Two nodes are connected by an **edge** if they fall within each other's pathogen dispersal radius — the distance within which one infection can realistically spread to another. The edge weight is the inverse distance between them: closer nodes are more strongly connected because proximity means higher transmission probability. This transforms the raw list of disease detections into a relational structure that captures how infections influence each other.

---

## Stage 1 — Finding Hotspots Through Message Passing

With the graph constructed, the goal is to identify which nodes are true hotspots — the locations driving spread across the field — versus which are peripheral cases that are less likely to propagate. The key intuition is this: **a node is a hotspot not just because of its own infection severity, but because of the cumulative infection pressure bearing down on it from all sides.**

This is exactly what **graph message passing** computes. Each node starts with a feature value equal to the area of its detected disease instance — a measure of how severely that location is infected. Message passing then runs iteratively. At each step, every node sends its current feature value to all of its neighbors, weighted by how close they are. Each node then updates its own feature by aggregating everything it received:

```
h_u^(k+1) = UPDATE( h_u^(k),  AGG( { h_v^(k) · 1/W(u,v) : v ∈ N(u) } ) )
```

What happens over multiple iterations is informative. After the first pass, a node's value reflects not just itself but the severity of its immediate neighbors. After the second pass, it reflects its neighbors' neighbors as well. A node that is individually moderately infected but surrounded on all sides by heavily infected neighbors will accumulate a very high feature value after just a few rounds — because all that infection in its neighborhood keeps flowing into it. An isolated node, however severely infected it is individually, has no neighbors contributing to it and will accumulate much less.

This is how hotspots emerge naturally from the graph structure, without any hand-engineered threshold or rule. The nodes with the highest aggregated feature values after convergence are the ones sitting at the dense core of infection clusters — exactly the locations that are most dangerous for field-wide spread.

These values are then **normalized** to produce a probability score between 0 and 1 for each node, which flows directly into dosage calculation in Stage 2.

---

## Stage 2a — Routing via Christofides TSP

With all diseased locations identified and scored, the drone needs a route. The naive approach — visit locations in a greedy nearest-neighbor order — seems reasonable but can produce routes that are nearly twice the optimal length in the worst case. The problem is NP-hard, meaning there is no known algorithm that finds the exact optimum efficiently for large inputs.

SprayCraft uses the **Christofides algorithm**, which takes a principled middle ground: it cannot guarantee the optimal tour, but it mathematically guarantees the tour it produces is **never more than 1.5× the optimal length**. For real agricultural deployment where you need a solution that scales and runs in the field, a 1.5× approximation bound is extremely strong — it means in the absolute worst case, the drone travels 50% more than the theoretical minimum, and in practice it's usually much closer.

The algorithm works by exploiting the structure of the problem. It first builds a Minimum Spanning Tree — the cheapest connected skeleton linking all locations. The MST alone isn't a valid tour because it's a tree, not a circuit, and some nodes may have odd degree. Christofides identifies those odd-degree nodes and computes the minimum-weight perfect matching between them — essentially the cheapest way to pair them up and add those connections. Combining the MST with this matching creates an Eulerian circuit (a path that uses every edge exactly once), which is then shortcut into a Hamiltonian circuit (a path that visits every node exactly once). The structure of this process is what enforces the 1.5× bound.

On the test field, this produced a tour of **1,056.13 meters** through 36 locations, with a sequence that naturally clusters spatially adjacent nodes together rather than bouncing across the field.

---

## Stage 2b — Coverage Paths and Variable Dosage

Arriving at a diseased location isn't the end — the drone needs to actually treat the full diseased area, not just fly over a single GPS point. At each stop, SprayCraft generates a **boustrophedon (serpentine) coverage path**: a series of parallel passes back and forth across the area, like a lawnmower pattern. The spacing between passes is set to exactly twice the spray radius, so adjacent passes touch each other's coverage edge without overlapping. This guarantees 100% coverage of the area with the minimum number of passes.

The spray radius at each location is determined by that node's **hotspot probability score** from Stage 1. A high-scoring hotspot gets a tighter, more concentrated coverage pattern; a low-scoring peripheral location gets a wider, lighter treatment. This is where the spatial analysis from Stage 1 directly controls the physical behavior of the drone.

How that score translates into actual dosage depends on the drone hardware — and here SprayCraft makes a practical design choice to support two common hardware classes:

**Variable Rate Sprayer (`VRS_BuiltIn = True`)** — Some agricultural drones have electronically controlled nozzles where the flow rate can be adjusted in real time via PWM or PID control. For these drones, the hotspot score directly modulates the flow rate. The drone flies at constant altitude; the nozzle simply delivers more or less pesticide based on the score. This is the most direct translation of the spatial analysis into dosage.

**Constant Rate Sprayer (`VRS_BuiltIn = False`)** — Many agricultural drones have a fixed flow rate and no flow control hardware. For these, SprayCraft uses flight altitude as the control variable instead. A drone sprayer produces a conical spray pattern: the lower it flies, the tighter and denser the cone's footprint on the ground; the higher it flies, the more spread out but diluted. By flying lower over hotspots, the same fixed flow rate is concentrated on a smaller area, effectively increasing the dose per unit area. Over low-risk areas, the drone climbs and the same flow rate covers more ground at lower concentration. The hotspot score maps to a target coverage radius, which maps to a required altitude via the cone geometry.

This dual-mode support means SprayCraft works with the existing fleet of agricultural drones without requiring any hardware upgrades — the adaptation happens entirely in how the route plan is generated.

The final output converts all meter-based waypoints to GPS coordinates via the **Haversine formula**, producing a flight plan that can be uploaded directly to a commercial drone.

---

## Results

Evaluated on a synthetic farmland with **36 diseased locations** in a 250×250 m area using apple leaf disease annotations (Black Rot, Cedar Apple Rust, Apple Scab).

| Metric | Value |
|---|---|
| Primary hotspots detected | Nodes 14 and 15 — highest aggregated features after message passing |
| TSP tour length | 1,056.13 m through all 36 locations |
| Total boustrophedon path | ~3,254.7 m |
| Flight altitude range (constant sprayer) | 3.0 m – 4.8 m |
| Spray radius range (variable sprayer) | 1.0 m – 2.0 m |
| TSP approximation bound | ≤ 1.5× optimal |

After two rounds of message passing, nodes 14 and 15 accumulated significantly higher feature values than all other nodes, correctly identifying the densest disease clusters as primary hotspots. The Christofides tour connected spatially adjacent nodes efficiently, and the boustrophedon paths at each location adapted their altitude and spacing to the hotspot scores — concentrating treatment where it matters most.

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
VRS_BuiltIn      = True     # True  = variable-rate sprayer (PWM/PID flow control)
                            # False = constant-rate sprayer (altitude-controlled dosage)
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
