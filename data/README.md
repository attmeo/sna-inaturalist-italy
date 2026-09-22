DATA DIRECTORY
==============

All tabular and graph files are gzip-compressed (.csv.gz, .graphml.gz).
pandas (read_csv) and networkx (read_graphml) decompress them transparently
from the file extension, so no manual step is needed.
JSON summaries are stored uncompressed.

Row counts below are exact (len(pd.read_csv(...)) for CSV, <node>/<edge>
elements for GraphML). Producing notebooks are in notebooks/.

Structure:
  data/raw/       crawled from the iNaturalist API by notebook 00
                  (never modified downstream)
  data/networks/  co-occurrence networks, backbone and seasonal layers
                  (notebooks 01, 02, 04)
  data/analysis/  aggregated results consumed by the report
                  (notebooks 03, 04, 05)


------------------------------------------------------------------------
1. data/raw/  -  crawled observations (notebook 00_crawl_inaturalist)
------------------------------------------------------------------------

observations.csv.gz  (631,778 rows)
  Research-grade animal observations for calendar year 2025 (first crawl
  phase), after removing non-animal records. One row per observation.

  Columns:
  - obs_id (int): iNaturalist observation ID (unique)
  - taxon_id (int): iNaturalist taxon ID of the identified species;
    this is the NODE ID of every network
  - species_name (str): scientific name
  - common_name (str): preferred common name (may be empty)
  - iconic_taxon (str): iNaturalist "iconic" group
    (Insecta, Aves, Reptilia, Mammalia, ...)
  - latitude, longitude (float): observation coordinates (WGS84)
  - grid_cell (str): "lat,lng" of the bottom-left corner of the
    0.1 x 0.1 degree cell containing the observation
  - observed_on (str): observation date, YYYY-MM-DD
  - month (int): month number (1-12)
  - season (str): season label; in this file the 4-label scheme
    winter/spring/summer/autumn (superseded by the 5-snapshot labels
    of observations_extended.csv.gz)
  - place_guess (str): free-text locality provided by the observer
  - positional_accuracy (float): reported GPS accuracy in metres
    (may be empty)

observations_supplementary.csv.gz  (80,573 rows)
  Second crawl phase: December 2024, January 2026 and February 2026,
  needed to complete the two meteorological winters.
  Same columns as above.

observations_extended.csv.gz  (681,813 rows)
  THE WORKING DATASET. Union of the two crawls, de-duplicated on obs_id,
  restricted to animals (iconic_taxon not in {Plantae, Fungi, Chromista,
  Protozoa, empty}) and with season reassigned to the 5-snapshot scheme:
    winter_0 = Dec 2024 - Feb 2025
    spring, summer, autumn = 2025
    winter_1 = Dec 2025 - Feb 2026
  Same columns as above. Covers 12,592 species and 6,475 grid cells.

species.csv.gz  (9,992 rows)
  Species-level metadata collected during the 2025 crawl (one row per
  taxon_id seen in the first phase). Species first seen in the
  supplementary crawl are not listed: notebook 01 falls back to the
  observation records for their name and iconic taxon.

  Columns:
  - taxon_id: iNaturalist taxon ID
  - species_name: scientific name
  - common_name: preferred common name
  - rank: taxonomic rank of the identification
    (species, subspecies, variety, form)
  - iconic_taxon: iNaturalist iconic group
  - observations_count: global iNaturalist observation count for the
    taxon at crawl time
  - threatened: iNaturalist "threatened" flag

checkpoint.json, crawl_stats.json
  Crawl bookkeeping: last observation ID reached (last_id, for resuming)
  and crawl-level totals (observations, species, grid cells,
  iconic-taxon counts, timestamp).


------------------------------------------------------------------------
2. data/networks/  -  networks and layers
------------------------------------------------------------------------

All GraphML files share the same attributes.
Node ID = taxon_id (as a string).

Node attributes:
  - species_name, common_name, iconic_taxon, rank, threatened:
    copied from species.csv.gz (or from the observations as fallback)
  - observations: number of observations of the species in the subset
    the network was built from
  - grid_cells: number of distinct grid cells in which the species was
    observed in that subset

Edge attributes:
  - weight: raw co-occurrence count = number of grid cells containing
    both species
  - jaccard: |cells(i) AND cells(j)| / |cells(i) OR cells(j)|
    -> the REFERENCE WEIGHT of the analysis
  - simpson: |cells(i) AND cells(j)| / min(|cells(i)|, |cells(j)|)
  - pmi: pointwise mutual information, log10(p_ij / (p_i * p_j));
    auxiliary, unbounded

The *.edgelist.csv.gz files contain the same edges as the matching
GraphML, with columns: source, target, weight, jaccard, simpson, pmi.


