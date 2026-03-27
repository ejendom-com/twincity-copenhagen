# TwinCity Copenhagen - ejendom.com API

> Ready-to-use building portfolio data from a real PropTech platform.
> For AEC Hackathon 2026 — Track 2 (PropTech product experience).

---

## Authentication

All requests require a single header:

```
X-Hackathon-Key: aec-hackathon-2026-twin-city
```

No signup, no OAuth, no tokens. Just add the header to every request.

**Base URL:**
```
https://dev-admin.ejendom.com/api/v1/hackathon
```

**Example:**
```bash
curl -H "X-Hackathon-Key: aec-hackathon-2026-twin-city" \
  https://dev-admin.ejendom.com/api/v1/hackathon/buildings
```

---

## Endpoints

### 1. List Properties (Portfolios)

```
GET /api/v1/hackathon/properties
```

Returns all property portfolios with summary data. A property is a collection of one or more buildings managed together.

**Response:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "Nørrebro Portfolio",
      "address": "Nørrebrogade 42, 2200 København N",
      "building_count": 12,
      "total_sqm": 34500,
      "total_apartments": 290,
      "energy_scores": ["A", "B", "C", "C", "D"],
      "maintenance_cost_per_sqm_ten_years": 250,
      "center_latitude": 55.6901,
      "center_longitude": 12.5528,
      "main_image": "https://..."
    }
  ]
}
```

**Use case:** Portfolio overview, map clustering, summary cards.

---

### 2. List Buildings

```
GET /api/v1/hackathon/buildings
```

Returns all buildings with coordinates, classification, dimensions, and construction year. This is the main endpoint for populating your 3D world.

**Query parameters** (all optional):

| Parameter | Example | Description |
|---|---|---|
| `property_id` | `?property_id=1` | Filter to a single portfolio |
| `classification` | `?classification=140,120` | Filter by BBR usage codes (comma-separated) |
| `year_from` | `?year_from=1900` | Minimum construction year |
| `year_to` | `?year_to=1950` | Maximum construction year |
| `bbox` | `?bbox=55.65,12.50,55.72,12.60` | Bounding box: `min_lat,min_lng,max_lat,max_lng` |

**Response:**
```json
{
  "data": [
    {
      "id": 1,
      "bbr_id": "d4f2e1a0-...",
      "name": "Nørrebrogade 42",
      "address": "Nørrebrogade 42, 2200 København N",
      "latitude": 55.6901,
      "longitude": 12.5528,
      "coordinate_x": 724501.41,
      "coordinate_y": 6178285.75,
      "classification": 140,
      "construction_year": 1923,
      "floors": 5,
      "apartments": 24,
      "sqm_built": 2850,
      "sqm_footprint": 570,
      "sqm_liveable": 2100,
      "sqm_business": 280,
      "sqm_basement": 400,
      "sqm_roof": 120,
      "energy_score": "C",
      "property_id": 1,
      "property_name": "Nørrebro Portfolio"
    }
  ],
  "meta": {
    "total": 87,
    "bbox": {
      "min_lat": 55.65,
      "max_lat": 55.72,
      "min_lng": 12.50,
      "max_lng": 12.60
    }
  }
}
```

**Key fields for 3D rendering:**

| Field | Use it for |
|---|---|
| `latitude` / `longitude` | Position on the map (WGS84) |
| `coordinate_x` / `coordinate_y` | Position in EPSG:25832 (if using OpenLayers) |
| `classification` | Pick the building sprite/model (see classification table below) |
| `floors` | Building height |
| `sqm_footprint` | Footprint size |
| `construction_year` | Age coloring / time slider |
| `energy_score` | Energy overlay (A-G scale) |

---

### 3. Building Detail

```
GET /api/v1/hackathon/buildings/{id}
```

Returns a single building with richer data: building elements (components), latest inspection date, and a risk summary.

**Response:**
```json
{
  "data": {
    "id": 1,
    "bbr_id": "d4f2e1a0-...",
    "name": "Nørrebrogade 42",
    "address": "Nørrebrogade 42, 2200 København N",
    "latitude": 55.6901,
    "longitude": 12.5528,
    "coordinate_x": 724501.41,
    "coordinate_y": 6178285.75,
    "classification": 140,
    "construction_year": 1923,
    "floors": 5,
    "apartments": 24,
    "sqm_built": 2850,
    "sqm_footprint": 570,
    "sqm_liveable": 2100,
    "sqm_business": 280,
    "sqm_basement": 400,
    "sqm_roof": 120,
    "energy_score": "C",
    "property_id": 1,
    "property_name": "Nørrebro Portfolio",
    "elements": [
      { "name": "Roof", "name_da": "Tag", "type": "roof", "tasks_count": 3 },
      { "name": "Facade", "name_da": "Facade", "type": "facade", "tasks_count": 1 }
    ],
    "latest_inspection": {
      "date": "2025-11-15"
    },
    "risk_summary": {
      "critical": 1,
      "high": 2,
      "medium": 5,
      "low": 3,
      "not_significant": 0
    }
  }
}
```

**Use case:** Info panel when clicking a building. Show elements breakdown, condition status, inspection freshness.

---

### 4. Building Condition

```
GET /api/v1/hackathon/buildings/{id}/condition
```

Maintenance and condition data per building element. Useful for heatmaps, color-coding buildings by maintenance need, or data overlays.

**Response:**
```json
{
  "data": {
    "building_id": 1,
    "maintenance_cost_per_sqm_ten_years": 250,
    "elements": [
      {
        "name": "Roof",
        "name_da": "Tag",
        "type": "roof",
        "task_count": 3,
        "cost_ten_years": 450000,
        "highest_risk": "CRITICAL"
      },
      {
        "name": "Facade",
        "name_da": "Facade",
        "type": "facade",
        "task_count": 1,
        "cost_ten_years": 120000,
        "highest_risk": "MEDIUM"
      }
    ],
    "energy": {
      "score": "C",
      "consumption_kwh_per_sqm": 145
    }
  }
}
```

**Risk levels:** `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `NOT_SIGNIFICANT`

