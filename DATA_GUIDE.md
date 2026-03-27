# TwinCity Copenhagen - Data Cheat Sheet

> Quick-start guide for fetching Danish building and address data from public APIs.
> Provided by [ejendom.com](https://ejendom.com) for AEC Hackathon 2026.

---

## Authentication Overview

You'll be working with two platforms. They have **different auth mechanisms**:

| Platform | Base URL | Auth method | Credentials needed? |
|---|---|---|---|
| **Dataforsyningen** | `https://api.dataforsyningen.dk` | Token as query param (`?token=XXX`) | Yes - free account |
| **Datafordeler** | `https://services.datafordeler.dk` | Username + password as query params | Yes - free account |
| **DAWA** (Dataforsyningen) | `https://api.dataforsyningen.dk` | **None** | No - completely open |

**DAWA** (Danmarks Adressers Web API) is your fastest starting point - no signup, no tokens, no auth. You can start fetching address data immediately.

### Getting credentials (for Datafordeler & Dataforsyningen)

1. **Dataforsyningen token**: Register at [dataforsyningen.dk](https://dataforsyningen.dk) → create a user → create a token
2. **Datafordeler username/password**: Register at [datafordeler.dk](https://datafordeler.dk) → create a "tjeneste bruger" (service user)

> **Hackathon tip**: Only one team member needs to do this. It takes ~10 minutes. Share the credentials via env vars.

---

## The Endpoints You Need

### 1. DAWA - Address Search (No Auth)

The fastest way to get started. Search for any Danish address and get structured data back.

```
GET https://api.dataforsyningen.dk/adgangsadresser/autocomplete?q={search}
```

**Example** - search for "Nørrebrogade":
```
https://api.dataforsyningen.dk/adgangsadresser/autocomplete?q=nørrebrogade
```

Returns a list of address suggestions with IDs you can use for further lookups.

**Get full address details**:
```
GET https://api.dataforsyningen.dk/adgangsadresser/{adresseId}
```

This returns coordinates (WGS84), municipality, postal code, ejerlav (cadastral district), matrikelnr (plot number), and more.

---

### 2. Jordstykke (Land Plot) - GeoJSON Geometry (No Auth)

Once you have `ejerlav.kode` and `matrikelnr` from the address lookup, fetch the plot boundary:

```
GET https://api.dataforsyningen.dk/jordstykker/{ejerlavkode}/{matrikelnr}?format=geojson&srid=25832
```

Returns GeoJSON with the land plot polygon - perfect for rendering plot boundaries on a map.

> **Note**: The coordinates use **EPSG:25832** (UTM zone 32N), which is Denmark's standard projection. If you're working with Leaflet/Mapbox (which use EPSG:4326/WGS84), you'll need to reproject. See the [Map Setup](#map-setup-with-openlayers) section.

---

### 3. BBR Buildings - The Core Data (Requires Datafordeler Auth)

This is the main endpoint for the TwinCity challenge. It returns building data from BBR (Bygnings- og Boligregistret).

**Get buildings on a land plot** (most common approach):
```
GET https://services.datafordeler.dk/BBR/BBRPublic/1/rest/bygning?jordstykke={jordstykkeId}&username={USER}&password={PASS}
```

**Get buildings by house number ID**:
```
GET https://services.datafordeler.dk/BBR/BBRPublic/1/rest/bygning?husnummer={husnummerId}&username={USER}&password={PASS}
```

**Get a single building by BBR ID**:
```
GET https://services.datafordeler.dk/BBR/BBRPublic/1/rest/bygning?Id={bbrId}&username={USER}&password={PASS}
```

---

### 4. DAR - Address Resolution (Requires Datafordeler Auth)

Resolve house numbers to addresses and vice versa:

**Addresses from house number**:
```
GET https://services.datafordeler.dk/DAR/DAR/1/REST/Adresse?husnummer={husNrId}&format=json&username={USER}&password={PASS}
```

**House number details**:
```
GET https://services.datafordeler.dk/DAR/DAR/2.0.0/rest/Husnummer?id={husNrId}&format=json&username={USER}&password={PASS}
```

**BFE (property ID) from house number**:
```
GET https://services.datafordeler.dk/DAR/DAR_BFE_Public/1/rest/husnummerTilBygningBfe?husnummerid={husNrId}&username={USER}&password={PASS}
```

---

### 5. WMS Tile Layers (For Map Backgrounds)

These are image tile services for rendering map layers - not data APIs. Useful for base maps.

| Layer | URL | Auth |
|---|---|---|
| Aerial photo (ortofoto) | `https://services.datafordeler.dk/GeoDanmarkOrto/orto_foraar/1.0.0/WMS` | Datafordeler user/pass |
| Cadastral boundaries | `https://api.dataforsyningen.dk/wms/MatGaeldendeOgForeloebigWMS_DAF` | Dataforsyningen token |
| House numbers / road names | `https://api.dataforsyningen.dk/wms/forvaltning2` | Dataforsyningen token |

---

## BBR Building Fields Reference

BBR field names are cryptic. Here's what the important ones mean:

### Building identification
| BBR Field | Type | What it is |
|---|---|---|
| `id_lokalId` | string (UUID) | Unique building ID |
| `byg007Bygningsnummer` | number | Building number on the plot |
| `status` | string | Lifecycle status (see below) |
| `kommunekode` | string | Municipality code |
| `husnummer` | string (UUID) | Reference to house number in DAR |
| `jordstykke` | string (UUID) | Reference to the land plot |

### Building characteristics
| BBR Field | Type | What it is |
|---|---|---|
| `byg021BygningensAnvendelse` | number | **Usage classification code** (see full table below) |
| `byg026Opførelsesår` | number | **Year of construction** |
| `byg038SamletBygningsareal` | number | Total built area (sqm) |
| `byg039BygningensSamledeBoligAreal` | number | Residential area (sqm) |
| `byg040BygningensSamledeErhvervsAreal` | number | Commercial area (sqm) |
| `byg041BebyggetAreal` | number | Footprint area (sqm) |
| `byg054AntalEtager` | number | **Number of floors** |
| `byg024AntalLejlighederMedKøkken` | number | Number of apartments |
| `byg404Koordinat` | string | **Coordinates** (WKT format, see gotcha below) |

### Nested data
| BBR Field | Type | What it is |
|---|---|---|
| `etageList` | array | Floor details (area per floor, roof, basement) |
| `opgangList` | array | Stairwells / entrances |
| `bygningPåFremmedGrundList` | array | If building is on foreign land (BPFG) |

### Building Status Codes
| Status | Meaning | Include? |
|---|---|---|
| `6` | Under construction | Maybe |
| `7` | Active/existing | Yes |
| `9` | Demolished / concluded | **No** |
| `11` | Incorrectly registered | **No** |

> **Important**: Always filter out status `9` and `11` - otherwise you'll render demolished or ghost buildings.

---

## BBR Usage Classification Codes (byg021)

This is how you visually differentiate buildings in TwinCity. Group by category for your rendering:

### Residential (100-series)
| Code | Description |
|---|---|
| 110 | Farmhouse (stuehus) |
| 120 | Detached single-family house (parcelhus) |
| 130, 131, 132 | Terraced / semi-detached house |
| 140 | Multi-storey apartment building |
| 150 | Student housing (kollegium) |
| 160 | Institutional housing |
| 185 | Annex to year-round residence |
| 190 | Other year-round residential |

### Agriculture & Production (200-series)
| Code | Description |
|---|---|
| 210, 219 | Agricultural production |
| 211-214 | Livestock (pigs, cattle, poultry, mink) |
| 215-218 | Greenhouse, barn, machinery |
| 220-229 | Industrial production, workshops |
| 230-239 | Energy & utility plants |

### Commercial & Transport (300-series)
| Code | Description |
|---|---|
| 310-319 | Transport (railway, airport, parking, harbour) |
| 320-329 | Office, retail, warehouse |
| 321 | Office |
| 322 | Retail |
| 323 | Warehouse |
| 324 | Shopping centre |
| 325 | Gas station |
| 330-339 | Hotel, restaurant, services |

### Public & Institutional (400-series)
| Code | Description |
|---|---|
| 411 | Cinema, theatre, concert |
| 412 | Museum |
| 413 | Library |
| 414 | Church |
| 421 | Primary school |
| 422 | University |
| 431 | Hospital |
| 443 | Military barracks |
| 444 | Prison |

### Leisure & Sports (500-series)
| Code | Description |
|---|---|
| 510 | Summer house |
| 521 | Holiday centre |
| 532 | Swimming hall |
| 533 | Sports hall |
| 540 | Allotment house (kolonihave) |

### Other (900-series)
| Code | Description |
|---|---|
| 910 | Garage |
| 920 | Carport |
| 930 | Shed / outhouse |
| 950 | Freestanding canopy |
| 990 | Dilapidated building |
| 999 | Unknown |

---

## Gotchas & Tips

### 1. CORS Will Block You

When calling Datafordeler/Dataforsyningen directly from a browser, the default `Authorization` header and `Accept` headers will trigger a CORS preflight that gets rejected.

**Fix** - strip the headers before the request:

```js
const fetchBuilding = async (url) => {
  const response = await fetch(url, {
    headers: {
      'Accept': '*/*',
      'Content-Type': 'text/plain',
    },
  });
  return response.json();
};
```

If you're using axios:

```js
const axiosConfig = {
  transformRequest: (data, headers) => {
    headers.Accept = '*/*';
    headers['Content-Type'] = 'text/plain';
    delete headers.Authorization;
    return data;
  },
};

const response = await axios.get(url, axiosConfig);
```

### 2. Coordinate Parsing

BBR coordinates come as a WKT string, not as `[x, y]`:

```
"POINT(723501.41 6176285.75)"
```

Parse it like this:

```js
function parseBBRCoordinate(coordinateString) {
  if (!coordinateString) return null;
  const match = coordinateString.match(/POINT\((-?\d+\.?\d*)\s+(-?\d+\.?\d*)\)/);
  if (!match) return null;
  return {
    x: parseFloat(match[1]),
    y: parseFloat(match[2]),
  };
}
```

These coordinates are in **EPSG:25832** (UTM zone 32N), not lat/lng. If you need WGS84 (lat/lng) for Leaflet or Mapbox, use [proj4js](https://www.npmjs.com/package/proj4):

```js
import proj4 from 'proj4';

proj4.defs('EPSG:25832', '+proj=utm +zone=32 +ellps=GRS80 +units=m +no_defs');

function toLatLng(x, y) {
  const [lng, lat] = proj4('EPSG:25832', 'EPSG:4326', [x, y]);
  return { lat, lng };
}
```

### 3. Building Lifecycle Filtering

Always filter your building results. BBR includes demolished buildings and registration errors:

```js
const INACTIVE_STATUSES = ['9', '11'];

const activeBuildings = buildings.filter(
  (b) => !INACTIVE_STATUSES.includes(b.status)
);
```

### 4. Some Buildings Have No Coordinates

Not every building in BBR has a `byg404Koordinat`. Skip them gracefully:

```js
const renderableBuildings = buildings.filter((b) => b.byg404Koordinat);
```

### 5. The Full Address → Buildings Pipeline

Getting from an address search to renderable buildings requires chaining multiple APIs. Here's the complete flow:

```js
async function getBuildingsFromAddress(addressId) {
  // 1. Get address details (DAWA - no auth)
  const address = await fetch(
    `https://api.dataforsyningen.dk/adgangsadresser/${addressId}`
  ).then((r) => r.json());

  // 2. Get land plot geometry (DAWA - no auth)
  const jordstykke = await fetch(
    `https://api.dataforsyningen.dk/jordstykker/${address.ejerlav.kode}/${address.matrikelnr}?format=geojson&srid=25832`
  ).then((r) => r.json());

  // 3. Get buildings on the plot (Datafordeler - requires auth)
  const featureId = jordstykke.properties.featureid;
  const buildings = await fetchBuilding(
    `https://services.datafordeler.dk/BBR/BBRPublic/1/rest/bygning?jordstykke=${featureId}&username=${USER}&password=${PASS}`
  );

  // 4. Filter and return active buildings with coordinates
  return buildings
    .filter((b) => !['9', '11'].includes(b.status))
    .filter((b) => b.byg404Koordinat)
    .map((b) => ({
      id: b.id_lokalId,
      coordinates: parseBBRCoordinate(b.byg404Koordinat),
      year: b.byg026Opførelsesår,
      usage: b.byg021BygningensAnvendelse,
      floors: b.byg054AntalEtager,
      areaSqm: b.byg038SamletBygningsareal,
      footprintSqm: b.byg041BebyggetAreal,
      residentialSqm: b.byg039BygningensSamledeBoligAreal,
      commercialSqm: b.byg040BygningensSamledeErhvervsAreal,
      apartments: b.byg024AntalLejlighederMedKøkken,
      buildingNumber: b.byg007Bygningsnummer,
    }));
}
```

---

## Map Setup with OpenLayers

If you use [OpenLayers](https://openlayers.org/) (recommended for Danish map services), here's the projection setup for Denmark:

```js
import proj4 from 'proj4';
import { register } from 'ol/proj/proj4';
import { get as getProjection } from 'ol/proj';
import View from 'ol/View';

// Register Denmark's projection
proj4.defs('EPSG:25832', '+proj=utm +zone=32 +ellps=GRS80 +units=m +no_defs');
register(proj4);

const projectionDenmark = getProjection('EPSG:25832');
projectionDenmark.setExtent([120000, 5661139.2, 1378291.2, 6500000]);

// Copenhagen center
const centerCopenhagen = [724000, 6176000];

const view = new View({
  center: centerCopenhagen,
  zoom: 13,
  projection: projectionDenmark,
});
```

### Adding WMS base layers (aerial photo, cadastral boundaries)

```js
import TileLayer from 'ol/layer/Tile';
import TileWMS from 'ol/source/TileWMS';

// Aerial photo (requires Datafordeler credentials)
const ortofotoLayer = new TileLayer({
  source: new TileWMS({
    url: 'https://services.datafordeler.dk/GeoDanmarkOrto/orto_foraar/1.0.0/WMS',
    params: {
      LAYERS: 'orto_foraar',
      TRANSPARENT: 'FALSE',
      username: DATAFORDELER_USERNAME,
      password: DATAFORDELER_PASSWORD,
      SERVICE: 'WMS',
      VERSION: '1.3.0',
      REQUEST: 'GetMap',
      FORMAT: 'image/jpeg',
    },
  }),
});

// Cadastral boundaries (requires Dataforsyningen token)
const matrikelLayer = new TileLayer({
  source: new TileWMS({
    url: 'https://api.dataforsyningen.dk/wms/MatGaeldendeOgForeloebigWMS_DAF',
    params: {
      LAYERS: 'MatrikelSkel_Gaeldende, Centroide_Gaeldende',
      TRANSPARENT: 'TRUE',
      token: DATAFORSYNINGEN_TOKEN,
    },
  }),
});

// House numbers and road names (requires Dataforsyningen token)
const labelLayer = new TileLayer({
  source: new TileWMS({
    url: 'https://api.dataforsyningen.dk/wms/forvaltning2',
    params: {
      TOKEN: DATAFORSYNINGEN_TOKEN,
      SERVICE: 'WMS',
      REQUEST: 'GetMap',
      FORMAT: 'image/png',
      TRANSPARENT: 'TRUE',
      LAYERS: 'Husnummer_ortofoto,Vejnavne_ortofoto',
      SRS: 'EPSG:25832',
    },
  }),
});
```

---

## Quick Start Suggestions

### Fastest path (no credentials needed)

1. Use DAWA address autocomplete to search for an area
2. Use DAWA jordstykke endpoint to get plot geometries as GeoJSON
3. Render plots on a map - you already have a working visual

### Full experience (with free credentials)

1. Register at datafordeler.dk for BBR access (~10 min)
2. Chain: address search → jordstykke → BBR buildings
3. Parse coordinates, classify by usage code, render in your game view
4. Use WMS layers for aerial photo backgrounds

### What data is best for the game-style rendering?

For each building you get from BBR, use these fields to drive your visuals:

- **`byg021BygningensAnvendelse`** → pick the sprite/model (house, office, factory, church...)
- **`byg054AntalEtager`** → building height
- **`byg041BebyggetAreal`** → footprint size
- **`byg026Opførelsesår`** → age coloring or time slider
- **`byg404Koordinat`** → placement on the map

---

## Useful Links

- [DAWA documentation](https://dawadocs.dataforsyningen.dk/) - Address API docs (no auth needed)
- [Datafordeler](https://datafordeler.dk/) - Register for BBR access
- [Dataforsyningen](https://dataforsyningen.dk/) - Register for map tokens
- [BBR data model documentation](https://teknik.bbr.dk/forretning/bygning) - Official BBR field reference
- [OpenLayers](https://openlayers.org/) - Recommended map library for Danish projections
- [proj4js](https://www.npmjs.com/package/proj4) - Coordinate reprojection

---

## Bonus: Photorealistic 3D Copenhagen with Google Maps

Want to skip the game-engine rendering and drop BBR data onto a photorealistic 3D city? Google's **Maps JavaScript API** has a 3D Maps feature that renders real-world buildings, streets, and terrain out of the box - and Copenhagen has full coverage.

### EEA / Denmark Availability - Read This First

Google has **two different 3D products** and the distinction matters:

| Product | How it works | EEA status |
|---|---|---|
| **Map Tiles API** (`tile.googleapis.com/v1/3dtiles/root.json`) | You fetch OGC 3D Tiles and render with CesiumJS/deck.gl | **Blocked** - returns `403` for EEA billing addresses since July 2025 |
| **Maps JavaScript API - 3D Maps** (`Map3DElement` / `<gmp-map-3d>`) | Google renders internally, you get an HTML element | **Works** - currently in free Preview, no billing required |

Use `Map3DElement`. It's the path that works for Danish developers, requires no 3D rendering library, and is free during Preview.

### Minimal Working Example - 3D Copenhagen

This is a single HTML file. No build tools, no dependencies, no billing. Just replace `YOUR_API_KEY` with a [Google Maps API key](https://console.cloud.google.com/apis/credentials) that has the Maps JavaScript API enabled.

```html
<!DOCTYPE html>
<html>
<head>
  <title>TwinCity Copenhagen - 3D</title>
  <style>
    html, body { height: 100%; margin: 0; }
    gmp-map-3d { width: 100%; height: 100%; }
  </style>
</head>
<body>
  <!-- Load Maps JS API -->
  <script async defer>
    (g=>{var h,a,k,p="The Google Maps JavaScript API",c="google",l="importLibrary",
    q="__ib__",m=document,b=window;b=b[c]||(b[c]={});var d=b.maps||(b.maps={}),
    r=new Set,e=new URLSearchParams,u=()=>h||(h=new Promise(async(f,n)=>{await(a=
    m.createElement("script"));e.set("libraries",[...r]+"");for(k in g)e.set(k.replace(
    /[A-Z]/g,t=>"_"+t[0].toLowerCase()),g[k]);e.set("callback",c+".maps."+q);a.src=
    `https://maps.${c}apis.com/maps/api/js?`+e;d[q]=f;a.onerror=()=>h=n(Error(p+
    " could not load."));a.nonce=m.querySelector("script[nonce]")?.nonce||"";
    m.head.append(a)}));d[l]?console.warn(p+" only loads once. Ignoring:",g):
    d[l]=(f,...n)=>r.add(f)&&u().then(()=>d[l](f,...n))})({
      key: "YOUR_API_KEY",
      v: "alpha",
    });
  </script>

  <script>
    async function init() {
      const { Map3DElement } = await google.maps.importLibrary("maps3d");

      const map = new Map3DElement({
        center: { lat: 55.6761, lng: 12.5683, altitude: 200 },
        range: 1500,
        tilt: 60,
        heading: 30,
        mode: "SATELLITE",
      });

      document.body.append(map);
    }
    init();
  </script>
</body>
</html>
```

Open it in a browser and you'll see photorealistic 3D Copenhagen with full pan/zoom/tilt controls.

### Overlaying BBR Data on the 3D Map

This is where it gets interesting for TwinCity. Combine the BBR building data from earlier in this guide with 3D map overlays:

#### Add markers for each BBR building

```js
async function addBuildingMarkers(map, buildings) {
  const { Marker3DElement } = await google.maps.importLibrary("maps3d");

  buildings.forEach((building) => {
    // BBR coordinates are EPSG:25832 — convert to lat/lng first
    const { lat, lng } = toLatLng(building.coordinates.x, building.coordinates.y);

    const marker = new Marker3DElement({
      position: { lat, lng, altitude: 50 },
      label: `${building.usage} — ${building.year}`,
      altitudeMode: "RELATIVE_TO_GROUND",
    });

    map.append(marker);
  });
}
```

#### Color-coded extruded polygons by building type

Use `Polygon3DElement` to draw extruded shapes that represent buildings, colored by their BBR usage category:

```js
async function addBuildingPolygon(map, building) {
  const { Polygon3DElement } = await google.maps.importLibrary("maps3d");

  const { lat, lng } = toLatLng(building.coordinates.x, building.coordinates.y);

  // Approximate a building footprint from BBR area data
  const halfSide = Math.sqrt(building.footprintSqm || 100) / 2;
  const offset = halfSide * 0.00001; // rough meters-to-degrees

  const color = getColorByUsage(building.usage);
  // Estimate height: ~3m per floor
  const height = (building.floors || 1) * 3;

  const polygon = new Polygon3DElement({
    strokeColor: color,
    strokeWidth: 2,
    fillColor: color,
    fillOpacity: 0.6,
    altitudeMode: "RELATIVE_TO_GROUND",
    extruded: true,  // This makes it 3D!
    drawsOccludedSegments: true,
  });

  polygon.outerCoordinates = [
    { lat: lat - offset, lng: lng - offset, altitude: height },
    { lat: lat + offset, lng: lng - offset, altitude: height },
    { lat: lat + offset, lng: lng + offset, altitude: height },
    { lat: lat - offset, lng: lng + offset, altitude: height },
    { lat: lat - offset, lng: lng - offset, altitude: height },
  ];

  map.append(polygon);
}

function getColorByUsage(code) {
  if (code >= 100 && code < 200) return "#4CAF50"; // Residential — green
  if (code >= 200 && code < 300) return "#FF9800"; // Production — orange
  if (code >= 300 && code < 400) return "#2196F3"; // Commercial — blue
  if (code >= 400 && code < 500) return "#9C27B0"; // Institutional — purple
  if (code >= 500 && code < 600) return "#00BCD4"; // Leisure — teal
  if (code >= 900)              return "#9E9E9E"; // Other — gray
  return "#FFFFFF";
}
```

#### Place custom 3D models (glTF)

The Maps JS API supports placing glTF 3D models directly on the map. You could create simplified game-style building models and place them at BBR coordinates:

```js
async function addBuildingModel(map, building) {
  const { Model3DElement } = await google.maps.importLibrary("maps3d");

  const { lat, lng } = toLatLng(building.coordinates.x, building.coordinates.y);

  const model = new Model3DElement({
    src: "/models/building-residential.glb",  // your custom glTF model
    position: { lat, lng, altitude: 0 },
    altitudeMode: "CLAMP_TO_GROUND",
    scale: building.floors || 1,
  });

  map.append(model);
}
```

This is the closest to the SimCity aesthetic from the challenge deck — real Google 3D city as backdrop, with custom game-style models or color-coded extrusions placed from BBR data.

### Camera Animations - "Watch the City Grow"

The time slider stretch goal from the challenge becomes straightforward with camera animation:

```js
async function flyToBuilding(map, lat, lng) {
  // Smooth camera animation
  await map.flyCameraTo({
    endCamera: {
      center: { lat, lng, altitude: 100 },
      range: 500,
      tilt: 65,
      heading: 0,
    },
    durationMillis: 2000,
  });
}

async function animateCityGrowth(map, buildings) {
  // Sort buildings by construction year
  const sorted = [...buildings].sort((a, b) => a.year - b.year);

  for (const building of sorted) {
    const { lat, lng } = toLatLng(building.coordinates.x, building.coordinates.y);
    await addBuildingPolygon(map, building);
    await flyToBuilding(map, lat, lng);
    await new Promise((r) => setTimeout(r, 500)); // pause between buildings
  }
}
```

### Coordinate Conversion Reminder

The BBR pipeline from earlier in this guide gives you EPSG:25832 coordinates. The Google Maps 3D API needs lat/lng (WGS84). Use the `toLatLng()` function from the [Coordinate Parsing](#2-coordinate-parsing) section:

```js
import proj4 from 'proj4';
proj4.defs('EPSG:25832', '+proj=utm +zone=32 +ellps=GRS80 +units=m +no_defs');

function toLatLng(x, y) {
  const [lng, lat] = proj4('EPSG:25832', 'EPSG:4326', [x, y]);
  return { lat, lng };
}
```

### Google Maps 3D - Useful Links

- [3D Maps Overview](https://developers.google.com/maps/documentation/javascript/3d/overview) - Getting started guide
- [Map3DElement Reference](https://developers.google.com/maps/documentation/javascript/reference/3d-map) - Full API reference
- [3D Maps Codelab](https://developers.google.com/codelabs/maps-platform/maps-platform-101-3d-maps-js) - Step-by-step tutorial
- [3D Models Example](https://developers.google.com/maps/documentation/javascript/examples/3d/models) - Placing glTF models
- [3D Markers & Animation Codelab](https://developers.google.com/codelabs/maps-platform/maps-platform-3d-maps-js-markers) - Markers and camera animation
- [EEA Restrictions Explained](https://developers.google.com/maps/comms/eea/map-tiles) - What's blocked vs. available

---

*Built with lessons learned from [ejendom.com](https://ejendom.com) - we've already navigated these APIs so you don't have to read 200 pages of docs. Good luck at the hackathon!*
