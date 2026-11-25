# 📍 Where We Are: GoAround Project Status

> **Executive Summary**: GoAround is a realistic local outing planner for Pune students and young professionals. The frontend UI, design, form controls, local trip saving, and Google Maps links are **built and working**. The backend schema and GeoJSON processing pipeline are **ready**, but the application is currently running against mock/local fallback data until the Supabase database migration and seed script are executed in the remote Supabase project.

---

## 🚦 Feature Status at a Glance

| Feature | Status | Notes | Key Files |
| :--- | :--- | :--- | :--- |
| **Landing Page & Hero** | 🟢 Completed & Working | Responsive UI, quick-planner bar, trust badges | [HeroSection.tsx](file:///c:/Users/USER/GoAround-final/src/components/HeroSection.tsx) |
| **Trip Planning Form** | 🟢 Completed & Working | 8 Pune hubs, 4 durations, 4 transport modes, budget & group controls | [TripPlannerForm.tsx](file:///c:/Users/USER/GoAround-final/src/components/TripPlannerForm.tsx) |
| **Curated Trails** | 🟢 Completed & Working | 4 ready-made outing templates with realistic budgets | [TemplatesSection.tsx](file:///c:/Users/USER/GoAround-final/src/components/TemplatesSection.tsx) |
| **Trip Itinerary View** | 🟢 Completed & Working | Metrics banner, route diagram, stop timeline, cost breakdown | [TripResultView.tsx](file:///c:/Users/USER/GoAround-final/src/components/TripResultView.tsx) |
| **Google Maps Directions** | 🟢 Completed & Working | "Open in Google Maps ↗" link on every stop card | [TripResultView.tsx](file:///c:/Users/USER/GoAround-final/src/components/TripResultView.tsx#L34-L44) |
| **Guest Trip Persistence** | 🟢 Completed & Working | LocalStorage save/remove/view drawer | [SavedTripsDrawer.tsx](file:///c:/Users/USER/GoAround-final/src/components/SavedTripsDrawer.tsx) |
| **GeoJSON Data Processor** | 🟢 Completed & Working | Deduplicated 175 OSM features &rarr; 164 unique Pune places | [import-geojson.ts](file:///c:/Users/USER/GoAround-final/scripts/import-geojson.ts) |
| **Database Schema & Seed** | 🟡 Needs Execution | SQL migration and 164-place seed file generated | [20261019000000_create_destinations.sql](file:///c:/Users/USER/GoAround-final/supabase/migrations/20261019000000_create_destinations.sql), [seed.sql](file:///c:/Users/USER/GoAround-final/supabase/seed.sql) |
| **Live Supabase Queries** | 🟡 Needs Testing | Code handles Supabase queries, but remote tables are unpopulated | [supabase.ts](file:///c:/Users/USER/GoAround-final/src/services/supabase.ts) |
| **Planner Engine Algorithm** | 🟡 Partially Implemented | Road distance math works; needs route circuit ordering & departure transit | [plannerEngine.ts](file:///c:/Users/USER/GoAround-final/src/services/plannerEngine.ts) |
| **Popular Destinations Grid** | 🟡 Partially Implemented | Hardcoded to top 9 spots; lacks search, category filter, or pagination | [PopularDestinationsGrid.tsx](file:///c:/Users/USER/GoAround-final/src/components/PopularDestinationsGrid.tsx) |
| **URL Routing & Deep Links** | 🔴 Not Implemented | Tab state resets on browser refresh; share link lacks trip data | [App.tsx](file:///c:/Users/USER/GoAround-final/src/App.tsx#L17) |
| **User Accounts / Auth** | 🔴 Not Implemented | No login or signup flow (guest-only at present) | — |
| **Cloud Trip Sync** | 🔴 Not Implemented | SQL table exists, but frontend only saves to browser localStorage | [supabase.ts](file:///c:/Users/USER/GoAround-final/src/services/supabase.ts#L110) |
| **Interactive Map (Tiles)** | 🔴 Not Implemented | Route map is a CSS schematic bar; no live Leaflet/Mapbox map | [TripResultView.tsx](file:///c:/Users/USER/GoAround-final/src/components/TripResultView.tsx#L240) |

---

## 🔍 In-Depth Subsystem Breakdown

### 1. Frontend Architecture
* **Stack**: React 19, Vite 8, TypeScript 7, Tailwind CSS v4.
* **State Management**: Simple React hooks in [App.tsx](file:///c:/Users/USER/GoAround-final/src/App.tsx).
* **Current Limitation**: No router (`react-router-dom`). Refreshing the browser while viewing an itinerary resets the view back to the homepage.

### 2. Supabase & Database State
* **Credentials**: [`.env`](file:///c:/Users/USER/GoAround-final/.env) contains valid Supabase URL and Anon Key.
* **Client**: Configured in [`src/services/supabase.ts`](file:///c:/Users/USER/GoAround-final/src/services/supabase.ts) with zero silent-mock fallback: if Supabase is offline or empty, it shows a clear status banner.
* **Current Blocker**: The database tables (`destinations`, `saved_trips`) have not yet been migrated into the live Supabase project.

### 3. Destination Data Reality
* **Source GeoJSON**: 175 OpenStreetMap points in Pune & Western Ghats.
* **Cleaned Dataset**: 164 unique places (11 duplicates merged) in [`supabase/seed.sql`](file:///c:/Users/USER/GoAround-final/supabase/seed.sql).
* **Category Breakdown**:
  * 🏛️ Heritage: 107
  * 🌄 Scenic Views: 39
  * 🧗 Adventure: 9
  * 🌿 Nature: 9
* **Data Transparency**: OpenStreetMap tags lack entry ticket costs, photos, and exact visit durations. GoAround handles this honestly by flagging unverified gate fees with amber warning boxes instead of inventing fake prices.

### 4. Trip Planner Algorithm
* **Engine**: Located in [`src/services/plannerEngine.ts`](file:///c:/Users/USER/GoAround-final/src/services/plannerEngine.ts).
* **Distance**: Computes straight-line Haversine multiplied by a 1.35x road tortuosity factor.
* **Identified Issues**:
  1. **Missing Departure Transit**: Schedule starts at 8:00 AM at Stop 1 without accounting for the drive from the starting hub.
  2. **Circuit Sequencing**: When places lack `bestTimeOfDay` (true for all 164 GeoJSON spots), stops are sorted in arbitrary pool order rather than the shortest driving circuit.
  3. **Hardcoded ID**: The trekking exclusion checks for `dest.id === 'sinhagad_fort'` rather than matching OSM source IDs.

### 5. Google Maps Directions
* **Location**: Stop cards in [`TripResultView.tsx`](file:///c:/Users/USER/GoAround-final/src/components/TripResultView.tsx#L34-L44).
* **Behavior**: Clicking "Open in Google Maps ↗" opens the precise coordinate pin (`destination=lat,lng`) in a new tab without requiring a paid Maps API key. The starting point is left to the user's GPS.

### 6. Authentication & Storage
* **Guest Outings**: Stored in `localStorage` under `goaround_saved_itineraries`. Works seamlessly offline.
* **Auth & Cloud Sync**: Not implemented. Users cannot log in or access saved trips across different devices.

---

## 🐛 Key Bugs & Bottlenecks to Fix

1. **`scripts/import-geojson.ts`**: Missing `dotenv/config`, preventing the CLI tool from reading `.env` credentials automatically.
2. **Trip Schedule Timeline**: The departure travel time from Starting Hub to Stop 1 is missing from the clock.
3. **Trip Sharing Link**: The "Share" button copies `window.location.href` without trip data or query params.
4. **Heavy Static Assets**: 8 sample JPEGs in `src/assets/images/` total ~7.8 MB uncompressed.

---

## 🎯 The Five Next Development Steps

1. **Populate Supabase Database**:
   Add `dotenv/config` to [`scripts/import-geojson.ts`](file:///c:/Users/USER/GoAround-final/scripts/import-geojson.ts) and run [`supabase/migrations/20261019000000_create_destinations.sql`](file:///c:/Users/USER/GoAround-final/supabase/migrations/20261019000000_create_destinations.sql) and [`supabase/seed.sql`](file:///c:/Users/USER/GoAround-final/supabase/seed.sql) in the Supabase Dashboard.
2. **Optimize Planner Engine Scheduling & Route Pathing**:
   Add initial departure transit time and nearest-neighbor circuit sequencing in [`src/services/plannerEngine.ts`](file:///c:/Users/USER/GoAround-final/src/services/plannerEngine.ts).
3. **URL Routing & Shareable Trip State**:
   Enable query string or route state so users can reload the page or share links without losing their itinerary.
4. **Interactive Map & Stop Swapping**:
   Add a real interactive map view (Leaflet/MapLibre) and a "Replace Stop" button on stop cards.
5. **Supabase User Accounts & Multi-Device Sync**:
   Add login/signup via Supabase Auth and sync local itineraries to `public.saved_trips`.
