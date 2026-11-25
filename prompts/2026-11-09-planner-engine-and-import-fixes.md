# Planner Engine Schedule, Route Optimization & GeoJSON Import Fixes

- **Date**: 2026-11-09
- **Feature / Task**: Fix schedule timeline starting transit, nearest-neighbor circuit sequencing, OSM ID matching, and dotenv loading in import script
- **Status**: in progress
- **Related Prompts**:
  - [2026-10-19-destination-data-integration.md](./2026-10-19-destination-data-integration.md)
  - [2026-10-20-google-maps-directions.md](./2026-10-20-google-maps-directions.md)

---

## Original Prompt

```text
Continue
```

*(Following the technical status report: address the primary algorithmic and script bugs identified in the planner engine, GeoJSON importer, and Vite configuration).*

---

## Implementation Plan

1. **GeoJSON Importer (`scripts/import-geojson.ts`)**:
   - Add `dotenv/config` so `.env` is loaded automatically during `npm run import:geojson`.
   - Provide clear instructions regarding `SUPABASE_SERVICE_ROLE_KEY` requirements for client upserts vs. running `supabase/seed.sql` in the Supabase SQL editor.

2. **Planner Engine (`src/services/plannerEngine.ts`)**:
   - **Initial Departure Transit**: Add travel time from the selected starting hub to Stop 1 into the chronological timeline so departure time is realistic.
   - **Nearest-Neighbor Sequencing**: Implement geographic proximity ordering for destination candidates where `bestTimeOfDay` is null/unspecified, preventing zig-zag routes across the region.
   - **OSM ID Matching**: Update filtering (e.g. `thingsToAvoid: 'strenuous_trekking'`) to check `dest.sourceOsmId`, tags, and subcategory instead of hardcoded mock ID `sinhagad_fort`.

3. **Vite Configuration (`vite.config.ts`)**:
   - Replace deprecated `__dirname` with `import.meta.dirname` to eliminate the Vite build warning.

---

## Changed / Created Files

- `scripts/import-geojson.ts`
- `src/services/plannerEngine.ts`
- `vite.config.ts`
- `prompts/2026-11-09-planner-engine-and-import-fixes.md`

---

## Verification & Notes

- Run `npm run lint` (`tsc --noEmit`) to verify zero type regressions.
- Run `npm run build` to verify clean bundle generation without warnings.
- Run `npm run import:geojson -- --dry-run` to verify dotenv and GeoJSON pipeline.