2a. Raw co-occurrence networks (notebook 01_build_network)

  Built from observations_extended.csv.gz with MIN_OBS = 3 and
  MIN_COOCCURRENCE = 2. The annual network uses calendar year 2025 only;
  each seasonal network uses the observations of that snapshot.

  File (graphml.gz + edgelist.csv.gz)   Nodes     Edges
  annual                                6,977     1,783,415
  winter_0                              1,223        44,966
  spring                                4,076       491,800
  summer                                5,417       962,476
  autumn                                3,230       253,322
  winter_1                              1,268        48,200

  - network_stats.json: sizes, densities and parameters of the six
    networks (keys: annual, seasonal, parameters)
  - seasonal_comparison.json: per-season observation, species, node and
    edge counts


2b. Backbone and seasonal layers (notebook 02_backbone)

  annual_backbone.graphml.gz
    3,023 nodes, 134,305 edges
    Disparity Filter (alpha = 0.05) on annual.graphml.gz. Its giant
    component (3,009 nodes) is the graph analysed in notebooks 03-06.

  winter_0_layer (.graphml.gz / .edgelist.csv.gz)
    3,009 nodes, 40,757 edges
    induced subgraph of winter_0 on the backbone giant-component node set

  spring_layer
    3,009 nodes, 144,444 edges
    induced subgraph + Disparity Filter alpha = 0.30

  summer_layer
    3,009 nodes, 178,998 edges
    induced subgraph + Disparity Filter alpha = 0.20

  autumn_layer
    3,009 nodes, 223,590 edges
    induced subgraph

  winter_1_layer
    3,009 nodes, 43,119 edges
    induced subgraph

  Every layer shares the same 3,009-node set; species absent from a
  season are kept as isolated nodes. jaccard is RECOMPUTED on each layer
  (neighbourhood Jaccard), so it is NOT comparable with the grid-cell
  Jaccard of the raw networks.

  - backbone_summary.json: alpha, original vs backbone sizes (key:
    annual) and per-season strategy, alpha, edge retention,
    giant-component coverage and isolates (key: seasonal)


2c. Community assignments (3,009 rows each, one per backbone
    giant-component species)

  community_assignments_annual_v2.csv.gz
    Produced by: 04_community_detection
    Columns: taxon_id, species_name, iconic_taxon, louvain_community,
    leiden_community, greedy_community, infomap_community, lpa_community
    Community label of each species under the five algorithms on the
    annual backbone.

  community_assignments_seasonal_v2.csv.gz
    Produced by: 04_community_detection
    Columns: taxon_id, species_name, iconic_taxon, annual_louvain,
    annual_infomap, {season}_louvain, {season}_infomap for the five
    seasons. -1 marks a species not active in that season.

  community_assignments_annual.csv.gz
    Produced by: first-generation community detection
    (not included in this repository)
    Columns: taxon_id, species_name, iconic_taxon, louvain_community,
    leiden_community, infomap_community, present_winter_0, ...,
    present_winter_1
    First-generation partition; its louvain_community is the partition
    mapped in notebook 06.

  community_assignments_seasonal.csv.gz
    Produced by: first-generation community detection
    (not included in this repository)
    Columns: taxon_id, species_name, iconic_taxon, annual_community,
    {season}_community
    First-generation seasonal Louvain partitions (-1 = absent), used by
    notebook 06.


------------------------------------------------------------------------
3. data/analysis/  -  aggregated results used by the report
------------------------------------------------------------------------

structural_metrics.json
  Produced by: 03_structural_analysis
  Keys: backbone (sizes), real (metrics of the giant component),
  path_distribution_pct, null_ER / null_BA (mean +/- std over 10
  realizations), small_world (sigma), degree_fit (power-law fit and
  likelihood-ratio tests), centrality_correlation, params

centrality_v2.csv.gz  (3,009 rows)
  Produced by: 03_structural_analysis
  Columns: taxon_id, degree, betweenness, closeness, eigenvector,
  species_name, iconic_taxon (four centralities per species)

community_results_v2.json
  Produced by: 04_community_detection
  Keys: graph, gamma_opt, sweep, algorithms (k, modularity, size stats),
  infomap_codelength, lpa_variability, nmi, ari, purity, baseline,
  stratified, null_models, seasonal, cross_season_nmi

dynamic_results_v2.json
  Produced by: 05_dynamic_communities
  Keys: params (theta, tau, minimum size), communities_per_season,
  events (per algorithm x matching criterion), net_balance,
  edge_jaccard (5 x 5 layer overlap), winter (Winter_0 <-> Winter_1
  recurrence and baseline), theta_sweep