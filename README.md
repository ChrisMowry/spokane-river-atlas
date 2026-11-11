# Spokane River Atlas

A web map and dashboard for the Spokane River watershed. The atlas shows, in one place:

- **River access locations**: put-ins, take-outs, parks, and fishing access
- **Water temperature sensors**: locations and recent and historical readings
- **Streamflow stations**: USGS gauges with discharge (cfs) and gage height (ft)
- **Drainage basin delineations**: watershed and sub-basin (HUC) boundaries
- **Monofilament recycling stations**: where anglers can drop off used fishing line
- **Other points of interest**: boat launches, hazards, portages, restrooms, parking, and so on

> **Status:** Planning and scaffolding. This README describes the target architecture and the local development workflow. Use it as the reference while the frontend and backend are built. Sections marked _(planned)_ don't exist yet.

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Architecture](#architecture)
3. [Repository Layout](#repository-layout)
4. [Frontend (Web UI)](#frontend-web-ui)
5. [Backend (Serverless API)](#backend-serverless-api)
6. [Data Model (DynamoDB)](#data-model-dynamodb)
7. [API Reference](#api-reference)
8. [Infrastructure (SAM / CloudFormation)](#infrastructure-sam--cloudformation)
9. [Running Locally](#running-locally)
10. [Deployment](#deployment)
11. [Environment Variables](#environment-variables)
12. [Conventions](#conventions)
13. [Roadmap](#roadmap)

---

## Tech Stack

| Layer | Technology |
| --- | --- |
| Monorepo | npm workspaces (`frontend`, `backend`, `packages/*`) |
| Frontend build | [Vite](https://vite.dev/) + React + TypeScript |
| UI components | [Material UI (MUI)](https://mui.com/) |
| Styling / utilities | [Tailwind CSS](https://tailwindcss.com/) (v4) alongside the MUI theme |
| Web map | [MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/) (open source, BSD-3) via [`react-map-gl/maplibre`](https://visgl.github.io/react-map-gl/) |
| Basemap | Vector tiles + style JSON: [OpenFreeMap](https://openfreemap.org/) to start, with an option to self-host [Protomaps](https://protomaps.com/) PMTiles on S3/CloudFront |
| Charts | MUI X Charts (time series for temperature and flow) |
| Data fetching | TanStack Query |
| Static hosting | Amazon S3 (private bucket) + Amazon CloudFront (Origin Access Control) |
| API | Amazon API Gateway (HTTP API) |
| Compute | AWS Lambda, Node.js 22.x, TypeScript, bundled with esbuild |
| Database | Amazon DynamoDB (single-table design, on-demand capacity) |
| Scheduled jobs | Amazon EventBridge Scheduler → Lambda (data ingestion) |
| Infrastructure as code | AWS SAM / CloudFormation |
| Local AWS emulation | SAM CLI (`sam local`) + DynamoDB Local (Docker) |

---

## Architecture

```
                               ┌──────────────────────────┐
                               │   Basemap style/tiles    │
                               │ (OpenFreeMap or PMTiles) │
                               └────────────▲─────────────┘
                                            │
┌──────────┐   HTTPS   ┌────────────────────┴───┐   OAC   ┌─────────────────────┐
│ Browser  │──────────▶│       CloudFront       │────────▶│ S3 (static frontend)│
│ (React)  │           │  /*      → S3 origin   │         └─────────────────────┘
│          │           │  /api/*  → API origin  │
└──────────┘           └───────────┬────────────┘
                                   │
                                   ▼
                       ┌────────────────────────┐
                       │ API Gateway (HTTP API) │
                       └───────────┬────────────┘
                                   │
                ┌──────────────────┼───────────────────┐
                ▼                  ▼                   ▼
        ┌──────────────┐   ┌──────────────┐    ┌──────────────┐
        │ Locations λ  │   │  Sensors λ   │    │ Stations λ   │   ... one Lambda per resource
        └──────┬───────┘   └──────┬───────┘    └──────┬───────┘
               └──────────────────┼───────────────────┘
                                  ▼
                       ┌────────────────────────┐
                       │     DynamoDB table     │
                       └───────────▲────────────┘
                                   │ writes readings
                       ┌───────────┴────────────┐      ┌─────────────────────────┐
                       │ Ingestion λ (scheduled)│─────▶│ USGS Water Data APIs,   │
                       │  EventBridge Scheduler │      │ sensor providers, etc.  │
                       └────────────────────────┘      └─────────────────────────┘
```

**Key design decisions**

- **One domain for UI and API.** CloudFront serves the SPA from S3 and forwards `/api/*` to API Gateway, so the browser makes same-origin requests and production needs no CORS. During local development, a Vite dev-server proxy gives the same `/api` behavior.
- **Private S3 bucket.** CloudFront reaches the bucket through Origin Access Control (OAC). The bucket has no public access and no static-website hosting.
- **SPA routing.** CloudFront maps 403 and 404 responses from the S3 origin to `/index.html` with status 200, so client-side routes work on refresh.
- **GeoJSON everywhere.** API responses for spatial resources are GeoJSON `FeatureCollection`s, which MapLibre GeoJSON sources consume directly.
- **Large, static geometry goes to S3, not DynamoDB.** Basin polygons can exceed DynamoDB's 400 KB item limit. They're stored as GeoJSON (or vector tiles) in S3 and served through CloudFront. DynamoDB stores only basin metadata.
- **Ingestion is separate from the read path.** Scheduled Lambdas pull sensor and streamflow data into DynamoDB, so user requests never wait on third-party APIs.

---

## Repository Layout

_(planned)_

```
spokane-river-atlas/
├── package.json                 # root: npm workspaces + shared scripts
├── tsconfig.base.json           # shared TypeScript compiler options
├── docker-compose.yml           # DynamoDB Local (+ optional admin UI)
├── .nvmrc                       # Node version (22)
├── README.md
│
├── frontend/                    # Vite + React + TS web UI
│   ├── index.html
│   ├── vite.config.ts           # includes /api proxy for local dev
│   ├── .env.example
│   ├── public/
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── theme/
│       │   ├── theme.config.ts  # ← edit colors and fonts here
│       │   ├── muiTheme.ts      # builds the MUI theme from theme.config
│       │   └── cssVariables.ts  # exposes theme tokens to Tailwind as CSS vars
│       ├── styles/
│       │   └── index.css        # Tailwind entry + CSS layer order
│       ├── map/                 # MapLibre map, layers, popups, legend
│       ├── features/            # access, sensors, streamflow, basins, recycling
│       ├── components/          # shared UI components
│       ├── api/                 # typed API client + TanStack Query hooks
│       └── pages/               # MapPage (A desktop / D mobile), DashboardPage (C), StationPage, AboutPage
│
├── backend/                     # Serverless API (SAM)
│   ├── template.yaml            # SAM / CloudFormation template
│   ├── samconfig.toml           # per-environment deploy settings
│   ├── env.local.json           # env overrides for `sam local`
│   ├── events/                  # sample events for `sam local invoke`
│   ├── src/
│   │   ├── handlers/            # one file per Lambda entry point
│   │   │   ├── locations.ts
│   │   │   ├── sensors.ts
│   │   │   ├── stations.ts
│   │   │   ├── basins.ts
│   │   │   └── ingest-streamflow.ts
│   │   ├── lib/                 # DynamoDB client, response helpers, validation
│   │   └── repositories/        # data-access per entity
│   ├── scripts/
│   │   ├── create-local-table.ts
│   │   └── seed.ts              # loads seed data into DynamoDB (local or AWS)
│   ├── seed-data/               # GeoJSON/JSON seed files
│   └── tests/
│
├── infrastructure/              # frontend hosting stack (S3 + CloudFront)
│   └── frontend-hosting.yaml
│
└── packages/
    └── shared/                  # types shared by frontend and backend
        └── src/
            ├── models.ts        # Location, Sensor, Reading, Station, Basin…
            └── geojson.ts
```

The `packages/shared` workspace keeps API contracts typed end to end. Both the Lambdas and the React client import the same `Location`, `Station`, and `Reading` types.

---

## Frontend (Web UI)

### UI design

The approved layouts are options **A**, **C**, and **D** on the [layout options canvas](https://claude.ai/artifact/8JwnaQXhsY3yskz48H7sr1). Option B was not selected. The mockups use sample data; the layout, components, and behavior described below are what to build.

| Route | Page | Desktop / tablet (≥ `md`, 900px) | Phone (< `md`) |
| --- | --- | --- | --- |
| `/` | Map | **A · Map-first explorer** | **D · Mobile bottom sheet** |
| `/dashboard` | Dashboard | **C · Dashboard-first** | C, single column |
| `/stations/:id` | Station detail | C's chart + table pattern for one station | Single column |
| `/about` | About | Simple content page | Same |

#### A · Map page (desktop)

- **App bar** (`primary` background): logo and title, nav links (Map, Dashboard, Stations, About), and a search field for places and stations.
- **Left panel** (~320px, MUI `Drawer` variant `permanent`):
  - **Map layers:** a checkbox per layer with its marker symbol and feature count. The basins row has a HUC 8 / 10 / 12 level picker. "Other points of interest" is off by default.
  - **In this view:** a list of features inside the current map bounds, updated on `moveend`. Selecting an item flies to it and opens its popup.
- **Map** (fills the remaining space):
  - Top left: basemap switcher (Streets / Topo / Light).
  - Top right: zoom in, zoom out, and locate-me controls.
  - Bottom: scale bar and the required OpenFreeMap / OpenStreetMap attribution.
- **Feature popup:** the type label, name, and source ID, then the latest values. A streamflow station shows discharge, gage height, and water temperature, plus a 7-day sparkline. The popup has **View station** (to `/stations/:id`) and **Directions** buttons.

#### C · Dashboard page

- **Left sidebar navigation** (dark): Dashboard, Map, Streamflow, Temperature, Access & amenities, Basins. On narrow screens it collapses to a top bar with a menu button.
- **Header row:** the page title, a time-range toggle (24 h / 7 d / 30 d / 1 y) that applies to every chart on the page, and a °C / °F unit picker.
- **Summary tiles:** flow at Spokane, flow near Post Falls, average water temperature, and number of sites reporting. Status (such as "1 sensor offline") always uses an icon and text as well as color.
- **Discharge chart:** one series for the selected time range, drawn over a shaded band showing the typical range for that date. It has a hover crosshair with a tooltip, a direct label on the latest value, and a **View as table** link. Use a single y-axis; put temperature in its own chart, never on a second axis.
- **Overview map:** a small, non-interactive map of the monitoring sites with an **Open full map** link.
- **Monitoring sites table:** site, type (marker symbol plus label), latest value, 24 h change, and last-updated time. Offline sites are flagged. The table scrolls horizontally on narrow screens.

#### D · Mobile map

- **Full-screen map.** It has a floating search bar with a layers button, plus a row of quick-filter buttons (Access, Flow, Temp, Recycling) that scrolls sideways.
- **Bottom sheet.** Selecting a feature opens it with the same content as the A popup. The sheet can be dragged to expand it, so it shows a peek on the map and the full details when expanded.
- **Bottom navigation:** Map, Conditions (the dashboard), Nearby (a list sorted by distance from your location), and About.
- **Locate-me button** sits above the sheet. Touch targets are at least 44px.

#### Map symbology

Every layer uses a distinct **shape** and a distinct **color**, so layers stay distinguishable for colorblind users and in print. The colors were checked with a color-vision-deficiency validator and are defined in `themeConfig.layers`.

| Layer | Marker | Color |
| --- | --- | --- |
| River access | Circle | `#1E6FB8` |
| Temperature sensor | Triangle | `#E4572E` |
| Streamflow station | Diamond | `#00968F` |
| Monofilament recycling | Rounded square | `#8E44AD` |
| Drainage basins | Dashed outline, 7% fill | `#7A9A1E` |
| Other points of interest | Hollow circle | `#6C757D` |

Markers have a 2px white outline so they stay readable on any basemap. The selected feature gets a halo ring. Layer colors are only for marks; text labels and values always use the normal text colors.

#### Shared components

- `LayerToggleList`, `FeatureList`, and `FeaturePopup`. On phones, `FeaturePopup` renders inside `BottomSheet`.
- `StatTile` and `Sparkline`.
- `TimeSeriesChart`, built on MUI X Charts, with the typical-range band, tooltip, and table view.
- `SitesTable` and `TimeRangeToggle`.
- `MarkerSymbol`: an SVG of each layer's shape and color, used in the legend, lists, and tables.

### Using MUI and Tailwind together

MUI provides the components and the design system (theme, palette, typography). Tailwind handles layout and one-off utility styling. To keep the two from fighting over specificity:

- **CSS cascade layers.** `src/styles/index.css` declares the layer order so Tailwind utilities can override MUI component styles:
  ```css
  @layer theme, base, mui, components, utilities;
  @import "tailwindcss";
  ```
- **MUI styles go into the `mui` layer.** Wrap the app in `StyledEngineProvider` with `enableCssLayer` (MUI v7+), and keep MUI's `CssBaseline` as the single CSS reset.
- **Shared tokens.** Tailwind colors and fonts point at CSS variables generated from the same theme config as MUI, so `bg-primary` in Tailwind and `color="primary"` in MUI always match.

### Configuring theme colors and fonts

All branding lives in **`frontend/src/theme/theme.config.ts`**, a single source of truth that feeds both the MUI theme and the Tailwind tokens.

```ts
// frontend/src/theme/theme.config.ts
export const themeConfig = {
  colors: {
    primary:   '#1F5F8B', // river blue
    secondary: '#3F7D58', // pine green
    accent:    '#D98E04', // basalt / sunset amber
    error:     '#C62828',
    warning:   '#ED6C02',
    info:      '#0288D1',
    success:   '#2E7D32',
  },
  // Map layer colors, also used in the legend and charts (see "Map symbology").
  // Re-check color-vision-deficiency separation if you change these.
  layers: {
    access:      '#1E6FB8',
    temperature: '#E4572E',
    streamflow:  '#00968F',
    recycling:   '#8E44AD',
    basins:      '#7A9A1E',
    poi:         '#6C757D',
  },
  typography: {
    fontFamily: '"Inter", "Roboto", "Helvetica", "Arial", sans-serif',
    headingFontFamily: '"Montserrat", "Inter", sans-serif',
    baseFontSize: 14,
  },
  shape: { borderRadius: 8 },
  defaultMode: 'light' as 'light' | 'dark' | 'system',
};
```

**To change the theme:**

1. Edit the hex values in `colors` and `layers`.
2. To change the font, update `typography.fontFamily` and `headingFontFamily`, then load the font. Either add a Google Fonts `<link>` to `frontend/index.html` or install a self-hosted package (for example, `npm i @fontsource/inter -w frontend`) and import it in `main.tsx`.
3. Restart `npm run dev` if the Vite server doesn't hot-reload the change.

`muiTheme.ts` turns this config into `createTheme({ palette, typography, shape, cssVariables: true })`. `cssVariables.ts` writes the same values as `--color-primary`, `--font-sans`, and so on, which Tailwind's `@theme` block references. Light and dark mode are both supported through MUI color schemes.

### MapLibre

- The app uses `maplibre-gl` with the MapLibre entry point of `react-map-gl`:
  ```ts
  import Map, { Source, Layer, Popup, NavigationControl } from 'react-map-gl/maplibre';
  import 'maplibre-gl/dist/maplibre-gl.css';
  ```
- **No access token or account is required.** MapLibre is open source and only needs a style URL, which is read from `VITE_MAP_STYLE_URL`.
- **Basemap options:**

  | Option | Style URL | Notes |
  | --- | --- | --- |
  | OpenFreeMap (default) | `https://tiles.openfreemap.org/styles/liberty` | Free, no key. Other styles include `bright` and `positron`. Good for development and v1. |
  | Self-hosted Protomaps PMTiles _(planned)_ | Style JSON in our own S3 bucket | Extract a Spokane-area `.pmtiles` file, host it on S3/CloudFront, and load it with the `pmtiles` protocol plugin. There's no third-party dependency and costs stay predictable. |
  | Commercial provider (MapTiler, Stadia, and others) | Provider style URL + API key | Use this for terrain, hillshade, or satellite imagery. Keys ship in the browser, so restrict them by domain. |

- **Attribution.** OpenStreetMap-based tiles require visible attribution. Keep MapLibre's `AttributionControl` enabled.
- **Search and geocoding.** MapLibre has no built-in geocoder. Search filters the atlas's own locations on the client. For address search, add the `@maplibre/maplibre-gl-geocoder` control backed by a geocoding service, and check that service's usage policy.
- Point layers come from API GeoJSON. Basin polygons are loaded from static GeoJSON or vector tiles hosted on S3/CloudFront.
- Map layer paint colors come from `themeConfig.layers`, so the map, legend, and charts share one palette.
- Initial view: centered on the Spokane River (about `-117.42, 47.66`, zoom ~9), with bounds covering the river from Lake Coeur d'Alene to its confluence with the Columbia at Lake Roosevelt.

---

## Backend (Serverless API)

### Overview

- **API Gateway HTTP API** routes requests to **one Lambda per resource**. Each Lambda handles the routes for its resource (list, get by ID, readings).
- Lambdas are written in **TypeScript on Node.js 22.x**. SAM bundles them with **esbuild** (`Metadata: BuildMethod: esbuild`), so no separate build step is needed.
- Data access uses AWS SDK v3 (`@aws-sdk/client-dynamodb` + `@aws-sdk/lib-dynamodb`).
- Input validation uses `zod`, and logging, tracing, and metrics use AWS Lambda Powertools for TypeScript.
- When `DYNAMODB_ENDPOINT` is set, the DynamoDB client points at that endpoint instead of AWS. This is how local development uses DynamoDB Local.

```ts
// backend/src/lib/dynamo.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient(
  process.env.DYNAMODB_ENDPOINT ? { endpoint: process.env.DYNAMODB_ENDPOINT } : {},
);
export const ddb = DynamoDBDocumentClient.from(client);
export const TABLE_NAME = process.env.TABLE_NAME!;
```

### Data sources (ingestion)

| Data | Source | Ingestion |
| --- | --- | --- |
| Streamflow | USGS Water Data APIs (`api.waterdata.usgs.gov`; the legacy NWIS `waterservices.usgs.gov` services are being retired). Example gauges: 12419000 Spokane River near Post Falls, ID; 12422500 Spokane River at Spokane, WA. | `ingest-streamflow` Lambda, every 15–60 min |
| Water temperature | USGS gauges that report temperature (parameter `00010`), plus partner or agency sensors as they're identified | `ingest-temperature` Lambda (planned) |
| Basin boundaries | USGS Watershed Boundary Dataset (WBD), HUC-8/10/12 within the Spokane basin (HUC-8 `17010305` Upper Spokane, `17010307` Lower Spokane, among others) | One-time preprocessing into simplified GeoJSON on S3 |
| Access, recycling, POIs | Curated seed data (GeoJSON) maintained in `backend/seed-data/` | `npm run seed` |

> Verify gauge IDs, HUC codes, and API endpoints against the live USGS services before you implement ingestion.

---

## Data Model (DynamoDB)

The model uses a single table (`spokane-river-atlas-<env>`) with on-demand billing and generic keys (`PK`, `SK`) plus one GSI.

| Entity | PK | SK | GSI1PK | GSI1SK | Notes |
| --- | --- | --- | --- | --- | --- |
| Location (access, recycling, poi) | `LOC#<id>` | `META` | `TYPE#<type>` | `LOC#<id>` | `name`, `lat`, `lng`, `type`, `amenities[]`, `description` |
| Sensor (temperature) | `SENSOR#<id>` | `META` | `TYPE#sensor` | `SENSOR#<id>` | `name`, `lat`, `lng`, `provider`, `units` |
| Sensor reading | `SENSOR#<id>` | `READ#<ISO-8601 timestamp>` | — | — | `value`, `units`, `ttl` (optional retention) |
| Streamflow station | `STATION#<usgsId>` | `META` | `TYPE#station` | `STATION#<usgsId>` | `name`, `lat`, `lng`, `parameters[]` |
| Station reading | `STATION#<usgsId>` | `READ#<ISO-8601 timestamp>` | — | — | `dischargeCfs`, `gageHeightFt`, `waterTempC` |
| Basin (metadata) | `BASIN#<huc>` | `META` | `TYPE#basin` | `BASIN#<huc>` | `name`, `hucLevel`, `areaSqMi`, `geometryUrl` (S3) |

**Access patterns**

| Pattern | Query |
| --- | --- |
| List all locations of a type | `GSI1` where `GSI1PK = TYPE#<type>` |
| Get one item | `GetItem` with `PK = <prefix>#<id>`, `SK = META` |
| Readings for a time range | `Query` where `PK = SENSOR#<id>` and `SK BETWEEN READ#<from> AND READ#<to>` |
| Latest reading | `Query` where `PK = …` and `SK begins_with READ#`, `ScanIndexForward=false`, `Limit=1` |

---

## API Reference

The base path is `/api` (same-origin through CloudFront in production, proxied by Vite locally).

| Method | Path | Description |
| --- | --- | --- |
| GET | `/api/health` | Health check |
| GET | `/api/locations?type=access\|recycling\|poi` | Locations as a GeoJSON `FeatureCollection` |
| GET | `/api/locations/{id}` | Single location |
| GET | `/api/sensors` | Temperature sensors (GeoJSON), each with its latest reading |
| GET | `/api/sensors/{id}/readings?from=&to=` | Time-series readings for a sensor |
| GET | `/api/stations` | Streamflow stations (GeoJSON), each with its latest reading |
| GET | `/api/stations/{id}/readings?from=&to=` | Time-series discharge and gage height |
| GET | `/api/basins` | Basin metadata, including the URL of each basin's geometry file |
| GET | `/api/dashboard/summary` | Aggregated current conditions for the dashboard |

The read API is public and unauthenticated in v1. Admin write endpoints (`POST`/`PUT`/`DELETE`) for curating locations are planned, protected by Amazon Cognito (a JWT authorizer on the HTTP API).

---

## Infrastructure (SAM / CloudFormation)

The infrastructure is split into two stacks per environment (`dev`, `prod`):

### 1. `backend/template.yaml`: API stack

- `AWS::Serverless::HttpApi`, stage `$default`, throttling configured
- `AWS::Serverless::Function` per handler: `Runtime: nodejs22.x`, `Architectures: [arm64]`, esbuild metadata, least-privilege `DynamoDBReadPolicy` / `DynamoDBCrudPolicy`
- `AWS::DynamoDB::Table`: `BillingMode: PAY_PER_REQUEST`, `PK`/`SK` keys, `GSI1`, point-in-time recovery, TTL on `ttl`
- Ingestion functions with `ScheduleV2` (EventBridge Scheduler) events
- `Globals` for shared environment variables (`TABLE_NAME`, `STAGE`, `LOG_LEVEL`)
- Parameters: `Stage`, `AllowedOrigin`
- Outputs: `ApiUrl`, `ApiDomain`, `TableName`

### 2. `infrastructure/frontend-hosting.yaml`: static site stack

- `AWS::S3::Bucket`: all public access blocked, encryption enabled, versioning on
- `AWS::CloudFront::OriginAccessControl` + bucket policy that allows only the distribution
- `AWS::CloudFront::Distribution`:
  - Default behavior → S3 origin (`CachingOptimized`), `index.html` as the default root object
  - `/api/*` behavior → API Gateway origin (`CachingDisabled`, `AllViewerExceptHostHeader` origin request policy)
  - Custom error responses: 403/404 → `/index.html` (200) for SPA routing
  - Optional: custom domain + ACM certificate (which must be in `us-east-1`) + Route 53 alias
- Parameters: `Stage`, `ApiDomain` (output of the API stack), optional `DomainName`, `CertificateArn`
- Outputs: `BucketName`, `DistributionId`, `DistributionDomainName`

Deploy the API stack first. Its `ApiDomain` output is passed into the hosting stack.

---

## Running Locally

### Prerequisites

| Tool | Version | Notes |
| --- | --- | --- |
| [Node.js](https://nodejs.org/) | 22.x LTS | `nvm use` reads `.nvmrc` (on Windows, use `nvm-windows` or `fnm`) |
| npm | 10+ | Ships with Node |
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | latest | Required for DynamoDB Local and `sam local` |
| [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) | latest | `aws configure` (only needed for deploying) |
| [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) | latest | `sam --version` |

### 1. Clone and install

```bash
git clone <repo-url> spokane-river-atlas
cd spokane-river-atlas
npm install              # installs all workspaces
```

### 2. Configure environment

```bash
cp frontend/.env.example frontend/.env.local
# the defaults work as-is. Optionally change VITE_MAP_STYLE_URL to use a different basemap.
```

`backend/env.local.json` (committed, contains no secrets) points the Lambdas at DynamoDB Local:

```json
{
  "Parameters": {
    "TABLE_NAME": "spokane-river-atlas-local",
    "DYNAMODB_ENDPOINT": "http://dynamodb-local:8000",
    "STAGE": "local",
    "LOG_LEVEL": "debug"
  }
}
```

### 3. Start DynamoDB Local

`docker-compose.yml` runs DynamoDB Local on a named Docker network (`sra-local`). Because `sam local` containers join the same network, Lambdas can reach the database at `http://dynamodb-local:8000`.

```bash
npm run db:up            # docker compose up -d   (DynamoDB Local on :8000)
npm run db:create        # creates the local table + GSI
npm run db:seed          # loads backend/seed-data into the local table
```

Optional: `npm run db:admin` starts `dynamodb-admin` at http://localhost:8001 so you can browse the table.

### 4. Start the API

```bash
npm run dev:api
# = sam build --cached && sam local start-api \
#     --port 3001 \
#     --env-vars env.local.json \
#     --docker-network sra-local \
#     --warm-containers EAGER
```

The API is now at http://localhost:3001/api. To check it:

```bash
curl http://localhost:3001/api/health
curl "http://localhost:3001/api/locations?type=access"
```

To invoke a single function, such as an ingestion job, with a sample event:

```bash
cd backend
sam local invoke IngestStreamflowFunction --event events/schedule.json \
  --env-vars env.local.json --docker-network sra-local
```

### 5. Start the frontend

```bash
npm run dev:web          # vite on http://localhost:5173
```

`frontend/vite.config.ts` proxies `/api` to the local API, which matches the CloudFront routing used in production:

```ts
server: {
  proxy: { '/api': { target: 'http://localhost:3001', changeOrigin: true } },
},
```

### 6. Everything at once

```bash
npm run dev              # db:up + dev:api + dev:web in parallel (concurrently)
```

> **Faster frontend-only work:** set `VITE_API_BASE_URL` to a deployed `dev` API, or enable the mock data layer (`VITE_USE_MOCKS=true`, planned with MSW) to skip Docker and SAM entirely.

### Root scripts _(planned)_

| Script | Description |
| --- | --- |
| `npm run dev` | DynamoDB Local + API + web UI |
| `npm run dev:web` | Vite dev server |
| `npm run dev:api` | `sam build` + `sam local start-api` |
| `npm run db:up` / `db:down` | Start/stop DynamoDB Local |
| `npm run db:create` | Create local table |
| `npm run db:seed` | Seed local table |
| `npm run build` | Type-check and build all workspaces |
| `npm run lint` | ESLint across the repo |
| `npm run test` | Vitest (frontend + backend unit tests) |
| `npm run deploy:api -- --config-env dev` | `sam build && sam deploy` |
| `npm run deploy:web -- --env dev` | Build frontend, sync to S3, invalidate CloudFront |

---

## Deployment

### Backend (API stack)

```bash
cd backend
sam build
sam deploy --guided --config-env dev     # first time; writes samconfig.toml
sam deploy --config-env dev              # subsequent deploys
```

### Frontend hosting stack (first time per environment)

```bash
aws cloudformation deploy \
  --template-file infrastructure/frontend-hosting.yaml \
  --stack-name spokane-river-atlas-web-dev \
  --parameter-overrides Stage=dev ApiDomain=<ApiDomain output from API stack>
```

### Frontend build and publish

```bash
npm run build -w frontend

aws s3 sync frontend/dist s3://<BucketName> --delete \
  --cache-control "public,max-age=31536000,immutable" --exclude index.html
aws s3 cp frontend/dist/index.html s3://<BucketName>/index.html \
  --cache-control "no-cache"

aws cloudfront create-invalidation --distribution-id <DistributionId> --paths "/index.html"
```

Hashed assets are cached for a year, and `index.html` is never cached, so a deploy only needs to invalidate `index.html`.

### CI/CD _(planned)_

GitHub Actions authenticates to AWS with OIDC (no long-lived keys). The pipeline runs lint and tests, then `sam deploy`, and then the frontend publish. `main` deploys to `dev`, and tagged releases deploy to `prod`.

---

## Environment Variables

### Frontend (`frontend/.env.local`; only `VITE_*` variables are exposed to the browser)

| Variable | Required | Example | Description |
| --- | --- | --- | --- |
| `VITE_MAP_STYLE_URL` | no | `https://tiles.openfreemap.org/styles/liberty` | MapLibre style JSON URL (default: OpenFreeMap Liberty) |
| `VITE_API_BASE_URL` | no | `/api` | API base URL (default `/api`) |
| `VITE_USE_MOCKS` | no | `false` | Use mock data instead of the API |

### Backend (set in `template.yaml` Globals / `env.local.json`)

| Variable | Description |
| --- | --- |
| `TABLE_NAME` | DynamoDB table name |
| `DYNAMODB_ENDPOINT` | Local only: DynamoDB Local endpoint |
| `STAGE` | `local` \| `dev` \| `prod` |
| `LOG_LEVEL` | Powertools log level |
| `USGS_API_BASE_URL` | USGS Water Data API base URL (ingestion) |

Never commit secrets. Keep `.env.local` gitignored. Any secret API keys a backend data provider needs go in AWS Systems Manager Parameter Store or Secrets Manager and are referenced from the template.

---

## Conventions

- **TypeScript strict mode** in every workspace. Shared types live in `packages/shared`.
- **ESLint + Prettier** at the root, and **Vitest** for unit tests (React Testing Library on the frontend).
- **Coordinates** are stored as WGS84 `lat`/`lng` numbers and emitted as GeoJSON `[lng, lat]`.
- **Timestamps** are ISO-8601 UTC strings. Convert to `America/Los_Angeles` only for display.
- **Units:** flow in cfs, gage height in ft, and temperature stored in °C. The UI lets users toggle °F/°C.
- **Resource naming:** `spokane-river-atlas-<resource>-<stage>`.
- **Tags** on all AWS resources: `Project=spokane-river-atlas`, `Stage=<stage>`.

---

## Roadmap

- [ ] Scaffold the monorepo (npm workspaces, shared tsconfig, lint and test tooling)
- [ ] Frontend: Vite + React + TS + MUI + Tailwind, theme config, app shell
- [ ] Map page, desktop (layout A): layer panel, "In this view" list, feature popup
- [ ] Map page, mobile (layout D): bottom sheet, quick filters, bottom navigation
- [ ] Dashboard page (layout C): stat tiles, discharge chart, sites table
- [ ] Self-hosted Protomaps PMTiles basemap on S3/CloudFront
- [ ] Backend: SAM template, DynamoDB table, `health` and `locations` endpoints
- [ ] Seed data: access points and monofilament recycling stations
- [ ] Streamflow station ingestion (USGS) and the station detail chart
- [ ] Temperature sensor ingestion and the dashboard
- [ ] Basin delineations (WBD → simplified GeoJSON on S3)
- [ ] Frontend hosting stack (S3 + CloudFront + OAC) and deploy scripts
- [ ] CI/CD with GitHub Actions + OIDC
- [ ] Admin editing (Cognito-protected write endpoints)
