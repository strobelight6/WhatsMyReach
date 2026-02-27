# WhatsMyReach — Map-Based Home Buying Tool
## Product & Technical Planning Document

**Version:** 0.2 Draft  
**Date:** February 2026  
**Status:** In Progress

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [Key User Stories](#3-key-user-stories)
4. [Recommended Technology Stack](#4-recommended-technology-stack)
5. [Architecture](#5-architecture)
6. [Backend API Design](#6-backend-api-design)
7. [Database Design](#7-database-design)
8. [Frontend Design](#8-frontend-design)
9. [POI Categories & Overpass Queries](#9-poi-categories--overpass-queries)
10. [Walking vs. Driving Logic](#10-walking-vs-driving-logic)
11. [Caching Strategy](#11-caching-strategy)
12. [Saved Views & Comparison](#12-saved-views--comparison)
13. [API Considerations & Rate Limits](#13-api-considerations--rate-limits)
14. [Phased Development Roadmap](#14-phased-development-roadmap)
15. [Open Questions](#15-open-questions)
16. [Success Metrics](#16-success-metrics)

---

## 1. Product Overview

WhatsMyReach is a web application that helps prospective home buyers evaluate neighborhoods by visualizing proximity and travel times to key destinations. A buyer enters a property address and immediately sees directions — walking or driving based on a configurable distance threshold — to nearby grocery stores, bars, restaurants, parks, and airports, complete with travel times and distances in miles.

Users can customize which destination categories they care about and save multiple address views for side-by-side comparison across sessions and devices.

The core value proposition is speed and clarity: rather than manually searching each amenity on a general-purpose map, a buyer gets a single, tailored view of what matters to them within seconds.

---

## 2. Goals & Non-Goals

### 2.1 Goals

- Allow users to input any US address and see nearby points of interest on an interactive map.
- Display walking directions for destinations within a user-defined distance threshold; driving directions beyond it.
- Show travel time and distance in miles for each route.
- Let users select and reorder which POI categories are displayed (e.g. hide bars, add hospitals).
- Suppress POI categories that return no results rather than showing empty states.
- Enable saving of address views (backed by a database) so users can return and compare across devices.
- Support side-by-side comparison of exactly 2 saved address views, with the data model designed to extend to N addresses in the future.
- Deliver a fast, mobile-friendly experience with aggressive frontend caching to minimize redundant API and routing requests.

### 2.2 Non-Goals (v1)

- Native mobile apps (iOS / Android) — the responsive web app suffices for v1.
- Real-time data (live traffic, opening hours) — static routing estimates are sufficient.
- International addresses — US-only for launch.
- Property listing data or MLS integration.
- Application analytics — deferred to a future phase.
- User authentication / accounts — views are saved anonymously by session or via a simple identifier in v1. Full auth is a Phase 2 item.

---

## 3. Key User Stories

| Actor | Want | Notes |
|---|---|---|
| Home buyer | Enter an address and see nearby grocery stores on a map | Core navigation feature |
| Home buyer | See walking time for close destinations and driving time for distant ones | Auto mode switching |
| Home buyer | Set my own walking/driving threshold distance | User preference |
| Home buyer | Choose which categories of places to display | Customizable POI filter |
| Home buyer | Only see categories that have actual results nearby | Hide empty categories |
| Home buyer | Save a view for an address so I can revisit it on any device | DB-backed saved views |
| Home buyer | Compare two saved addresses side-by-side | 2-view comparison |
| Home buyer | See distance in miles and travel time for every route | Route metadata |
| Home buyer | Not wait for data I've already loaded when revisiting an address | Frontend caching |

---

## 4. Recommended Technology Stack

The stack prioritizes open-source tools, free tiers for third-party services, and a clear separation between a stateless frontend and a Python or Go backend. The goal is to validate the application with minimal cost before any paid services are introduced.

### 4.1 Stack Summary

| Layer | Tool | Rationale |
|---|---|---|
| **Frontend Framework** | React + Vite | Fast dev builds, ecosystem maturity |
| **Styling** | Tailwind CSS | Utility-first, minimal CSS overhead |
| **Map Rendering** | Leaflet.js + React-Leaflet | Open-source, MIT license, free OSM tile support |
| **Map Tiles** | OpenStreetMap (via Leaflet default) | Free, open data, no API key required |
| **State Management** | Zustand | Lightweight, no boilerplate, hooks-first |
| **Frontend Cache** | Zustand + in-memory / IndexedDB (via idb-keyval) | Avoids redundant geocode/POI/route calls within and across sessions |
| **Backend Language** | **Python (FastAPI)** or **Go (chi / Fiber)** — see 4.3 | Lightweight, performant, non-JS backend |
| **Backend Runtime** | Single container (Docker) | Simple deployment; scales horizontally if needed |
| **Geocoding** | Nominatim (OSM) — proxied via backend | Free; backend proxy adds caching and shields the rate limit |
| **POI Search** | Overpass API (OSM) — proxied via backend | Free, open; backend caches results by (coords, radius, category) |
| **Routing** | OSRM (Open Source Routing Machine) — proxied via backend | Open-source; backend caches routes by (origin, destination, mode) |
| **Database** | PostgreSQL | Reliable, open-source, excellent JSON support for flexible POI storage |
| **ORM / Query Layer** | SQLAlchemy 2.x + Alembic (Python) *or* pgx/sqlc (Go) | Type-safe, migration-friendly |
| **Background Tasks** | FastAPI BackgroundTasks *or* simple goroutines | Async POI pre-fetch, cache warm-up |
| **Deployment — Frontend** | Vercel | Zero-config, free tier, CDN-served |
| **Deployment — Backend** | Render or Railway (free tier to paid) | Docker deploys, managed PostgreSQL available |
| **Package Manager** | pnpm (frontend) / pip + uv (Python) or Go modules | Fast, reproducible |

### 4.2 Backend Language Recommendation: Python (FastAPI)

For this project, **FastAPI** is the recommended backend choice unless the team has strong Go preference. Reasons:

- FastAPI's async support (via `asyncio` + `httpx`) is well-suited to the backend's primary job: proxying and caching HTTP calls to Nominatim, Overpass, and OSRM.
- SQLAlchemy 2.x with async support pairs naturally with FastAPI and PostgreSQL via `asyncpg`.
- Pydantic models (built into FastAPI) provide clean request/response validation and serve as living API documentation.
- Auto-generated OpenAPI docs (`/docs`) are useful for frontend developers consuming the API.
- The team can move to Go if throughput becomes a concern at scale — the API surface is small and the migration would be straightforward.

**If Go is preferred:** use the `chi` router (lightweight, stdlib-compatible) with `pgx` for PostgreSQL and `golang-migrate` for migrations. The API design in Section 6 applies to both languages.

### 4.3 Why These Choices

**Leaflet vs. Google Maps / Mapbox**

Leaflet is MIT-licensed and pairs with OpenStreetMap tiles at zero cost with no API key. Google Maps and Mapbox both require keys and have usage-based billing that can surprise early-stage projects. The trade-off is that OSM tile quality and POI data richness can be slightly lower in some regions, but it is entirely sufficient for a US home-buying use case.

**Backend as API Proxy**

Rather than calling Nominatim, Overpass, and OSRM directly from the browser, all third-party requests are routed through the backend. This provides three benefits: (1) API keys and credentials never reach the client; (2) the backend can cache responses in PostgreSQL, dramatically reducing upstream API calls; (3) rate limiting and retry logic lives in one place.

**PostgreSQL over SQLite or NoSQL**

PostgreSQL's JSONB column type handles the variable-shape POI and route payloads well, while relational tables handle structured data (saved views, user sessions) cleanly. It is also the natural fit for the managed database tiers on Render and Railway.

**Nominatim + Overpass vs. a Single Commercial Geocoding API**

Together they cover what Google Places would provide commercially, at no cost. The main constraint is rate limits on the public servers — the backend proxy with a PostgreSQL cache mitigates this significantly. Scaling to production would warrant either self-hosting these services or layering in a commercial API (Geoapify or LocationIQ both have generous free tiers).

**OSRM for Routing**

OSRM is an open-source routing engine built on OSM road data. The project-OSRM public demo server accepts HTTP routing requests for free and supports walking and driving profiles, returning distance, duration, and full GeoJSON route geometry. For production scale, OSRM can be self-hosted on a small VPS. The public demo server is not suitable for production traffic but is ideal for the proof-of-concept phase.

---

## 5. Architecture

### 5.1 High-Level Architecture

```
+------------------------------------------------------------------+
|                        Browser (React SPA)                        |
|                                                                    |
|  +--------------+  +--------------+  +------------------------+  |
|  | Zustand Store|  |  Leaflet Map |  |  IndexedDB Cache        |  |
|  |  (app state) |  |  (rendering) |  |  (cross-session cache)  |  |
|  +------+-------+  +--------------+  +------------------------+  |
|         |  REST API calls                                          |
+---------|----------------------------------------------------------+
          | HTTPS
          v
+------------------------------------------------------------------+
|                    Backend API (FastAPI / Go)                      |
|                                                                    |
|  +--------------+  +--------------+  +------------------------+  |
|  |  /geocode    |  |  /pois       |  |  /routes               |  |
|  |  /views      |  |  /compare    |  |  /health               |  |
|  +------+-------+  +------+-------+  +-----------+------------+  |
|         |                 |                       |               |
|  +------v-----------------v-----------------------v-----------+  |
|  |                   PostgreSQL Database                        |  |
|  |  geocode_cache | poi_cache | route_cache | saved_views       |  |
|  +-------------------------------------------------------------+  |
+-----------------------------+------------------------------------+
                              |  Outbound HTTP (proxied, cached)
                   +----------+-----------------+
                   v          v                 v
              Nominatim   Overpass API     OSRM Demo Server
              (geocode)   (POI search)     (routing)
```

### 5.2 Request Data Flow

1. User enters an address in the search bar (debounced, 400ms).
2. Frontend checks its **in-memory cache** (Zustand) then **IndexedDB cache** for a prior geocode result for this address. On miss, calls `GET /api/geocode?address=...`.
3. Backend checks **PostgreSQL geocode_cache**. On miss, calls Nominatim, stores result, returns `{ lat, lng }`.
4. Frontend checks cache for POI results for this `(lat, lng, radius, category[])`. On miss, calls `GET /api/pois?lat=&lng=&categories=&radius=`.
5. Backend fans out Overpass queries for each requested category concurrently. Results are cached in **poi_cache** keyed by `(lat_rounded, lng_rounded, radius_m, category)`. Returns a map of `category → POI[]`.
6. Frontend filters out any categories that returned zero results — these are not displayed or shown in the filter panel for this address.
7. For each POI, the frontend computes straight-line distance. If under the walking threshold, it requests a walking route; otherwise a driving route. Routes are fetched via `GET /api/route?orig_lat=&orig_lng=&dest_lat=&dest_lng=&mode=walk|drive`.
8. Backend checks **route_cache** by `(orig, dest, mode)`. On miss, calls OSRM, stores result, returns `{ distance_miles, duration_seconds, geojson }`.
9. Frontend renders the map: origin marker, POI markers (color-coded by category), and route polylines with popup showing name, travel mode, distance, and time.
10. User can save the current view. Frontend calls `POST /api/views` which persists the view to the **saved_views** table.

### 5.3 Deployment Topology

| Component | Platform | Notes |
|---|---|---|
| React SPA | Vercel | CDN-served, zero-config from Git |
| FastAPI backend | Render (Web Service) | Docker container, free tier for POC |
| PostgreSQL | Render (Managed PostgreSQL) | Free tier: 1GB, sufficient for POC |
| OSRM | project-osrm.org demo (POC) → self-hosted (production) | Self-host on Hetzner CX21 (~€5/mo) for production |

---

## 6. Backend API Design

All endpoints are prefixed `/api/v1`. The backend is stateless — all persistence is in PostgreSQL.

### 6.1 Endpoints

#### `GET /api/v1/geocode`

Converts a free-text address to coordinates.

**Query params:**

| Param | Type | Required | Description |
|---|---|---|---|
| `address` | string | yes | Free-text address string |

**Response `200`:**
```json
{
  "address": "123 Main St, Springfield, IL 62701",
  "lat": 39.7981,
  "lng": -89.6542,
  "display_name": "123 Main Street, Springfield, Sangamon County, Illinois, 62701, United States",
  "cached": true
}
```

---

#### `GET /api/v1/pois`

Returns points of interest near a coordinate for one or more categories.

**Query params:**

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `lat` | float | yes | — | Origin latitude |
| `lng` | float | yes | — | Origin longitude |
| `categories` | string (CSV) | yes | — | Comma-separated category keys, e.g. `grocery,park,bar` |
| `radius_km` | float | no | `5.0` | Search radius in km (overridden to 48.3km for `airport` category) |
| `limit` | int | no | `5` | Max POIs per category |

**Response `200`:**
```json
{
  "results": {
    "grocery": [
      {
        "id": "osm:node:123456",
        "name": "Whole Foods Market",
        "lat": 39.8012,
        "lng": -89.6489,
        "distance_miles": 0.31,
        "tags": { "opening_hours": "Mo-Su 07:00-22:00" }
      }
    ],
    "park": [ "..." ],
    "bar": []
  },
  "empty_categories": ["bar"]
}
```

> Categories with zero results are returned in `empty_categories` and should be hidden by the frontend.

---

#### `GET /api/v1/route`

Returns routing data between two coordinates.

**Query params:**

| Param | Type | Required | Description |
|---|---|---|---|
| `orig_lat` | float | yes | Origin latitude |
| `orig_lng` | float | yes | Origin longitude |
| `dest_lat` | float | yes | Destination latitude |
| `dest_lng` | float | yes | Destination longitude |
| `mode` | string | yes | `walk` or `drive` |

**Response `200`:**
```json
{
  "mode": "walk",
  "distance_miles": 0.31,
  "duration_seconds": 375,
  "duration_display": "6 min",
  "geojson": {
    "type": "LineString",
    "coordinates": [ [-89.6542, 39.7981], "..." ]
  },
  "cached": true
}
```

---

#### `POST /api/v1/views`

Saves an address view.

**Request body:**
```json
{
  "session_id": "uuid-string",
  "label": "123 Main St",
  "address": "123 Main St, Springfield, IL 62701",
  "lat": 39.7981,
  "lng": -89.6542,
  "settings": {
    "walking_threshold_miles": 1.0,
    "search_radius_km": 5.0,
    "active_categories": ["grocery", "restaurant", "park"]
  },
  "poi_snapshot": { "..." },
  "route_snapshot": { "..." }
}
```

**Response `201`:**
```json
{
  "id": "view-uuid",
  "created_at": "2026-02-27T14:00:00Z"
}
```

---

#### `GET /api/v1/views`

Returns all saved views for a session.

**Query params:** `session_id` (string, required)

**Response `200`:**
```json
{
  "views": [
    {
      "id": "view-uuid",
      "label": "123 Main St",
      "address": "123 Main St, Springfield, IL 62701",
      "lat": 39.7981,
      "lng": -89.6542,
      "created_at": "2026-02-27T14:00:00Z",
      "settings": { "..." }
    }
  ]
}
```

---

#### `GET /api/v1/views/{view_id}`

Returns full detail for a single saved view including POI and route snapshot data.

---

#### `DELETE /api/v1/views/{view_id}`

Deletes a saved view.

---

#### `GET /api/v1/compare`

Returns a structured comparison of two (or more) saved views.

**Query params:** `view_ids` (comma-separated; exactly 2 in v1; API accepts N for forward compatibility)

**Response `200`:**
```json
{
  "views": [
    { "id": "view-a-uuid", "label": "123 Main St", "address": "..." },
    { "id": "view-b-uuid", "label": "456 Oak Ave", "address": "..." }
  ],
  "comparison": {
    "grocery": [
      { "view_id": "view-a-uuid", "name": "Whole Foods", "distance_miles": 0.31, "duration_display": "6 min", "mode": "walk" },
      { "view_id": "view-b-uuid", "name": "Trader Joe's", "distance_miles": 1.2, "duration_display": "8 min", "mode": "drive" }
    ],
    "park": [ "..." ]
  }
}
```

> The comparison object is keyed by category with one entry per view. This structure naturally extends to N views by adding more entries per category — no API changes required.

---

#### `GET /api/v1/health`

Returns `{ "status": "ok", "db": "ok" }`. Used by the deployment platform for health checks.

---

### 6.2 Error Responses

All errors follow a consistent envelope:

```json
{
  "error": {
    "code": "GEOCODE_NOT_FOUND",
    "message": "No results found for the provided address.",
    "detail": null
  }
}
```

**Standard error codes:** `GEOCODE_NOT_FOUND`, `UPSTREAM_TIMEOUT`, `RATE_LIMITED`, `INVALID_PARAMS`, `VIEW_NOT_FOUND`, `INTERNAL_ERROR`.

---

## 7. Database Design

### 7.1 Schema

#### `geocode_cache`
```sql
CREATE TABLE geocode_cache (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    address_hash  TEXT NOT NULL UNIQUE,   -- SHA256 of normalized address
    address_input TEXT NOT NULL,
    display_name  TEXT,
    lat           DOUBLE PRECISION NOT NULL,
    lng           DOUBLE PRECISION NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at    TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '90 days'
);
CREATE INDEX ON geocode_cache (address_hash);
CREATE INDEX ON geocode_cache (expires_at);
```

#### `poi_cache`
```sql
CREATE TABLE poi_cache (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cache_key    TEXT NOT NULL UNIQUE,  -- SHA256 of (lat_r, lng_r, radius_m, category)
    lat_rounded  DOUBLE PRECISION NOT NULL,
    lng_rounded  DOUBLE PRECISION NOT NULL,
    radius_m     INTEGER NOT NULL,
    category     TEXT NOT NULL,
    result       JSONB NOT NULL,        -- Array of POI objects
    poi_count    INTEGER NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '7 days'
);
CREATE INDEX ON poi_cache (cache_key);
CREATE INDEX ON poi_cache (expires_at);
```

#### `route_cache`
```sql
CREATE TABLE route_cache (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cache_key        TEXT NOT NULL UNIQUE,  -- SHA256 of (orig_lat_r, orig_lng_r, dest_lat_r, dest_lng_r, mode)
    orig_lat         DOUBLE PRECISION NOT NULL,
    orig_lng         DOUBLE PRECISION NOT NULL,
    dest_lat         DOUBLE PRECISION NOT NULL,
    dest_lng         DOUBLE PRECISION NOT NULL,
    mode             TEXT NOT NULL CHECK (mode IN ('walk', 'drive')),
    distance_miles   DOUBLE PRECISION NOT NULL,
    duration_seconds INTEGER NOT NULL,
    geojson          JSONB NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at       TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '30 days'
);
CREATE INDEX ON route_cache (cache_key);
CREATE INDEX ON route_cache (expires_at);
```

#### `saved_views`
```sql
CREATE TABLE saved_views (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id     TEXT NOT NULL,          -- Client-generated UUID, stored in browser localStorage
    label          TEXT NOT NULL,
    address        TEXT NOT NULL,
    lat            DOUBLE PRECISION NOT NULL,
    lng            DOUBLE PRECISION NOT NULL,
    settings       JSONB NOT NULL,         -- { walking_threshold_miles, search_radius_km, active_categories[] }
    poi_snapshot   JSONB,                  -- Snapshot of POI results at save time
    route_snapshot JSONB,                  -- Snapshot of route results at save time
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON saved_views (session_id);
CREATE INDEX ON saved_views (created_at DESC);
```

> **Future auth migration:** Add a nullable `user_id UUID REFERENCES users(id)` column. When a user logs in, associate their `session_id` views with their account. No other schema changes needed.

### 7.2 Cache Key Strategy

Coordinates are rounded to 4 decimal places (~11m precision) before hashing for POI and route cache keys. This allows nearby lookups to share cache entries without meaningful accuracy loss.

```python
# Python example
import hashlib

def make_poi_cache_key(lat: float, lng: float, radius_m: int, category: str) -> str:
    lat_r = round(lat, 4)
    lng_r = round(lng, 4)
    raw = f"{lat_r}:{lng_r}:{radius_m}:{category}"
    return hashlib.sha256(raw.encode()).hexdigest()
```

### 7.3 Cache Expiry Policy

| Cache Table | TTL | Rationale |
|---|---|---|
| `geocode_cache` | 90 days | Addresses are stable; OSM data changes slowly |
| `poi_cache` | 7 days | Businesses open/close; weekly refresh is reasonable |
| `route_cache` | 30 days | Road networks change infrequently |

A background task runs nightly to remove expired entries:

```sql
DELETE FROM geocode_cache WHERE expires_at < now();
DELETE FROM poi_cache WHERE expires_at < now();
DELETE FROM route_cache WHERE expires_at < now();
```

---

## 8. Frontend Design

### 8.1 Component Hierarchy

```
<App>
├── <AddressSearch>          # Input with debounced geocode call
├── <MapView>                # React-Leaflet MapContainer
│   ├── <OriginMarker>       # Pin for the searched address
│   ├── <POIMarker>          # One per POI; click → popup with time/distance/mode
│   └── <RoutePolyline>      # GeoJSON route; blue = walk, orange = drive
├── <FilterPanel>            # Category toggles; hidden for categories with 0 results
├── <ThresholdSlider>        # Walk/drive distance threshold (0.25–3 mi)
├── <SavedViewsList>         # Sidebar; lists views; click to reload; delete
└── <ComparePanel>           # Active when 2 views selected; summary table
```

### 8.2 Zustand Store Shape

```typescript
type CategoryKey = 'grocery' | 'restaurant' | 'bar' | 'park' | 'cafe' |
                   'gym' | 'hospital' | 'school' | 'transit' | 'airport';

interface AppStore {
  // Search
  search: {
    address: string;
    coords: { lat: number; lng: number } | null;
    status: 'idle' | 'loading' | 'error';
    errorMessage: string | null;
  };

  // Settings
  settings: {
    walkingThresholdMiles: number;     // default: 1.0
    searchRadiusKm: number;            // default: 5.0
    activeCategories: Set<CategoryKey>;
  };

  // POI & Route Data
  pois: Record<CategoryKey, POI[]>;
  routes: Record<string, RouteResult>;        // keyed by `${poi.id}:${mode}`
  availableCategories: CategoryKey[];         // categories with >=1 result for current address

  // UI State
  ui: {
    selectedPOIId: string | null;
    compareViewIds: [string, string] | null;  // tuple for v1; extend to string[] for N-way
  };

  // Saved Views
  savedViews: SavedViewSummary[];
  sessionId: string;    // UUID generated on first load, persisted in localStorage

  // Actions
  setAddress: (address: string) => void;
  geocodeAddress: () => Promise<void>;
  fetchPOIs: () => Promise<void>;
  fetchRoute: (poi: POI) => Promise<void>;
  saveCurrentView: (label: string) => Promise<void>;
  loadSavedViews: () => Promise<void>;
  deleteSavedView: (id: string) => Promise<void>;
  setCompareViews: (ids: [string, string]) => void;
  updateSettings: (patch: Partial<AppStore['settings']>) => void;
}
```

> **Extensibility note:** `compareViewIds` is typed as `[string, string]` for v1. Changing it to `string[]` and updating the compare API call and `<ComparePanel>` to render N columns is the only change required to support more than 2 views.

### 8.3 Frontend Caching

The frontend maintains two cache layers to avoid redundant requests to the backend.

**Layer 1 — In-memory (Zustand)**
Holds geocode results, POI results, and routes for the current session. Survives navigation within the SPA. Lost on page refresh.

**Layer 2 — IndexedDB (via `idb-keyval`)**
Persists geocode and POI results across page refreshes. Route GeoJSON is not persisted to IndexedDB by default due to payload size. TTL is enforced client-side by storing a timestamp alongside each entry and evicting on read if expired.

```typescript
// Cache-aside pattern example
async function geocodeWithCache(address: string): Promise<Coords> {
  const key = `geocode:${normalizeAddress(address)}`;

  // L1: in-memory
  const inMemory = geocodeMemCache.get(key);
  if (inMemory) return inMemory;

  // L2: IndexedDB
  const stored = await idbGet(key);
  if (stored && stored.expires_at > Date.now()) {
    geocodeMemCache.set(key, stored.value);
    return stored.value;
  }

  // Cache miss — call backend
  const result = await api.geocode(address);
  geocodeMemCache.set(key, result);
  await idbSet(key, {
    value: result,
    expires_at: Date.now() + 7 * 24 * 60 * 60 * 1000   // 7 days
  });
  return result;
}
```

**Frontend Cache TTLs:**

| Data Type | In-Memory (L1) | IndexedDB (L2) |
|---|---|---|
| Geocode results | Session | 7 days |
| POI results | Session | 1 day |
| Route results | Session | Not persisted |

---

## 9. POI Categories & Overpass Queries

Each category maps to an Overpass QL filter executed by the backend. Categories with zero results for the current address are not shown in the UI.

| Key | Label | Overpass Filter | Search Radius |
|---|---|---|---|
| `grocery` | Grocery / Supermarket | `amenity=supermarket` OR `shop=supermarket` OR `shop=grocery` | 5 km (configurable) |
| `restaurant` | Restaurants | `amenity=restaurant` | 5 km |
| `bar` | Bars & Nightlife | `amenity=bar` OR `amenity=pub` OR `amenity=nightclub` | 5 km |
| `park` | Parks & Green Space | `leisure=park` OR `leisure=nature_reserve` | 5 km |
| `cafe` | Coffee Shops | `amenity=cafe` | 5 km |
| `gym` | Gym / Fitness | `leisure=fitness_centre` OR `leisure=sports_centre` | 5 km |
| `hospital` | Hospital / Urgent Care | `amenity=hospital` OR `amenity=clinic` | 5 km |
| `school` | Schools | `amenity=school` OR `amenity=university` | 5 km |
| `transit` | Transit Stop | `highway=bus_stop` OR `railway=station` OR `railway=tram_stop` | 5 km |
| `airport` | Airport | `aeroway=aerodrome` (filtered to nodes with `iata` tag — major airports only) | **48.3 km (30 miles, fixed)** |

**Airport special handling:**
- Always uses a fixed 30-mile (48.3 km) radius, regardless of the user's configured general radius.
- Filtered to OSM nodes/ways with an `iata` tag to exclude small private airfields.
- Always uses driving directions regardless of the walking threshold.

**Default result limit:** 5 nearest POIs per category (user-configurable 1–10 in settings).

**Overpass query template (Python):**
```python
def build_overpass_query(lat: float, lng: float, radius_m: int, filters: list[str]) -> str:
    filter_union = "\n  ".join(
        f'node[{f}](around:{radius_m},{lat},{lng});' for f in filters
    )
    return f"""
[out:json][timeout:15];
(
  {filter_union}
);
out body 10;
"""
```

---

## 10. Walking vs. Driving Logic

Travel mode is determined by comparing the straight-line haversine distance between the origin and each POI against the user's walking threshold.

| Condition | Mode | Map Style |
|---|---|---|
| Straight-line distance ≤ threshold | Walk | Blue polyline + walk icon |
| Straight-line distance > threshold | Drive | Orange polyline + car icon |
| Airport category (always) | Drive | Orange polyline + plane/car icon |

The default threshold is **1.0 mile**, adjustable via the `ThresholdSlider` component (range: 0.25–3.0 miles, step: 0.25 miles).

Changing the threshold re-evaluates all active POI modes. Both walk and drive routes are cached after first load, so toggling the threshold does not incur additional OSRM calls for previously-loaded POIs.

---

## 11. Caching Strategy

Caching is applied at three layers. The goal is that a returning user loading a previously-searched address incurs zero upstream API calls.

```
Request path (full cache miss):
  Browser L1 (in-memory) → Browser L2 (IndexedDB) → Backend L3 (PostgreSQL) → Upstream API

Request path (L1 hit):
  Browser L1 (in-memory) ✓  — zero network calls
```

### Summary Table

| Layer | Store | What's Cached | TTL | Scope |
|---|---|---|---|---|
| L1 | Zustand in-memory | Geocode, POIs, Routes | Session | Current tab |
| L2 | Browser IndexedDB | Geocode, POIs | 1–7 days | This browser |
| L3 | PostgreSQL (backend) | Geocode, POIs, Routes | 7–90 days | All users / all browsers |

### Cache Invalidation

- Backend entries expire via `expires_at`; a nightly cleanup job removes stale rows.
- Frontend entries store a timestamp; reads evict silently on expiry.
- Manual invalidation is not needed for v1 — stale POI data within the TTL window is an acceptable trade-off.

---

## 12. Saved Views & Comparison

### 12.1 Session Identity

Views are tied to a `session_id` — a UUID generated on the user's first visit and stored in `localStorage`. This allows views to persist across page refreshes and return visits without requiring user accounts.

> **Future auth migration:** Replace `session_id` with a `user_id` FK once OAuth is introduced. The `saved_views` schema is designed for this with a nullable `user_id` column addition.

### 12.2 Saving a View

A view snapshot captures: address, resolved coordinates, active category settings, distance threshold, and the full set of resolved POI and route data at save time. This ensures saved views always render consistently regardless of upstream data changes after the save date.

### 12.3 Comparing Two Views

The comparison UI allows the user to select exactly 2 saved views. The `GET /api/v1/compare` endpoint returns a structured comparison organized by category. The frontend renders a two-column summary table: one column per address, one row per category, showing nearest POI name, travel mode icon, distance, and travel time. Categories where neither address has results are omitted from the table.

**Extending to N views:** The API accepts `view_ids` as a list and returns an array of entries per category — not a hardcoded two-column structure. Supporting a third address requires: (1) removing the 2-ID validation on the query param, and (2) updating `<ComparePanel>` to render a third column. No backend schema or response format changes needed.

### 12.4 Map Behavior During Comparison

When comparison mode is active, the map shows a toggle between the two address maps with synchronized zoom levels. A future enhancement could offer a split-screen view — validate demand before building.

---

## 13. API Considerations & Rate Limits

| Service | Constraint | Backend Mitigation |
|---|---|---|
| Nominatim | 1 req/s; valid `User-Agent` required; no bulk geocoding | PostgreSQL cache (90-day TTL); 1s request queue; custom User-Agent header |
| Overpass API | Fair-use policy; no hard rate limit on public server | PostgreSQL cache (7-day TTL); coordinate rounding maximizes cache hit rate |
| OSRM Demo Server | No hard rate limit; not suitable for production traffic | PostgreSQL cache (30-day TTL); max 10 concurrent upstream calls; self-host before public launch |
| OpenStreetMap Tiles | Tile usage policy; no bulk fetching | Standard Leaflet usage is within policy; browser caches tiles automatically |

### Production Upgrade Path

1. **Geocoding:** Switch from public Nominatim to [Geoapify](https://www.geoapify.com/) or [LocationIQ](https://locationiq.com/) — both offer 5,000–10,000 free requests/day with higher reliability and SLAs.
2. **Routing:** Self-host OSRM on a Hetzner CX21 (~€5/month). A prebuilt North America OSM extract from Geofabrik loads in under an hour and handles production routing traffic comfortably.
3. **POI Search:** Self-host Overpass API if usage warrants, or switch to Geoapify's Places API (mirrors OSM data, simple REST interface, free tier available).

---

## 14. Phased Development Roadmap

### Phase 1 — Core MVP (Weeks 1–3)

- [ ] Project scaffold: Vite + React + Tailwind + React-Leaflet
- [ ] Backend scaffold: FastAPI (or Go/chi) + PostgreSQL + Alembic migrations
- [ ] Docker + docker-compose for local development
- [ ] `GET /geocode` with Nominatim proxy and DB cache
- [ ] `GET /pois` with Overpass proxy, DB cache, and empty-category response
- [ ] `GET /route` with OSRM proxy and DB cache
- [ ] Map display: origin marker, POI markers, route polylines on click
- [ ] Distance + duration in POI popup
- [ ] Filter panel hides categories with zero results for the current address

### Phase 2 — Customization & Persistence (Weeks 4–5)

- [ ] Full POI category panel with toggles (all 10 categories)
- [ ] Walking threshold slider
- [ ] Session ID generation and localStorage persistence
- [ ] `POST /views`, `GET /views`, `GET /views/{id}`, `DELETE /views/{id}` endpoints
- [ ] Saved views sidebar with reload and delete
- [ ] Frontend L2 cache (IndexedDB via `idb-keyval`)
- [ ] Airport 30-mile radius override
- [ ] Nightly cache cleanup job (Render cron or APScheduler)

### Phase 3 — Comparison & Polish (Weeks 6–7)

- [ ] `GET /compare` endpoint
- [ ] Two-view comparison panel with summary table
- [ ] Mobile-responsive layout
- [ ] Loading skeletons, error states, and empty states
- [ ] Accessible map interactions (keyboard navigation, ARIA labels)
- [ ] CI/CD pipeline (GitHub Actions → Render deploy on merge to main)

### Phase 4 — Scale Prep (Post-Validation)

- [ ] Replace OSRM demo server with self-hosted instance
- [ ] Geocoding provider upgrade (Geoapify or LocationIQ)
- [ ] User account model (OAuth) with `user_id` replacing `session_id` on saved views
- [ ] Backend rate limiting and abuse protection (`slowapi` for FastAPI or middleware for Go)
- [ ] Performance profiling of concurrent upstream OSRM requests under load
- [ ] Consider shorter `poi_cache` TTL (24–48h) for `restaurant` and `bar` categories

---

## 15. Open Questions

- **POI result limit:** The default of 5 nearest per category may miss a well-regarded option slightly further away. Consider a "show more" expansion in a later phase — nearest-N is sufficient for v1.
- **User accounts timing:** Session-based views work for the POC but frustrate users who switch browsers or clear cookies. Prioritize lightweight OAuth (Google) in Phase 4 or earlier based on user feedback.
- **OSRM concurrency:** With all 10 categories active and 5 POIs each, a cold address load triggers up to 50 OSRM route requests. The backend should cap concurrent upstream OSRM calls (suggested: 10) with a semaphore to avoid overwhelming the demo server.
- **Comparison UI layout:** The initial design uses a toggle between two maps. A split-screen view would be more useful but requires care on mobile — validate demand before building.
- **POI cache TTL for restaurants/bars:** A 7-day TTL may be too stale for frequently-changing categories. A per-category TTL override (24–48h for `restaurant` and `bar`) should be added to the `poi_cache` table design.

---

## 16. Success Metrics

| Metric | Target |
|---|---|
| **Time to First Map** | User sees a populated map with routes within 5 seconds of submitting an address on a standard connection |
| **Route Resolution Rate** | ≥ 90% of POIs successfully receive a route from OSRM without error |
| **Geocode Success Rate** | ≥ 95% of valid US addresses are successfully geocoded |
| **Backend Cache Hit Rate** | ≥ 70% of POI and route requests served from PostgreSQL cache after first week of usage |
| **Frontend Cache Hit Rate** | ≥ 50% of repeat address lookups served entirely from browser cache (zero backend calls) |
| **Mobile Usability** | Core flows (search, view map, filter categories, save view) completable on a 375px viewport without horizontal scroll |
| **Saved View Load Time** | Saved view loads within 2 seconds when all data is in cache |
