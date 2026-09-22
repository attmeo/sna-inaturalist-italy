# Seasonal Dynamics of Animal Species Co-occurrence Networks in Italy

**Matteo Trivelli** — Final project for the *Social Network Analysis* course

This repository contains the **analysis code and the dataset only**

---

## Pipeline

Run the notebooks in [`notebooks/`](notebooks/) **in numerical order**. Each one starts with a
cell that locates the repository root, so it can be launched either from the repository root
or from `notebooks/`.

| # | Notebook | What it does | Report | Input | Output |
|---|---|---|---|---|---|
| 00 | `00_crawl_inaturalist.ipynb` | Crawls the API in two phases, assigns grid cells and 5-snapshot seasons, merges and de-duplicates | §2.1–2.2 | iNaturalist API | `data/raw/observations*.csv.gz`, `species.csv.gz` |
| 01 | `01_build_network.ipynb` | Applies MIN_OBS; builds the annual network (calendar 2025) and the 5 seasonal networks with 4 edge weights (count, Jaccard, Simpson, PMI) | §2.4 | `data/raw/` | `data/networks/{annual,winter_0,…,winter_1}.graphml.gz` + edge lists, `network_stats.json` |
| 02 | `02_backbone.ipynb` | Disparity Filter with α sweep on the annual network; builds the 5 seasonal layers on the fixed backbone node set, recomputes Jaccard | §2.5–2.6 | `data/networks/*.graphml.gz` | `annual_backbone.graphml.gz`, `*_layer.graphml.gz`, `backbone_summary.json` |
| 03 | `03_structural_analysis.ipynb` | Structural metrics vs 10 ER + 10 BA null models, exact all-pairs distances, `powerlaw` degree fits, four centralities | §3 | `annual_backbone.graphml.gz` | `data/analysis/structural_metrics.json`, `centrality_v2.csv.gz`, `figures/struct_*` |
| 04 | `04_community_detection.ipynb` | Five algorithms from three objective families (Louvain, Leiden, Greedy Modularity, Infomap, Label Propagation), NMI/ARI, taxonomic purity, stratified analysis, dual-criterion null test, seasonal CD with Louvain and Infomap | §4, §5.3 | `annual_backbone.graphml.gz`, `*_layer.graphml.gz` | `community_assignments_*_v2.csv.gz`, `data/analysis/community_results_v2.json`, `figures/cd2_*` |
| 05 | `05_dynamic_communities.ipynb` | Community life-cycle events under symmetric and asymmetric matching on both partitionings, edge overlap between layers, Winter₀ ↔ Winter₁ recurrence with a random baseline | §5.2, §5.4–5.5, §5.7 | `community_assignments_seasonal_v2.csv.gz`, `*_layer.graphml.gz` | `data/analysis/dynamic_results_v2.json`, `figures/dyn2_*` |
| 06 | `06_geographic_communities.ipynb` | Maps each grid cell to its dominant community, per-season maps, fraction of cells changing dominant community between snapshots | §5.6 | `data/raw/observations_extended.csv.gz`, `community_assignments_{annual,seasonal}.csv.gz` (first-generation Louvain partition, shipped in `data/networks/`) | `figures/geo_*` |



---

## Key parameters

All values are read from the notebooks' configuration cells.

| Parameter | Value | Where | Meaning |
|---|---|---|---|
| Bounding box | 35.5–47.1 °N, 6.6–18.5 °E | 00 | crawl area |
| `PER_PAGE`, `DELAY_SECONDS` | 200, 1.1 s | 00 | API page size and pause between requests |
| `GRID_CELL_SIZE` / `GRID_SIZE` / `CELL_SIZE` | 0.1° | 00, 01, 06 | grid cell ≈ 11 × 8 km at 42 °N |
| `MIN_OBS` | 3 | 01 | species with fewer observations are removed |
| `MIN_COOCCURRENCE` | 2 | 01 | an edge needs co-occurrence in at least 2 cells |
| `ALPHA` | 0.05 | 02 | Disparity Filter significance level, annual backbone |
| `SEASON_CONFIG` | Winter₀, Autumn, Winter₁: induced subgraph, no filter; Spring: induced + α = 0.30; Summer: induced + α = 0.20 | 02 | seasonal layer construction |
| `WEIGHT` | `weight` (raw count) | 02 | weight the Disparity Filter is applied to |
| `WEIGHT_ATTR` | `jaccard` | 04, 06 | weight used by the community detection algorithms |
| `N_NULL` | 10 | 03, 04 | ER and BA realizations per null model |
| `BETWEEN_K` | 500 | 03 | pivots for the approximate betweenness |
| `GAMMAS` → `gamma_opt` | [0.75, 0.875, 1.0, 1.125, 1.25, 1.5, 2.0] → **1.0** | 04 | Louvain resolution sweep on the annual backbone and selected value |
| `GAMMAS_SEASONAL` → per-season `gamma_opt` | [0.5, 0.75, 1.0, 1.25, 1.5, 2.0] → Winter₀ 2.0, Spring 2.0, Summer 2.0, Autumn 1.25, Winter₁ 1.5 | 04 | per-season Louvain resolution sweep |
| `N_RUNS`, `N_RUNS_LEIDEN`, `N_RUNS_SWEEP`, `INFOMAP_TRIALS` | 50, 10, 10, 10 | 04 | repetitions of the stochastic algorithms |
| `MIN_COMMUNITY_SIZE`, `PURITY_THRESHOLD`, `PURITY_NORM_THRESHOLD` | 10, 0.70, 0.30 | 04, 06 | minimum community size for purity and the purity significance thresholds |
| `MIN_SIZE_FILTER` | 5 | 04, 06 | communities below this size count as noise in the seasonal partitions |
| `THETA_J` (θ) | 0.15 | 05 | symmetric Jaccard matching threshold |
| `TAU_ASYM` (τ) | 0.50 | 05 | asymmetric matching threshold on max(F, B) |
| `MIN_COMM_SIZE` | 5 | 05 | communities smaller than this are excluded from the event analysis |
| `SIZE_CHANGE_THRESHOLD` | ±50 % | 05 | growth / shrink classification |
| `RANDOM_SEED` | 42 | 03, 04, 06 | seed for null models and stochastic algorithms (`cdlib` does not expose a seed for Leiden) |

---

## Reproducing

```bash
git clone <this repository>
cd <repository>
python -m venv .venv && source .venv/bin/activate      # or: conda create -n sna python=3.12
pip install -r requirements.txt
jupyter lab

---

## Repository structure

```
├── README.md
├── requirements.txt
├── notebooks/                       # the pipeline, in execution order (00 → 06)
└── data/
    ├── README.md                    # every file: producer, columns, row count
    ├── raw/                         # crawled observations and species table (00)
    ├── networks/                    # networks, backbone, seasonal layers, community assignments
    └── analysis/                    # JSON / CSV results consumed by the report
```

Running the notebooks recreates `figures/` (PNG 300 dpi + PDF) next to `notebooks/`.

---

## Data source and licence

Observations from [iNaturalist](https://www.inaturalist.org), research-grade only, retrieved
through the public API v1. Individual observations carry their own licences chosen by their
observers; the CSV files in `data/raw/` contain observation ids, taxon ids, coordinates and
dates only, and are provided for reproducibility of the analysis.