**Use case:** Color buildings by worst risk level, show maintenance cost overlays, compare energy performance across a portfolio.

---

## Classification Codes

The `classification` field uses Denmark's BBR usage codes. Use these to pick sprites, models, or colors:

| Code range | Category | Suggested color |
|---|---|---|
| 110-190 | Residential | `#4CAF50` green |
| 210-239 | Agriculture / Production | `#FF9800` orange |
| 220-229 | Industrial | `#795548` brown |
| 310-319 | Transport / Parking | `#607D8B` blue-gray |
| 320-329 | Office / Retail / Warehouse | `#2196F3` blue |
| 330-339 | Hotel / Restaurant / Service | `#E91E63` pink |
| 410-419 | Culture (cinema, museum, library, church) | `#9C27B0` purple |
| 420-429 | Education (school, university) | `#3F51B5` indigo |
| 430-449 | Health / Institutional | `#F44336` red |
| 510-540 | Leisure / Holiday | `#00BCD4` teal |
| 530-539 | Sports | `#CDDC39` lime |
| 910-960 | Garage / Shed / Outbuilding | `#9E9E9E` gray |

---

## Quick Start Examples

### JavaScript (fetch)

```js
const API_BASE = 'https://dev-admin.ejendom.com/api/v1/hackathon';
const API_KEY = 'aec-hackathon-2026-twin-city';

async function fetchBuildings(params = '') {
  const response = await fetch(`${API_BASE}/buildings${params}`, {
    headers: { 'X-Hackathon-Key': API_KEY },
  });
  const { data } = await response.json();
  return data;
}

// All buildings
const buildings = await fetchBuildings();

// Only residential buildings built before 1950
const oldHousing = await fetchBuildings('?classification=120,130,140&year_to=1950');

// Buildings within a bounding box (central Copenhagen)
const central = await fetchBuildings('?bbox=55.67,12.55,55.69,12.59');
```

### Python (requests)

```python
import requests

API_BASE = "https://dev-admin.ejendom.com/api/v1/hackathon"
HEADERS = {"X-Hackathon-Key": "aec-hackathon-2026-twin-city"}

buildings = requests.get(f"{API_BASE}/buildings", headers=HEADERS).json()["data"]

for b in buildings:
    print(f"{b['address']} — {b['floors']} floors, built {b['construction_year']}")
```

### Combining with Google Maps 3D

```js
// Fetch buildings from ejendom.com API, place on Google 3D map
const buildings = await fetchBuildings();

buildings.forEach((b) => {
  if (!b.latitude || !b.longitude) return;

  const marker = new Marker3DElement({
    position: { lat: b.latitude, lng: b.longitude, altitude: 30 },
    label: `${b.address} (${b.construction_year})`,
    altitudeMode: "RELATIVE_TO_GROUND",
  });
  map.append(marker);
});
```

---

## Notes

- This API is available on the **development environment only**
- All data is read-only — there are no write/mutation endpoints
- Building coordinates are provided in both WGS84 (`latitude`/`longitude`) and EPSG:25832 (`coordinate_x`/`coordinate_y`)
- Some buildings may have `null` coordinates — filter them out for map rendering
- The `meta.bbox` in the buildings response gives you the bounding box of the result set

---

*Provided by [ejendom.com](https://ejendom.com) for AEC Hackathon 2026.*
