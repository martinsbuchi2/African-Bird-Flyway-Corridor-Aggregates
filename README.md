# African Bird Migration — Flyway Corridor Aggregates (Phase 2)

> **Phase 2 of the corridor analysis workflow.** Spatially joins stopover
> sites and species distribution points into the Phase 1 flyway corridor
> polygons, computing per-corridor aggregates: stopover counts, dwell duration,
> unique species counts, dominant habitat, and habitat breakdown.

---

## Project Overview

| Property | Value |
|---|---|
| **Project file** | `African_Bird_Flyway_Corridor_Aggregates.qgz` |
| **CRS** | EPSG:4326 — WGS 84 |
| **Source corridor zones** | 6 polygons (from Phase 1) |
| **Stopovers joined** | 96 |
| **Distribution points joined** | 94 |
| **Output features** | 6 polygons with 14 attribute fields |
| **Spatial-join predicate** | Polygon `contains` Point |

This is **Phase 2** built on top of the foundational corridor zones from
Phase 1. The corridor geometries are unchanged — what's added are aggregate
point-data metrics that turn each corridor polygon into a self-contained
summary record of the bird activity inside it.

## Folder Structure

```
African_Bird_Flyway_Corridor_Aggregates/
├── African_Bird_Flyway_Corridor_Aggregates.qgz   (21.6 KB)
├── README.md
├── Input_layers/
│   ├── flyway_corridor_zones.gpkg          (Phase 1 output, used as input)
│   ├── migration_routes.gpkg               (27 lines · context)
│   ├── stopover_sites.gpkg                 (96 points · spatial-join input)
│   └── species_distribution.gpkg           (94 points · spatial-join input)
└── Output_layer/
    └── flyway_corridor_aggregates.gpkg     (6 polygons · 14 attributes)
```

## Methodology

### Phase 2 — Spatial Join & Aggregation

```
1. Load Phase 1 flyway corridor polygons (6 features)
2. For each stopover point:
     find the corridor whose polygon contains it
     append the stopover to that corridor's bucket
3. For each species distribution point:
     find the corridor whose polygon contains it
     append the point to that corridor's bucket
4. For each corridor, compute:
     n_stopovers, n_dist_pts
     mean_dur_days, total_dwell_days
     n_species_stops, n_species_dist, unique_species (set union)
     dominant_habitat, habitat_breakdown
5. Save 6 polygons to GeoPackage with all attributes attached
```

### Output Attribute Schema

| Field | Type | Description |
|---|---|---|
| `flyway` | String | Flyway name |
| `n_routes` | Integer | Source route lines (carried from Phase 1) |
| `length_km` | Real | Dissolved-route length (carried from Phase 1) |
| `area_deg2` | Real | Buffer polygon area (carried from Phase 1) |
| `n_stopovers` | Integer | Stopover sites contained in this corridor |
| `n_dist_pts` | Integer | Distribution points contained in this corridor |
| `mean_dur_days` | Real | Mean stopover duration (days) |
| `total_dwell_days` | Real | Total dwell across all stopovers (bird-days proxy) |
| `n_species_stops` | Integer | Distinct species at stopover sites |
| `n_species_dist` | Integer | Distinct species at distribution points |
| `unique_species` | Integer | Distinct species (union of stopovers ∪ distribution) |
| `dominant_habitat` | String | Most-common habitat among stopovers |
| `habitat_breakdown` | String | All habitats with counts (e.g. `Wetland:5; Coastal:3`) |
| `species_list` | String | All unique species in the corridor |

## Results

| Flyway | Stops | Dist Pts | Mean Dur | Total Dwell | Unique Spp | Dominant Habitat |
|---|---|---|---|---|---|---|
| **East African Flyway** | 35 | 25 | 8.8 d | 309 d | 10 | Grassland |
| **Central African Flyway** | 24 | 6 | 8.4 d | 202 d | 8 | Agricultural |
| **Mediterranean Flyway** | 18 | 26 | 8.0 d | 144 d | 8 | Coastal |
| Atlantic Flyway | 10 | 5 | 8.1 d | 81 d | 2 | Grassland |
| Sahara Flyway | 6 | 2 | 8.7 d | 52 d | 4 | Wetland |
| West African Flyway | 3 | 3 | 4.7 d | 14 d | 3 | Woodland |

### Key Findings

