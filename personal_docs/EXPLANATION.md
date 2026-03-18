# 🧠 EduNation — API Architecture Deep Dive

A complete explanation of how data flows through the EduNation backend, what commands are used to query each data source, how the caching strategy works, and every optimization opportunity before deploying to production.

---

## 📖 Table of Contents

- [Route Map](#-route-map)
- [Data Source 1: REST Countries API](#-data-source-1-rest-countries-api)
- [Data Source 2: World Bank API](#-data-source-2-world-bank-api)
- [Data Source 3: Our World in Data (OWID)](#-data-source-3-our-world-in-data-owid)
- [Data Source 4: GeoJSON (GitHub)](#-data-source-4-geojson-github)
- [Data Source 5: Happiness JSON (Local File)](#-data-source-5-happiness-json-local-file)
- [Data Source 6: Supabase PostgreSQL (Direct)](#-data-source-6-supabase-postgresql-direct)
- [Three-Tier Caching Strategy](#-three-tier-caching-strategy)
- [Full Data Flow Diagrams](#-full-data-flow-diagrams)
- [Pre-Deployment Optimizations](#-pre-deployment-optimizations)

---

## 🗺️ Route Map

Every API endpoint and which controller/service handles it:

| Route | Controller | Service(s) Used | Data Source |
|---|---|---|---|
| `GET /api/countries` | `countryController.getCountries` | `restCountries.service` | REST Countries API |
| `GET /api/country/:code` | `countryController.getCountryByCode` | `restCountries.service` | REST Countries API |
| `GET /api/country/:code/neighbours` | `countryController.getCountryNeighbours` | `restCountries.service` | REST Countries API |
| `GET /api/country/:code/indicators` | `countryController.getCountryIndicators` | `worldBank.service` + `restCountries.service` + local JSON | Redis → Supabase DB → World Bank API + Local File |
| `GET /api/country/:code/analytics` | `countryController.getCountryAnalytics` | `owid.service` | OWID CSV (DigitalOcean) |
| `GET /api/geo` | `countryController.getGeoGeometry` | `geo.service` | GitHub raw GeoJSON |
| `GET /api/simulation/baseline/:code` | `simulationController.getSimulationBaseline` | Supabase direct + local JSON | Supabase `simulation_baselines` table |
| `GET /api/simulation/rankings` | `simulationController.getGlobalRankings` | Supabase direct | Supabase `simulation_baselines` table |
| `GET /api/indicators/:code/history` | `simulationController.getIndicatorHistory` | Supabase direct | Supabase `indicator_values` table |

---

## 🌐 Data Source 1: REST Countries API

**Base URL:** `https://restcountries.com/v3.1`

### What it provides
Country metadata: name, ISO codes, region, population, flag SVG, coordinates, borders.

### How it's called

```typescript
// Fetch ALL countries (globe view)
axios.get('https://restcountries.com/v3.1/all?fields=name,cca2,cca3,region,population,flags,latlng')

// Fetch ONE country by ISO3 code
axios.get('https://restcountries.com/v3.1/alpha/IND')

// Fetch MULTIPLE neighbours by comma-separated codes
axios.get('https://restcountries.com/v3.1/alpha?codes=CHN,PAK,BGD&fields=name,cca2,cca3,...')
```

### Caching behavior
- Cache key: `rest_countries_all`, or the full URL for single-country lookups
- TTL: **24 hours**
- Flow: Redis → Supabase `api_cache` → Live API

---

## 📊 Data Source 2: World Bank API

**Base URL:** `https://api.worldbank.org/v2`

### What it provides
25 economic/social indicators including GDP, population, life expectancy, inflation, unemployment, education spending, etc.

### How it's called

```typescript
// Fetch 50 data points for a specific indicator and country
axios.get('https://api.worldbank.org/v2/country/IND/indicator/NY.GDP.MKTP.CD?format=json&per_page=50')
```

The response format is always a two-element array:
```json
[
  { "page": 1, "pages": 1, "per_page": 50, "total": 25 },
  [
    { "date": "2023", "value": 3385090000000 },
    { "date": "2022", "value": 3150250000000 },
    ...
  ]
]
```

### The three-tier fetch inside `fetchIndicator()`

This is the **core function** of the entire backend. Here is exactly what happens when the frontend requests indicators:

```
Step 1: Redis check
  └─ redisClient.get('nh:wb_IND_NY.GDP.MKTP.CD')
  └─ If HIT → return immediately (~50ms)

Step 2: Supabase DB check (indicator_values table)
  └─ supabase.from('indicator_values')
       .select('year, value, country:country_id!inner(iso3), indicator:indicator_id!inner(code)')
       .eq('country.iso3', 'IND')
       .eq('indicator.code', 'NY.GDP.MKTP.CD')
       .order('year', { ascending: false })
  └─ If rows found → format to WB response shape → cache to Redis → return (~200ms)

Step 3: Live World Bank API call
  └─ axios.get('https://api.worldbank.org/v2/country/IND/indicator/NY.GDP.MKTP.CD?format=json&per_page=50')
  └─ Cache result in BOTH Redis AND Supabase api_cache → return (~800ms)
```

### Indicator map (all 25)

| Key | World Bank Code | Metric |
|---|---|---|
| `gdp` | `NY.GDP.MKTP.CD` | GDP (current US$) |
| `pop` | `SP.POP.TOTL` | Total population |
| `lifeExp` | `SP.DYN.LE00.IN` | Life expectancy at birth |
| `inflation` | `FP.CPI.TOTL.ZG` | Inflation (consumer prices %) |
| `unemployment` | `SL.UEM.TOTL.ZS` | Unemployment (% of labor force) |
| `gini` | `SI.POV.GINI` | GINI index |
| `fdi` | `BX.KLT.DINV.CD.WD` | Foreign direct investment |
| `military` | `MS.MIL.XPND.GD.ZS` | Military expenditure (% GDP) |
| `healthSpend` | `SH.XPD.CHEX.GD.ZS` | Health expenditure (% GDP) |
| `eduSpend` | `SE.XPD.TOTL.GD.ZS` | Education expenditure (% GDP) |
| `techExports` | `TX.VAL.TECH.CD` | High-tech exports (US$) |
| `internet` | `IT.NET.USER.ZS` | Internet users (% population) |
| `mobile` | `IT.CEL.SETS.P2` | Mobile subscriptions per 100 people |
| `poverty` | `SI.POV.DDAY` | Poverty headcount ratio |
| `electricity` | `EG.ELC.ACCS.ZS` | Access to electricity (%) |
| `renewables` | `EG.FEC.RNEW.ZS` | Renewable energy consumption (%) |
| `forest` | `AG.LND.FRST.ZS` | Forest area (% land) |
| `agriLand` | `AG.LND.AGRI.ZS` | Agricultural land (% land) |
| `urbanPop` | `SP.URB.TOTL.IN.ZS` | Urban population (%) |
| `literacy` | `SE.ADT.LITR.ZS` | Literacy rate (%) |
| `laborForce` | `SL.TLF.TOTL.IN` | Total labor force |
| `exports` | `NE.EXP.GNFS.CD` | Exports of goods & services |
| `imports` | `NE.IMP.GNFS.CD` | Imports of goods & services |
| `debt` | `GC.DOD.TOTL.GD.ZS` | Central govt debt (% GDP) |
| `tourism` | `ST.INT.ARVL` | International tourism arrivals |

### Concurrency
All 25 indicators are fetched using `Promise.all()` — meaning they fire **simultaneously**, not sequentially. If all 25 are cached in Redis, the entire response returns in around ~100ms total.

---

## 🌿 Data Source 3: Our World in Data (OWID)

**CO₂ URL:** `https://nyc3.digitaloceanspaces.com/owid-public/data/co2/owid-co2-data.csv`

### What it provides
Historical CO₂ emissions time-series (total and per-capita) for every country.

### How it's called

```typescript
// Downloads the ENTIRE CSV (~40MB), parses it, and extracts rows for the requested ISO3 code
const response = await axios.get(CO2_URL, { responseType: 'text' });
const records = parse(response.data, { columns: true, skip_empty_lines: true });
const countryData = records.filter((row) => row.iso_code === 'IND');
```

### ⚠️ Performance note
This is the **single heaviest operation** in the backend. It downloads a ~40MB CSV on every cache miss. See optimizations below.

### Caching behavior
- Cache key: `owid_co2_IND`
- TTL: **24 hours**
- Only the filtered + minimized country-specific data is cached, not the full CSV

---

## 🗺️ Data Source 4: GeoJSON (GitHub)

**URL:** `https://raw.githubusercontent.com/vasturiano/react-globe.gl/master/example/datasets/ne_110m_admin_0_countries.geojson`

### What it provides
Polygon boundaries for every country — used to render the 3D globe.

### How it's called

```typescript
axios.get(GEO_URL)
// Cached for 30 DAYS because country borders rarely change
```

### Caching behavior
- Cache key: `geo_countries_geojson`
- TTL: **720 hours (30 days)**

---

## 😊 Data Source 5: Happiness JSON (Local File)

**Path:** `backend/src/data/unified_happiness.json`

### What it provides
World Happiness Report scores from 2009–2024 for ~150 countries, unified from multiple raw sources.

### How it's called

```typescript
const hapPath = path.join(process.cwd(), 'src/data/unified_happiness.json');
const hapData = JSON.parse(fs.readFileSync(hapPath, 'utf8'));
const score = hapData['India']['2023']; // → 4.036
```

### Caching behavior
- **No external caching needed** — it's a local file read from disk (~1ms)
- Data is read synchronously with `fs.readFileSync`

---

## 🗄️ Data Source 6: Supabase PostgreSQL (Direct)

Used by the simulation controller for data that was pre-populated by the `ingest.ts` script.

### Queries used

```sql
-- Look up a country by ISO3
SELECT id, name, iso3 FROM countries WHERE iso3 = 'IND';

-- Get simulation baseline
SELECT * FROM simulation_baselines WHERE country_id = 42;

-- Get global rankings (sorted by GDP)
SELECT gdp, life_expectancy, co2_emissions, unemployment, countries(name, iso3)
FROM simulation_baselines ORDER BY gdp DESC;

-- Get indicator time-series history
SELECT year, value FROM indicator_values
WHERE country_id = 42 AND indicator_id = 7 ORDER BY year ASC;

-- Get indicator values for the three-tier fetch (Step 2)
SELECT year, value FROM indicator_values
INNER JOIN countries ON countries.id = indicator_values.country_id
INNER JOIN indicators ON indicators.id = indicator_values.indicator_id
WHERE countries.iso3 = 'IND' AND indicators.code = 'NY.GDP.MKTP.CD'
ORDER BY year DESC;
```

These are all executed via the **Supabase JS client** which internally calls the PostgREST API:

```typescript
supabase.from('indicator_values')
  .select('year, value, country:country_id!inner(iso3), indicator:indicator_id!inner(code)')
  .eq('country.iso3', 'IND')
  .eq('indicator.code', 'NY.GDP.MKTP.CD')
  .order('year', { ascending: false });
```

---

## 🏗️ Three-Tier Caching Strategy

The caching layer lives in `backend/src/utils/cache.ts` and wraps every external data fetch.

### `getCache(endpoint)` — Read path

```
1. Check Upstash Redis
   └─ redisClient.get(`nh:${endpoint}`)
   └─ Upstash auto-deserializes JSON
   └─ If HIT → return data

2. Check Supabase api_cache table
   └─ supabase.from('api_cache')
        .select('response, expires_at')
        .eq('endpoint', endpoint)
        .order('cached_at', { ascending: false })
        .limit(1).single()
   └─ If expired → return null
   └─ If VALID → backfill Redis with remaining TTL → return data

3. Return null (cache miss)
```

### `setCache(source, endpoint, response, ttlHours)` — Write path

```
1. Write to Redis
   └─ redisClient.set(`nh:${endpoint}`, response, { ex: ttlHours * 3600 })

2. Write to Supabase api_cache
   └─ DELETE existing row for this endpoint
   └─ INSERT new row with response JSON + expires_at timestamp
```

### Redis commands used
| Command | Usage |
|---|---|
| `GET nh:wb_IND_NY.GDP.MKTP.CD` | Read cached indicator data |
| `SET nh:wb_IND_NY.GDP.MKTP.CD {json} EX 86400` | Write with 24h TTL |

### Supabase api_cache commands used
| Operation | SQL equivalent |
|---|---|
| Read | `SELECT response, expires_at FROM api_cache WHERE endpoint = '...' ORDER BY cached_at DESC LIMIT 1` |
| Delete old | `DELETE FROM api_cache WHERE endpoint = '...'` |
| Write new | `INSERT INTO api_cache (source, endpoint, response, expires_at) VALUES (...)` |

---

## 🔄 Full Data Flow Diagrams

### Flow 1: User clicks a country on the globe

```
Frontend                        Backend                          External
━━━━━━━━                        ━━━━━━━                          ━━━━━━━━
User clicks India
    │
    ├─► GET /api/country/IND ──────► restCountries.service
    │                                   ├─ Redis GET nh:rest_countries_alpha_IND
    │                                   ├─ (miss) Supabase api_cache SELECT
    │                                   ├─ (miss) axios.get(restcountries.com/v3.1/alpha/IND)
    │                                   ├─ SET Redis + INSERT api_cache
    │                                   └─► return country metadata
    │
    ├─► GET /api/country/IND/indicators ──► countryController.getCountryIndicators
    │                                        ├─ Promise.all(25 indicators)
    │                                        │   ├─ fetchIndicator('IND', 'NY.GDP.MKTP.CD')
    │                                        │   │   ├─ Redis GET → miss
    │                                        │   │   ├─ Supabase indicator_values SELECT → HIT (from ingest.ts)
    │                                        │   │   ├─ SET Redis (backfill)
    │                                        │   │   └─ return formatted data
    │                                        │   ├─ fetchIndicator('IND', 'SP.POP.TOTL') ...
    │                                        │   └─ ... (x25 concurrent)
    │                                        ├─ Read unified_happiness.json from disk
    │                                        └─► return merged { gdp, pop, lifeExp, ..., happiness }
    │
    └─► Frontend renders dashboard with progressive spinners
```

### Flow 2: Same country requested again (fully cached)

```
Frontend                        Backend
━━━━━━━━                        ━━━━━━━
User clicks India (2nd time)
    │
    ├─► GET /api/country/IND/indicators
    │       ├─ Promise.all(25 indicators)
    │       │   ├─ Redis GET nh:wb_IND_NY.GDP.MKTP.CD → HIT ✅ (~5ms)
    │       │   ├─ Redis GET nh:wb_IND_SP.POP.TOTL → HIT ✅ (~5ms)
    │       │   └─ ... all 25 from Redis
    │       └─► return merged payload (~100ms total)
```

### Flow 3: Simulation baseline

```
Frontend                        Backend                          Supabase
━━━━━━━━                        ━━━━━━━                          ━━━━━━━━
Open simulation tab
    │
    └─► GET /api/simulation/baseline/IND
            ├─ Redis GET simulation_baseline_IND → miss
            ├─ Supabase: SELECT id FROM countries WHERE iso3 = 'IND'
            ├─ Supabase: SELECT * FROM simulation_baselines WHERE country_id = 42
            ├─ Read happiness from unified_happiness.json
            ├─ Supabase: SELECT population FROM countries WHERE id = 42
            ├─ SET Redis + INSERT api_cache (TTL: 6 hours)
            └─► return { gdp, pop, lifeExp, co2, happiness, ... }
```

---

## ⚡ Pre-Deployment Optimizations

### 🔴 Critical (Do Before Deploying)

#### 1. Lock down CORS
**Current:** `app.use(cors())` — allows ANY website to call your API.
**Fix:**
```typescript
app.use(cors({
  origin: 'https://edunation.vercel.app',
  credentials: true
}));
```

#### 2. Add rate limiting
No rate limiting exists. A bad actor could exhaust your Supabase/Redis free tier.
```bash
npm install express-rate-limit
```
```typescript
import rateLimit from 'express-rate-limit';
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 200 }));
```

#### 3. OWID CSV should be pre-cached at build time
The 40MB CSV download on every cache miss is unsustainable in production.
**Fix:** Run a build-time script that downloads the CSV once, extracts and saves per-country JSON files, then read from disk at runtime (similar to the happiness JSON approach).

---

### 🟡 Important (Highly Recommended)

#### 4. Add `compression` middleware
Responses (especially the 25-indicator payloads) can be large. Gzip them:
```bash
npm install compression
```
```typescript
import compression from 'compression';
app.use(compression());
```

#### 5. Add request validation with Helmet
```bash
npm install helmet
```
```typescript
import helmet from 'helmet';
app.use(helmet());
```

#### 6. Happiness JSON should be loaded once into memory
Currently `fs.readFileSync` is called on **every indicators request**. Instead:
```typescript
// At the top of countryController.ts
const hapData = JSON.parse(fs.readFileSync(hapPath, 'utf8'));
// Then reference hapData directly in handlers — no repeated disk I/O
```

#### 7. Add health check endpoint
For Render's health monitoring:
```typescript
app.get('/health', (req, res) => res.status(200).json({ status: 'ok', uptime: process.uptime() }));
```

---

### 🟢 Nice to Have (Polish)

#### 8. Cache warming on startup
Pre-fetch the most popular countries (USA, IND, CHN, GBR, DEU) on server boot so the first user never triggers a cold API call.

#### 9. Stale-while-revalidate
Instead of blocking on cache miss, return stale data and refresh in the background:
```
if (cached && expired) {
  refreshInBackground(endpoint); // async, don't await
  return cached; // serve stale instantly
}
```

#### 10. Add structured logging
Replace `console.log`/`console.error` with a structured logger like `pino`:
```bash
npm install pino pino-pretty
```
This gives you JSON logs with timestamps, request IDs, and log levels — essential for debugging production issues on Render.

#### 11. Connection pooling
Supabase JS client handles this internally, but if you ever move to raw `pg`, use `pg-pool` with `max: 10` connections.

#### 12. CDN for GeoJSON
The GeoJSON file (~500KB) is served through your Express server. In production, serve it directly from a CDN (e.g., Vercel Edge, Cloudflare R2) so it never hits your backend at all.

---

## 📊 Performance Budget Summary

| Operation | Cold (no cache) | Warm (Redis hit) |
|---|---|---|
| Country metadata | ~400ms | ~50ms |
| 25 indicators | ~3-5s (25 parallel WB API calls) | ~100ms |
| Simulation baseline | ~300ms (3 Supabase queries) | ~50ms |
| CO₂ analytics (OWID) | ~8-12s (40MB CSV download + parse) | ~50ms |
| GeoJSON | ~800ms | ~50ms |
| Global rankings | ~200ms | ~50ms |

After the first user hits a country, **every subsequent request for that country is served from Redis in under 100ms** for the next 24 hours.

---

*This document reflects the architecture as of March 2026.*