- **East African Flyway dominates absolute activity** — 35 stopovers, 25
  distribution points, all 10 species use it, and the highest total dwell
  load (309 bird-days). This is consistent with every other corridor
  analysis in the project: East African is the busiest corridor.

- **Mediterranean Flyway has more distribution points than stopovers** —
  unique among the 6 corridors. 26 distribution points vs only 18 stopovers,
  suggesting a corridor where birds are observed more than they actively
  stop to refuel. This may reflect transit behaviour over the Mediterranean.

- **Central African Flyway is grassland-and-agricultural-heavy** — 24
  stopovers but only 6 distribution points. Birds rest there but are
  observed less often outside the rest periods. Agricultural land dominates.

- **West African Flyway has anomalously short dwell** — mean 4.7 days vs
  ~8 days everywhere else. Only 3 stopovers, suggesting a transit-only
  corridor where birds pass through quickly without major refuelling.

- **Habitat dominance varies by corridor** — Grassland (East African,
  Atlantic), Agricultural (Central African), Coastal (Mediterranean),
  Wetland (Sahara), Woodland (West African). No single habitat dominates
  across the whole network.

## How to Reproduce

### PyQGIS Pseudocode

```python
from collections import Counter

# Load Phase 1 corridors and point inputs
zones    = {f['flyway']: f.geometry() for f in zones_lyr.getFeatures()}
stops    = list(stop_lyr.getFeatures())
dist_pts = list(dist_lyr.getFeatures())

# Spatial join: assign each point to its containing corridor
agg = {fw: {'stops':[], 'dist':[], 'habitats':Counter()} for fw in zones}
for s in stops:
    for fw, g in zones.items():
        if g.contains(s.geometry()):
            agg[fw]['stops'].append(s)
            agg[fw]['habitats'][s['habitat']] += 1
            break
for p in dist_pts:
    for fw, g in zones.items():
        if g.contains(p.geometry()):
            agg[fw]['dist'].append(p)
            break

# Compute per-corridor aggregates
for fw, data in agg.items():
    n_stops    = len(data['stops'])
    mean_dur   = sum(s['duration_d'] for s in data['stops']) / n_stops
    spp_stops  = set(s['species'] for s in data['stops'])
    spp_dist   = set(p['species'] for p in data['dist'])
    unique_spp = spp_stops | spp_dist
    dom_hab    = data['habitats'].most_common(1)[0][0]
```

### Critical Replication Notes

- **Use `polygon.contains(point)` not `point.within(polygon)`** — semantically
  identical, but the polygon-side predicate is faster when iterating points
  against a small polygon set (6 polygons × 96+94 points = ≤1140 tests).
- **Break out of the inner loop after first match.** Phase 1 corridors are
  buffered routes that can overlap slightly at flyway transitions; without
  the early break a point near a boundary could be counted twice. The first-
  match rule respects polygon ordering — if you need a deterministic tie-
  breaker, sort `zones` by flyway name first.
- **Unique species is a set union, not a sum.** Adding `n_species_stops +
  n_species_dist` would overcount species that appear in both inputs.
- **Habitat breakdown is computed from stopovers only**, not distribution
  points. Distribution points lack the habitat field.

## File Inventory

| File | Folder | Size | Description |
|---|---|---|---|
| `African_Bird_Flyway_Corridor_Aggregates.qgz` | Root | 21.6 KB | QGIS project |
| `README.md` | Root | — | This file |
| `flyway_corridor_zones.gpkg` | `Input_layers/` | 144.0 KB | Phase 1 output (used as input) |
| `migration_routes.gpkg` | `Input_layers/` | 104.0 KB | 27 route lines (context) |
| `stopover_sites.gpkg` | `Input_layers/` | 116.0 KB | 96 stopover sites (join input) |
| `species_distribution.gpkg` | `Input_layers/` | 112.0 KB | 94 distribution points (join input) |
| `flyway_corridor_aggregates.gpkg` | `Output_layer/` | 144.0 KB | 6 polygons with 14 attributes |

---

*African Bird Migration — Flyway Corridor Aggregates (Phase 2)*
*CRS: EPSG:4326  ·  Spatial-join: polygon contains point*
*Built on Phase 1 corridor zones (50 km buffered, dissolved by flyway)*
*QGIS 3.40.14-Bratislava  ·  PyQGIS spatial-join + aggregation pipeline*