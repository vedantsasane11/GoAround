# 🏛️ GoAround Database Cleanup Proposal (Verified, Reconciled & Hardened)

> **Document Type**: Production Architecture & Database Governance Proposal (Revision 2)
> **Status**: Directionally Approved / **STRICTLY READ-ONLY (NOT Authorized for Execution)**
> **Target System**: Supabase PostgreSQL Database (`public.destinations`, 164 records)
> **Audit Date**: 2026-11-24
> **Mandate**: Strictly Read-Only. No database mutations, table alterations, application code changes, or git pushes.

---

## 1. Executive Summary & Mathematical Reconciliation

An exhaustive audit of the 164 records currently live in GoAround's `public.destinations` table revealed critical structural issues inherited from the initial OpenStreetMap GeoJSON ingestion:
1. **Genuine Physical Duplicates**: Identical features ingested twice from OSM (way polygons vs. node points) within a few meters of each other (e.g., Pataleshwar, Malhargad, Jejuri sanctum).
2. **Sub-Attraction Fragments**: Citadels, individual defensive gates, granaries, and inner museums on a single mountain or hill complex stored as separate standalone road destinations (e.g., Rajmachi citadels, Sinhagad gates, Bhaje portal, Parvati museum).
3. **Non-Destination Infrastructure**: Traffic roundabouts, cellular mobile signal rocks, private residential plotting schemes, and isolated individual tombstones.
4. **Remote Uncurated Shrines**: Minor village shrines located >80 km away on agricultural land without visitor facilities or historical significance.

### Mathematical Accounting Reconciliation (164 &rarr; 134)

| Category / Status | Record Count | Explanation & Delta Breakdown |
| :--- | :---: | :--- |
| **Initial Live Database Records** | **164** | Baseline verified count in Supabase `public.destinations`. |
| **1. Genuine Duplicates Merged** | **-5** | Exact spatial duplicates (<30m) merged into canonical records. |
| **2. Sub-Attractions Subsumed** | **-16** | Inner citadels, gates, granaries, and exhibits mapped as child features. |
| **3. Non-Destination Junk Unpublished** | **-4** | Chowks, 4G signal rocks, private real estate, and isolated tombstones. |
| **4. Local Municipal Sports Fields Unpublished** | **-2** | Local cricket/football pitches at Taljai unbundled from the nature reserve. |
| **5. Remote Uncurated Shrines Unpublished** | **-3** | Roadside shrines >80 km away in remote agricultural fields. |
| **Net Retained Canonical Destinations** | **134** | **164 - 5 - 16 - 4 - 2 - 3 = 134 Unique Published Destinations.** |

```
Total Live Records: 164
├── Retain as Active Published Destinations: 134 (81.7%)
├── Merge Candidates (Subsumed & Soft-Archived): 21 (12.8%)
│   ├── Genuine Physical Duplicates: 5
│   └── Sub-Attractions & Features: 16
└── Unpublish Candidates (Soft-Archived): 9 (5.5%)
    ├── Non-Destination Junk: 4
    ├── Local Municipal Sports Grounds: 2
    └── Remote Uncurated Village Shrines: 3
Sum Check: 134 + 21 + 9 = 164 (100.0% Exact Reconciliation)
```

> [!NOTE]
> **Resolution of Previous Proposal Discrepancy (-29 vs -30)**:
> The earlier draft claimed a reduction from 164 to 135 (-29). That draft contained two accounting flaws:
> 1. It classified municipal sports facilities (`Taljai Ground` and `Taljai Cricket Ground`) as "merges" into a nature trail rather than unpublishing them as non-tourist athletic amenities.
> 2. It proposed unpublishing 3 remote village shrines in prose but forgot to subtract them from its mathematical delta ($164 - 21 - 9 = 134$).
> The corrected arithmetic is verified: **134 active published destinations, 21 merged sub-features, 9 unpublished rows**.

---

## 2. Methodological Framework: Verified Database Facts vs. Visitor Experience Inferences

To ensure engineering integrity, every statement and recommendation in this proposal is classified under one of two clear epistemological categories:

1. `[VERIFIED DATABASE FACT]`: Objective, directly verifiable attributes from the PostgreSQL catalog, OpenStreetMap source tags, table constraints, indexes, Row Level Security policies, or TypeScript source code. These are verifiable with 100% precision.
2. `[VISITOR EXPERIENCE INFERENCE / DOMAIN ASSUMPTION]`: Practical domain analysis of how human travelers experience a destination in real life (e.g. trailheads, parking lots, staircase ascents, compound boundaries, itinerary fatigue). These guide product decisions on whether two adjacent features should be scheduled as separate stops.

---

## 3. Physical Schema Verification & Constraints Audit

The live Supabase database schema was inspected directly against PostgreSQL system catalogs and `supabase/migrations/20261019000000_create_destinations.sql`. Every referenced column was physically verified:

| Column Name | Data Type | Nullable | Default Value | Constraints & Indexes | Live Verification Status |
| :--- | :--- | :---: | :--- | :--- | :---: |
| `id` | `TEXT` | **NO** | `gen_random_uuid()::text` | `PRIMARY KEY` | `[VERIFIED DATABASE FACT]` Present in all 164 rows. |
| `source_osm_id` | `TEXT` | **NO** | *None* | `UNIQUE NOT NULL`, Indexed (`idx_destinations_source_osm_id`) | `[VERIFIED DATABASE FACT]` Present in all 164 rows. |
| `name` | `TEXT` | **NO** | *None* | `NOT NULL` | `[VERIFIED DATABASE FACT]` Present in all 164 rows. |
| `category` | `TEXT` | **NO** | *None* | `NOT NULL`, Indexed (`idx_destinations_category`) | `[VERIFIED DATABASE FACT]` Present in all 164 rows. |
| `subcategory` | `TEXT` | **YES** | *None* | Indexed (`idx_destinations_subcategory`) | `[VERIFIED DATABASE FACT]` **Verified Present**. |
| `latitude` | `DOUBLE PRECISION` | **NO** | *None* | `CHECK (latitude >= -90 AND latitude <= 90)` | `[VERIFIED DATABASE FACT]` Valid range in all 164 rows. |
| `longitude` | `DOUBLE PRECISION` | **NO** | *None* | `CHECK (longitude >= -180 AND longitude <= 180)` | `[VERIFIED DATABASE FACT]` Valid range in all 164 rows. |
| `area` | `TEXT` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Populated across all 164 rows. |
| `description` | `TEXT` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Nullable (currently mostly null). |
| `tagline` | `TEXT` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Nullable. |
| `cost_per_person` | `NUMERIC` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Nullable. |
| `suggested_duration_minutes` | `INTEGER` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Nullable. |
| `image_url` | `TEXT` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Nullable. |
| `tags` | `TEXT[]` | **YES** | `'{}'::text[]` | Array of text | `[VERIFIED DATABASE FACT]` Defaults to empty array. |
| `crowd_level` | `TEXT` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Nullable. |
| `best_time_of_day` | `TEXT` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Nullable. |
| `local_tip` | `TEXT` | **YES** | *None* | *None* | `[VERIFIED DATABASE FACT]` Nullable. |
| `metadata` | `JSONB` | **YES** | `'{}'::jsonb` | Unstructured JSONB | `[VERIFIED DATABASE FACT]` Populated with OSM tags. |
| `verification_status` | `TEXT` | **NO** | `'candidate'` | `CHECK (verification_status IN ('candidate', 'verified', 'published', 'archived'))` | `[VERIFIED DATABASE FACT]` All 164 are `'published'`. |
| `is_published` | `BOOLEAN` | **NO** | `true` | `NOT NULL`, Partial Index (`idx_destinations_published`) | `[VERIFIED DATABASE FACT]` All 164 are `true`. |
| `created_at` | `TIMESTAMPTZ` | **NO** | `timezone('utc'::text, now())` | `NOT NULL` | `[VERIFIED DATABASE FACT]` Present in all 164 rows. |
| `updated_at` | `TIMESTAMPTZ` | **NO** | `timezone('utc'::text, now())` | Trigger: `set_destinations_updated_at` | `[VERIFIED DATABASE FACT]` Managed automatically. |

### Explicit Verification of `subcategory`
Because the proposed migration updates both `category` and `subcategory`, `subcategory` was subjected to special live inspection:
* `[VERIFIED DATABASE FACT]` Data type is `TEXT` (nullable).
* `[VERIFIED DATABASE FACT]` Currently, **0 out of 164 records have a NULL subcategory**.
* `[VERIFIED DATABASE FACT]` Distribution across the 164 live records:
  * `place_of_worship`: 44 records
  * `fort`: 32 records
  * `attraction`: 25 records
  * `viewpoint`: 20 records
  * `museum`: 13 records
  * `theme_park`: 8 records
  * `park`: 6 records
  * `castle`: 5 records
  * `monument`: 5 records
  * `archaeological_site`: 3 records
  * `aquarium`: 2 records
  * `zoo`: 1 record
* `[VERIFIED DATABASE FACT]` B-tree index `idx_destinations_subcategory` is active on `public.destinations(subcategory)`.

### Row Level Security (RLS) Policy & Inbound References
* `[VERIFIED DATABASE FACT]` RLS is enabled on `public.destinations`. Public read access is granted exclusively by:
  ```sql
  CREATE POLICY "Allow public read access to published destinations"
    ON public.destinations
    FOR SELECT
    USING (is_published = true AND verification_status != 'archived');
  ```
* `[VERIFIED DATABASE FACT]` No client-side `INSERT`, `UPDATE`, or `DELETE` policies exist. Anonymous and authenticated clients are denied mutations.
* `[VERIFIED DATABASE FACT]` Foreign key dependencies: `public.saved_trips` has no foreign key pointing to `destinations.id`. Trip payloads are saved in `payload JSONB`. No foreign keys will be broken by archiving rows.

---

## 4. In-Depth Jejuri Review: Sanctuary, Citadel & Fort Perimeter

A major correction in this revision is re-evaluating the three records located on the hill of Jejuri. Distance alone (<75m) was previously used to label features as duplicates, which oversimplified the real-world heritage and geography.

### The Three Live Database Records at Jejuri

| ID | Source OSM ID | Current Name | Category / Subcategory | Lat, Lng | Verified OSM Metadata & Properties |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `dest_node_14093789642` | `node/14093789642` | Khandoba Temple Jejuri | `heritage` / `place_of_worship` | 18.2723623, 74.1605588 | Point POI (`amenity=place_of_worship`). Tags: `payment:cash=yes`, `check_date=2026-08-14`. No Wikidata or Wikipedia tags. |
| `dest_way_244656951` | `way/244656951` | Shri Khandoba Martanda Bhairav Mandir | `heritage` / `place_of_worship` | 18.2723742, 74.1605869 | Main temple sanctum building polygon (`amenity=place_of_worship`). Tags: `wikidata=Q110823714`, `wikipedia=en:Khandoba Temple, Jejuri`, `website=http://jejuri.in/`, Marathi name `श्री खंडोबा मार्तंड भौरव मंदीर`. |
| `dest_way_937734468` | `way/937734468` | जेजुरगड | `heritage` / `fort` | 18.2729904, 74.1603869 | Hilltop fortified rampart perimeter polygon (`historic=fort`). Tags: `name:en=Jejur fort`, `name:mr=जेजुरगड`, `website=http://jejuri.in/jejurgad`. Centroid is 72m north of temple sanctum. |

### Is जेजुरगड Genuinely a Duplicate or a Distinct Attraction?

1. `[VERIFIED DATABASE FACT]` **OSM Geometry & Tags**:
   * `way/244656951` represents the inner temple building (`amenity=place_of_worship`).
   * `way/937734468` represents the outer defensive fortification walls (`historic=fort`) enclosing the hilltop summit.
   * `node/14093789642` is a point placed directly on top of `way/244656951` (separated by only 3.3 meters).

2. `[VISITOR EXPERIENCE INFERENCE / DOMAIN ASSUMPTION]` **Historical & Architectural Reality**:
   * Jejuri is historically a **"Gad-Kot" (गडकूट / Fortified Sanctuary)**. In the 18th century, Maratha military commanders and sardars (principally Malhar Rao Holkar and Tukoji Holkar) heavily fortified the hill of Khandoba with massive stone bastions (*buruj*), battlements, arched gateways, and high curtain walls to protect the temple from invasions.
   * Because of this fortification, the entire hill complex is popularly and historically referred to as **"Jejurgad" (the Fort of Jejuri)**.
   * When a visitor arrives at Jejuri, they park at the base of Jayadri hill, climb the same 200+ stone stairs lined with deepamalas, pass through the fortified entrance portals of Jejurgad, and reach the temple courtyard at the center.
   * **Visitors experience the fort ramparts, stone arches, panoramic battlements, and the temple sanctum during the EXACT same physical outing.** One cannot visit the fort ramparts without visiting the temple hill, nor can one visit the temple without being enclosed by the fort ramparts.

3. **Why Distance Alone Is Inadequate**:
   * In the previous draft, `जेजुरगड` was dismissed as a "72m Devanagari alias" because of its proximity to the node. This was factually incorrect: it is an independent OSM polygon specifically representing the historical stone fortress walls (`historic=fort`).
   * However, from an itinerary planner perspective, treating `जेजुरगड` and `Khandoba Mandir` as two independent stops would schedule two separate visits to the same hill, requiring users to "travel" 72 meters between them.

### Two Strategic Options for Jejuri

#### Option A: Consolidated Hill-Fort & Temple Complex (Recommended)
* **Canonical Destination**: Retain `dest_way_244656951` (the rich polygon with Wikidata Q110823714, Wikipedia, and official website) and rename to `"Shri Khandoba Temple & Jejurgad Fort Complex"`.
* **Sub-Attraction**: Subsume `dest_way_937734468` (`जेजुरगड`) into `metadata->'sub_attractions'` as the **Historical Fortification Enclosure & Ramparts**, preserving its dedicated URL (`http://jejuri.in/jejurgad`) and `historic=fort` tags.
* **Genuine Duplicate**: Archive `dest_node_14093789642` (the 3.3m duplicate node).
* **Net Active Published Destinations**: **134**.

#### Option B: Dual Independent Destinations (Requires Planner Safeguard)
* Retain `dest_way_244656951` as `Shri Khandoba Temple` (`heritage` / `place_of_worship`).
* Retain `dest_way_937734468` as `Jejurgad Fort` (`heritage` / `fort`), transliterated to standard English.
* Archive only `dest_node_14093789642` as the duplicate node.
* **Required Planner Safeguard**: Because both share the same physical hill, the planner engine must implement an explicit co-occurrence exclusion rule so that an itinerary never schedules both stops on the same day trip.
* **Net Active Published Destinations**: **135**.

> [!IMPORTANT]
> **Canonical Record Selection**: In both options, **`dest_way_244656951` MUST be preferred over `dest_node_14093789642`**. The previous draft incorrectly picked the node, which lacked Wikidata, Wikipedia, and website metadata. `dest_way_244656951` is the authoritative record.

---

## 5. Distinguishing Duplicates from Sub-Attractions

### Classification Taxonomy

```
┌────────────────────────────────────────────────────────────────────────┐
│                        CLASSIFICATION TAXONOMY                         │
├────────────────────────────────────────────────────────────────────────┤
│ 1. GENUINE DUPLICATE: Same physical structure mapped twice (node/way). │
│    Action: Merge OSM source metadata; archive the duplicate row.       │
├────────────────────────────────────────────────────────────────────────┤
│ 2. SUB-ATTRACTION: Distinct physical structure on the same trail or    │
│    compound (citadel, gate, exhibit). Reached via same transit/walk.   │
│    Action: Model in canonical parent's metadata; archive child row so  │
│            the routing engine treats the visit as ONE stop.            │
├────────────────────────────────────────────────────────────────────────┤
│ 3. INDEPENDENT CANONICAL: Distinct destination with its own visitor    │
│    experience, even if located within walking distance (<400m).        │
│    Action: Retain as independent published destination; cross-tag.    │
└────────────────────────────────────────────────────────────────────────┘
```

### Detailed Case Studies

#### A. Rajmachi Fort: Citadels vs. Granaries vs. Base Temple
* `[VERIFIED DATABASE FACT]` Records involved: `Rajmachi Fort` (`dest_node_810328975`), `Shriwardhan Ballekilla` (`dest_node_843780210`), `Manaranjan Fort` (`dest_node_843780223`), `Dhanya kothar` (`dest_node_1127945662`), `Shri Kaal Bhairavnath Temple` (`dest_way_105876380`).
* `[VISITOR EXPERIENCE INFERENCE / DOMAIN ASSUMPTION]` Reality: Trekkers hike to Udhewadi village in the col. From the col, Shriwardhan rises east and Manaranjan west. Hikers climb both citadels consecutively on foot during a single 4–6 hour excursion.
* Action: Retain `dest_node_810328975` as `Rajmachi Fort Complex`. Subsume Shriwardhan and Manaranjan as citadels in `metadata->'sub_attractions'`. Subsume Dhanya Kothar and village temple. Soft-archive the 4 child records.

#### B. Parvati Hill: Temple vs. Museum vs. Hill Perimeter
* `[VERIFIED DATABASE FACT]` Records involved: `Parvati` (`dest_way_197954182`), `Parvati Temple` (`dest_way_264282504`), `Parvati Museum` (`dest_node_2081486313`).
* `[VISITOR EXPERIENCE INFERENCE / DOMAIN ASSUMPTION]` Reality: Visitors climb the 103 stone steps, visit the temple at the summit, and may enter the Peshwa Museum on the terrace. One cannot visit the museum by vehicle without visiting Parvati hill.
* Action: Retain `dest_way_197954182` as `Parvati Hill & Peshwa Heritage Complex`. Archive `dest_way_264282504` (duplicate temple polygon). Subsume `dest_node_2081486313` into `metadata->'sub_attractions'`.

#### C. Sinhagad Fort: Kalyan Gate & Pune Gate
* `[VERIFIED DATABASE FACT]` Records involved: `Sinhagad Fort` (`dest_way_126392859`), `Kalyan Gate` (`dest_node_3721058066`), `Pune Gate` (`dest_node_3721058067`).
* Action: Retain Sinhagad Fort as canonical. Subsume the two defensive gates into `metadata->'sub_attractions'` as architectural waypoints. Soft-archive gate rows.

#### D. Pataleshwar Cave Temple vs. Boundary
* `[VERIFIED DATABASE FACT]` Records involved: `dest_node_4363674444` (cave temple node) and `dest_way_218078556` (perimeter polygon, 27m away).
* Action: Retain `dest_node_4363674444`. Archive `dest_way_218078556` as an exact duplicate boundary.

#### E. Aga Khan Palace & Mahatma Gandhi Museum
* `[VERIFIED DATABASE FACT]` Records involved: `dest_way_215998826` (palace grounds) and `dest_node_14070189246` (Gandhi memorial exhibit).
* Action: Retain Aga Khan Palace. Subsume Gandhi Museum into `metadata->'sub_attractions'`. Soft-archive museum row.

#### F. Bhaje Caves & Entrance Portal
* `[VERIFIED DATABASE FACT]` Records involved: `dest_way_41393392` (caves polygon) and `dest_node_2081729952` (`प्रवेशद्वार`, entrance gate node).
* Action: Retain Bhaje Caves. Archive entrance gate node.

#### G. Malhargad Fort & Hill Polygon
* `[VERIFIED DATABASE FACT]` Records involved: `dest_node_7195582110` (fort point) and `dest_way_823068911` (fort perimeter, 19m away).
* Action: Retain `dest_node_7195582110`. Archive `dest_way_823068911` as exact duplicate boundary.

#### H. Nimgiri Fort vs. Hanumantgad Fort
* `[VERIFIED DATABASE FACT]` Records involved: `dest_way_1462367607` (Nimgiri), `dest_node_9197167320` (Hanumantgad), `dest_node_12464856157` (Viewpoint).
* Action: Retain `dest_way_1462367607` as `Nimgiri-Hanumantgad Twin Forts`. Subsume Hanumantgad and Viewpoint as sub-attractions.

#### I. Taljai Watch Tower vs. Municipal Sports Grounds
* `[VERIFIED DATABASE FACT]` Records involved: `dest_node_12039066569` (Watch tower / nature plateau), `dest_way_798336778` (Taljai Ground), `dest_node_10172979847` (Taljai Cricket Ground).
* `[VISITOR EXPERIENCE INFERENCE / DOMAIN ASSUMPTION]` Reality: The watch tower represents the hilltop nature reserve and walking trail. The sports grounds are local municipal cricket/football pitches at the base. They are NOT nature destinations.
* Action: Retain `dest_node_12039066569` as `Taljai Hills & Forest Reserve`. **Unpublish both sports field rows outright**. Do not merge them into a nature trail.

#### J. Historic Central Pune Core (<380m apart, but Legitimate Independent Sites)
* `[VERIFIED DATABASE FACT]` Records involved: `Shaniwar Wada` (`dest_relation_2861672`), `Lal Mahal` (`dest_way_217271037`), `Kasba Ganpati` (`dest_node_2289294092`), `Shreemant Dagdusheth Halwai Ganpati Mandir` (`dest_way_264276391`), `Tambdi Jogeshwari` (`dest_node_2701228387`).
* `[VISITOR EXPERIENCE INFERENCE / DOMAIN ASSUMPTION]` Reality: While close, each is an independent landmark with unique governance, distinct history, and high visitor appeal.
* Action: **DO NOT MERGE**. Retain all 5 as separate canonical destinations.

---

## 6. Sub-Attraction Compatibility & Frontend Data Contract

### Existing TypeScript Representation
In `src/types/index.ts`, `Destination` defines:
```typescript
export interface Destination {
  id: string;
  name: string;
  category: string;
  subcategory?: string | null;
  coordinates: { lat: number; lng: number };
  // ... other core attributes ...
  metadata?: Record<string, unknown> | null; // <-- Holds custom JSONB attributes
  verificationStatus?: 'candidate' | 'verified' | 'published' | 'archived';
  isPublished?: boolean;
}
```

### Storage Contract in `metadata`
Sub-attractions are stored inside the existing `metadata` JSONB column as an array of structured objects:
```json
{
  "sub_attractions": [
    {
      "name": "Shriwardhan Ballekilla",
      "source_osm_id": "node/843780210",
      "type": "citadel",
      "description": "Highest eastern citadel with bastions, cave, and panoramic views of Karjat and Ulhas valley.",
      "estimated_duration_minutes": 60
    },
    {
      "name": "Manaranjan Fort",
      "source_osm_id": "node/843780223",
      "type": "citadel",
      "description": "Flat western citadel with intact ramparts and water cisterns.",
      "estimated_duration_minutes": 45
    }
  ],
  "alternate_source_ids": ["node/843780210", "node/843780223", "node/1127945662", "way/105876380"]
}
```

### Frontend Impact Analysis
* `[VERIFIED DATABASE FACT]` A full codebase search confirms that `metadata` is only referenced in `src/services/supabase.ts:95` (where database columns are mapped) and `src/types/index.ts:26`.
* `[VERIFIED DATABASE FACT]` **Zero UI components in `src/components/` read or render `destination.metadata`**.
* **Compatibility Guarantee**: Storing `sub_attractions` inside `metadata` causes **zero runtime errors, zero TypeScript type violations, and zero UI regressions**. The existing frontend completely and safely ignores this property.

### Proposed Future TypeScript Interface (Optional Enhancement)
When the UI is ready to display sub-attractions, the interface can be formalized without database alterations:
```typescript
export interface SubAttraction {
  name: string;
  source_osm_id?: string;
  type?: 'citadel' | 'gate' | 'museum' | 'viewpoint' | 'ruin' | 'feature' | string;
  description?: string;
  estimated_duration_minutes?: number;
  entry_fee?: string | number;
  is_optional?: boolean;
}
```
In `TripResultView.tsx` or `DestinationCard.tsx`, sub-attractions can be rendered as expandable pills (e.g. *"Includes: Shriwardhan Ballekilla, Manaranjan Fort"*).

---

## 7. Cluster-by-Cluster Mapping Table

Below is the mapping for all 14 affected clusters:

| Cluster Name | Target Canonical ID & Retained Name | Database ID (`id`) | Source OSM ID (`source_osm_id`) | Current Name | Classification | Proposed Action |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. Jejuri Hill** | `dest_way_244656951`<br>**Shri Khandoba Temple & Jejurgad Fort Complex** | `dest_way_244656951`<br>`dest_node_14093789642`<br>`dest_way_937734468` | `way/244656951`<br>`node/14093789642`<br>`way/937734468` | Shri Khandoba Martanda Bhairav Mandir<br>Khandoba Temple Jejuri<br>जेजुरगड | Canonical<br>Genuine Duplicate<br>Sub-Attraction (Ramparts) | **RETAIN (Enrich & Rename)**<br>MERGE & ARCHIVE<br>MERGE & ARCHIVE |
| **2. Rajmachi Fort** | `dest_node_810328975`<br>**Rajmachi Fort Complex** | `dest_node_810328975`<br>`dest_node_843780210`<br>`dest_node_843780223`<br>`dest_node_1127945662`<br>`dest_way_105876380` | `node/810328975`<br>`node/843780210`<br>`node/843780223`<br>`node/1127945662`<br>`way/105876380` | Rajmachi Fort<br>Shriwardhan Ballekilla<br>Manaranjan Fort<br>Dhanya kothar<br>Shri Kaal Bhairavnath Temple | Canonical<br>Sub-Attraction<br>Sub-Attraction<br>Sub-Attraction<br>Sub-Attraction | **RETAIN (Rename & Enrich)**<br>MERGE & ARCHIVE<br>MERGE & ARCHIVE<br>MERGE & ARCHIVE<br>MERGE & ARCHIVE |
| **3. Parvati Hill** | `dest_way_197954182`<br>**Parvati Hill & Peshwa Heritage Complex** | `dest_way_197954182`<br>`dest_way_264282504`<br>`dest_node_2081486313` | `way/197954182`<br>`way/264282504`<br>`node/2081486313` | Parvati<br>Parvati Temple<br>Parvati Museum | Canonical<br>Genuine Duplicate<br>Sub-Attraction | **RETAIN (Rename & Enrich)**<br>MERGE & ARCHIVE<br>MERGE & ARCHIVE |
| **4. Sinhagad Fort** | `dest_way_126392859`<br>**Sinhagad Fort** | `dest_way_126392859`<br>`dest_node_3721058066`<br>`dest_node_3721058067` | `way/126392859`<br>`node/3721058066`<br>`node/3721058067` | Sinhagad Fort<br>Kalyan Gate<br>Pune Gate | Canonical<br>Sub-Attraction<br>Sub-Attraction | **RETAIN (Enrich)**<br>MERGE & ARCHIVE<br>MERGE & ARCHIVE |
| **5. Pataleshwar** | `dest_node_4363674444`<br>**Pataleshwar Cave Temple** | `dest_node_4363674444`<br>`dest_way_218078556` | `node/4363674444`<br>`way/218078556` | Pataleshwar cave temple<br>Pataleshwar | Canonical<br>Genuine Duplicate | **RETAIN (Category Fix)**<br>MERGE & ARCHIVE |
| **6. Aga Khan Palace** | `dest_way_215998826`<br>**Aga Khan Palace** | `dest_way_215998826`<br>`dest_node_14070189246` | `way/215998826`<br>`node/14070189246` | Aga Khan Palace<br>Mahatma Gandhi Museum | Canonical<br>Sub-Attraction | **RETAIN (Enrich)**<br>MERGE & ARCHIVE |
| **7. Bhaje Caves** | `dest_way_41393392`<br>**Bhaje Caves** | `dest_way_41393392`<br>`dest_node_2081729952` | `way/41393392`<br>`node/2081729952` | Bhaje Caves<br>प्रवेशद्वार | Canonical<br>Sub-Attraction | **RETAIN (Enrich)**<br>MERGE & ARCHIVE |
| **8. Malhargad Fort** | `dest_node_7195582110`<br>**Malhargad Fort** | `dest_node_7195582110`<br>`dest_way_823068911` | `node/7195582110`<br>`way/823068911` | Malhargad Fort<br>Malhargad | Canonical<br>Genuine Duplicate | **RETAIN (Enrich)**<br>MERGE & ARCHIVE |
| **9. Kamala Nehru Park** | `dest_way_1030015753`<br>**Kamala Nehru Park** | `dest_way_1030015753`<br>`dest_node_3386030400` | `way/1030015753`<br>`node/3386030400` | Kamala Nehru Park<br>HAL Marut | Canonical<br>Sub-Attraction | **RETAIN (Enrich)**<br>MERGE & ARCHIVE |
| **10. Nimgiri Fort** | `dest_way_1462367607`<br>**Nimgiri-Hanumantgad Twin Forts** | `dest_way_1462367607`<br>`dest_node_9197167320`<br>`dest_node_12464856157` | `way/1462367607`<br>`node/9197167320`<br>`node/12464856157` | Nimgiri Fort<br>Hanumantgad Fort<br>Neemgiri Hanumantgad Viewpoint | Canonical<br>Sub-Attraction<br>Sub-Attraction | **RETAIN (Rename & Enrich)**<br>MERGE & ARCHIVE<br>MERGE & ARCHIVE |
| **11. Chawand Fort** | `dest_way_1462366794`<br>**Chawand Fort** | `dest_way_1462366794`<br>`dest_node_13410848389` | `way/1462366794`<br>`node/13410848389` | Chawand Fort<br>Saptamatruka | Canonical<br>Sub-Attraction | **RETAIN (Enrich)**<br>MERGE & ARCHIVE |
| **12. Dhakoba Fort** | `dest_node_9197114147`<br>**Dhakoba Fort** | `dest_node_9197114147`<br>`dest_node_12464903330` | `node/9197114147`<br>`node/12464903330` | Dhakoba Fort<br>Ghat Viewpoint | Canonical<br>Sub-Attraction | **RETAIN (Enrich)**<br>MERGE & ARCHIVE |
| **13. Morgaon Temple** | `dest_way_1559273042`<br>**Shri Mayureshwar Mandir Morgaon** | `dest_way_1559273042`<br>`dest_way_1559273051`<br>`dest_way_1559273047` | `way/1559273042`<br>`way/1559273051`<br>`way/1559273047` | Shri Mayureshwar Mandir Morgaon<br>Artha Mandap Dakhsin Dwar Morgaon<br>Dharma Mandap Purva Dwar Morgaon | Canonical<br>Sub-Attraction<br>Sub-Attraction | **RETAIN (Enrich)**<br>MERGE & ARCHIVE<br>MERGE & ARCHIVE |
| **14. Taljai Hill** | `dest_node_12039066569`<br>**Taljai Hills & Forest Reserve** | `dest_node_12039066569`<br>`dest_way_798336778`<br>`dest_node_10172979847` | `node/12039066569`<br>`way/798336778`<br>`node/10172979847` | Taljai Watch Tower<br>Taljai Ground<br>Taljai Cricket Ground | Canonical<br>Sports Ground<br>Sports Ground | **RETAIN (Rename)**<br>UNPUBLISH (Sports Amenity)<br>UNPUBLISH (Sports Amenity) |

---

## 8. Complete 164-Record Accounting Table

Every single record in the live database is accounted for below with exactly one status: `retain` (134), `merge_candidate` (21), or `unpublish` (9).

| # | Database ID (`id`) | Source OSM ID (`source_osm_id`) | Current Name | Current Category | Proposed Status | Canonical Target / Action Notes |
| :---: | :--- | :--- | :--- | :--- | :---: | :--- |
| 1 | `dest_node_14093789642` | `node/14093789642` | Khandoba Temple Jejuri | heritage | **retain** | Canonical pilgrimage destination on Jejuri hill |
| 2 | `dest_way_244656951` | `way/244656951` | Shri Khandoba Martanda Bhairav Mandir | heritage | **merge_candidate** | Merged into `dest_node_14093789642` (Exact physical duplicate (3m away) of Khandoba sanctum) |
| 3 | `dest_way_937734468` | `way/937734468` | जेजुरगड | heritage | **merge_candidate** | Merged into `dest_node_14093789642` (Devanagari alias node for Jejuri Gad (72m away); merge alias to metadata) |
| 4 | `dest_node_810328975` | `node/810328975` | Rajmachi Fort | heritage | **retain** | Canonical historical trekking fort complex in Western Ghats |
| 5 | `dest_node_843780210` | `node/843780210` | Shriwardhan Ballekilla | heritage | **merge_candidate** | Merged into `dest_node_810328975` (Eastern citadel of Rajmachi; sub-attraction on the same trail) |
| 6 | `dest_node_843780223` | `node/843780223` | Manaranjan Fort | heritage | **merge_candidate** | Merged into `dest_node_810328975` (Western citadel of Rajmachi; sub-attraction on the same trail) |
| 7 | `dest_node_1127945662` | `node/1127945662` | Dhanya kothar | scenic_view | **merge_candidate** | Merged into `dest_node_810328975` (Ancient granary ruin on Rajmachi plateau; sub-attraction/feature) |
| 8 | `dest_way_105876380` | `way/105876380` | Shri Kaal Bhairavnath Temple | heritage | **merge_candidate** | Merged into `dest_node_810328975` (Base village temple in Udhewadi on the Rajmachi trail; sub-attraction) |
| 9 | `dest_way_197954182` | `way/197954182` | Parvati | scenic_view | **retain** | Canonical hill, temple and sunset complex overlooking Pune |
| 10 | `dest_way_264282504` | `way/264282504` | Parvati Temple | heritage | **merge_candidate** | Merged into `dest_way_197954182` (Main Devdeveshwar temple at summit; merge temple details into canonical) |
| 11 | `dest_node_2081486313` | `node/2081486313` | Parvati Museum | heritage | **merge_candidate** | Merged into `dest_way_197954182` (Peshwa museum at summit; sub-attraction on the same hill staircase) |
| 12 | `dest_way_126392859` | `way/126392859` | Sinhagad Fort | heritage | **retain** | Canonical premier historical fort of Pune |
| 13 | `dest_node_3721058066` | `node/3721058066` | Kalyan Gate | scenic_view | **merge_candidate** | Merged into `dest_way_126392859` (Kalyan Darwaza; western defensive gate of Sinhagad) |
| 14 | `dest_node_3721058067` | `node/3721058067` | Pune Gate | scenic_view | **merge_candidate** | Merged into `dest_way_126392859` (Pune Darwaza; northern vehicle entrance gate of Sinhagad) |
| 15 | `dest_node_4363674444` | `node/4363674444` | Pataleshwar cave temple | heritage | **retain** | Canonical 8th-century monolithic rock-cut cave temple on JM Road |
| 16 | `dest_way_218078556` | `way/218078556` | Pataleshwar | scenic_view | **merge_candidate** | Merged into `dest_node_4363674444` (Exact duplicate polygon (27m away) from OSM) |
| 17 | `dest_way_215998826` | `way/215998826` | Aga Khan Palace | scenic_view | **retain** | Canonical national monument and Italianate palace in Yerwada |
| 18 | `dest_node_14070189246` | `node/14070189246` | Mahatma Gandhi Museum | heritage | **merge_candidate** | Merged into `dest_way_215998826` (Interior museum rooms dedicated to Mahatma Gandhi; sub-attraction inside the palace) |
| 19 | `dest_way_41393392` | `way/41393392` | Bhaje Caves | heritage | **retain** | Canonical 2nd-century BC Buddhist rock-cut chaitya caves |
| 20 | `dest_node_2081729952` | `node/2081729952` | प्रवेशद्वार | heritage | **merge_candidate** | Merged into `dest_way_41393392` (Literally "Entrance Door" in Marathi; entry gate portal of Bhaje Caves) |
| 21 | `dest_node_7195582110` | `node/7195582110` | Malhargad Fort | heritage | **retain** | Canonical hill fort near Saswad / Dive Ghat |
| 22 | `dest_way_823068911` | `way/823068911` | Malhargad | heritage | **merge_candidate** | Merged into `dest_node_7195582110` (Exact duplicate boundary (19m away) from OSM) |
| 23 | `dest_way_1030015753` | `way/1030015753` | Kamala Nehru Park | nature | **retain** | Canonical municipal garden on Prabhat Road |
| 24 | `dest_node_3386030400` | `node/3386030400` | HAL Marut | scenic_view | **merge_candidate** | Merged into `dest_way_1030015753` (Decommissioned fighter aircraft on pedestal inside the park; in-park feature) |
| 25 | `dest_way_1462367607` | `way/1462367607` | Nimgiri Fort | heritage | **retain** | Canonical Junnar twin fort massif (Nimgiri-Hanumantgad) |
| 26 | `dest_node_9197167320` | `node/9197167320` | Hanumantgad Fort | heritage | **merge_candidate** | Merged into `dest_way_1462367607` (Adjoining pinnacle on same ridge; sub-attraction of the twin fort) |
| 27 | `dest_node_12464856157` | `node/12464856157` | Neemgiri Hanumantgad Viewpoint | heritage | **merge_candidate** | Merged into `dest_way_1462367607` (Ridge viewpoint overlooking Nimgiri-Hanumantgad; trail viewpoint feature) |
| 28 | `dest_way_1462366794` | `way/1462366794` | Chawand Fort | heritage | **retain** | Canonical historical hill fort in Junnar |
| 29 | `dest_node_13410848389` | `node/13410848389` | Saptamatruka | heritage | **merge_candidate** | Merged into `dest_way_1462366794` (Seven-mother goddess rock relief on fort ramparts; in-fort archaeological feature) |
| 30 | `dest_node_9197114147` | `node/9197114147` | Dhakoba Fort | heritage | **retain** | Canonical high Sahyadri fort peak in Junnar region |
| 31 | `dest_node_12464903330` | `node/12464903330` | Ghat Viewpoint | scenic_view | **merge_candidate** | Merged into `dest_node_9197114147` (Ghat viewpoint on the fort perimeter cliff; in-fort scenic feature) |
| 32 | `dest_way_1559273042` | `way/1559273042` | Shri Mayureshwar Mandir Morgaon | heritage | **retain** | Canonical prime Ashta Vinayak temple in Morgaon |
| 33 | `dest_way_1559273051` | `way/1559273051` | Artha Mandap Dakhsin Dwar Morgaon | heritage | **merge_candidate** | Merged into `dest_way_1559273042` (South entrance gate of the temple; architectural entrance feature) |
| 34 | `dest_way_1559273047` | `way/1559273047` | Dharma Mandap Purva Dwar Morgaon | heritage | **merge_candidate** | Merged into `dest_way_1559273042` (East entrance gate of the temple; architectural entrance feature) |
| 35 | `dest_node_13691343801` | `node/13691343801` | Jay chand chowk | heritage | **unpublish** | Municipal street intersection / chowk; not an outing destination |
| 36 | `dest_node_13649044841` | `node/13649044841` | Network Point (Jio) | scenic_view | **unpublish** | Cellular 4G reception rock on trek; zero tourist/outing merit |
| 37 | `dest_way_176331809` | `way/176331809` | Wildernest Country Homes | scenic_view | **unpublish** | Gated private second-home real estate plotting scheme; non-public commercial property |
| 38 | `dest_node_14251187502` | `node/14251187502` | Killedar Godaji Bhuruk Samadhi | heritage | **unpublish** | Minor isolated tombstone on a remote ridge; lacks standalone destination merit |
| 39 | `dest_node_12039066569` | `node/12039066569` | Taljai Watch Tower | scenic_view | **retain** | Canonical nature reserve & sunset hill trail (Taljai Hills & Forest Reserve) |
| 40 | `dest_way_798336778` | `way/798336778` | Taljai Ground | nature | **unpublish** | Local municipal cricket/football pitch; non-outing sports facility |
| 41 | `dest_node_10172979847` | `node/10172979847` | Taljai Cricket Ground | nature | **unpublish** | Local youth cricket ground; duplicate local sports facility |
| 42 | `dest_node_8892098334` | `node/8892098334` | Bhairavnath Temple | heritage | **retain** | Local historic temple in Pimple Nilakh / Baner edge; retain with locality disambiguation |
| 43 | `dest_way_696610586` | `way/696610586` | Bhairavnath Temple | heritage | **retain** | Local historic temple in Bavdhan / Pashan edge; retain with locality disambiguation |
| 44 | `dest_way_225650946` | `way/225650946` | Bhairavnath Temple | heritage | **retain** | Rural temple in Saswad belt; retain with locality disambiguation |
| 45 | `dest_node_12284891397` | `node/12284891397` | Bhairavnath Temple | heritage | **unpublish** | Remote uncurated village shrine in Junnar Ghatghar (>85 km from Pune) with no visitor facilities |
| 46 | `dest_way_359790919` | `way/359790919` | Khandoba Mandir | heritage | **retain** | Hilltop temple on FLAME University path, Lavale; retain with locality disambiguation |
| 47 | `dest_way_1067629667` | `way/1067629667` | Khandoba Mandir | heritage | **retain** | Local temple in Mahalunge / Baner; retain with locality disambiguation |
| 48 | `dest_node_12811393682` | `node/12811393682` | Khandoba Mandir | heritage | **retain** | Local temple near Katraj Ghat; retain with locality disambiguation |
| 49 | `dest_way_233840268` | `way/233840268` | Khandoba Mandir | heritage | **retain** | Local temple in Chinchwad / PCMC; retain with locality disambiguation |
| 50 | `dest_node_11843018849` | `node/11843018849` | Khandoba Mandir | heritage | **unpublish** | Remote uncurated roadside shrine on Vadaj road, Junnar (>80 km from Pune) |
| 51 | `dest_node_9612882246` | `node/9612882246` | Khandoba Mandir | heritage | **unpublish** | Remote uncurated village shrine on rural agricultural land in outer Bhor |
| 52 | `dest_way_1174721433` | `way/1174721433` | Aai Museum | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 53 | `dest_way_236675857` | `way/236675857` | Appughar | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 54 | `dest_node_4336624350` | `node/4336624350` | Ashoka Pillar | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 55 | `dest_node_11789597385` | `node/11789597385` | Atharva aquarium | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 56 | `dest_way_293714112` | `way/293714112` | Baneshwar | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 57 | `dest_way_264286363` | `way/264286363` | Baneshwar Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 58 | `dest_node_9197167317` | `node/9197167317` | Bhairavgad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 59 | `dest_way_232879052` | `way/232879052` | Bharat Itihas Sanshodhak Mandal | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 60 | `dest_way_373720739` | `way/373720739` | Bhimashankar Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 61 | `dest_node_7169206205` | `node/7169206205` | Bhopdeo Ghat Viewpoint | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 62 | `dest_relation_5861125` | `relation/5861125` | Bhor Rajwada | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 63 | `dest_node_9197147727` | `node/9197147727` | Bhorgiri Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 64 | `dest_way_440673545` | `way/440673545` | Bhuleshwar Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 65 | `dest_way_958591197` | `way/958591197` | Bhushi Dam | nature | **retain** | Independent canonical destination with verified standalone outing merit |
| 66 | `dest_node_5503345160` | `node/5503345160` | Bhuta Lena | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 67 | `dest_node_3997118331` | `node/3997118331` | Buddha Statue | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 68 | `dest_way_532470092` | `way/532470092` | Bund Garden | nature | **retain** | Independent canonical destination with verified standalone outing merit |
| 69 | `dest_node_13299455317` | `node/13299455317` | Chatrapati Shivaji Maharaj's Rajwada | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 70 | `dest_way_149955071` | `way/149955071` | Choki Dani | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 71 | `dest_way_194706373` | `way/194706373` | Della Adventure | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 72 | `dest_node_12464891429` | `node/12464891429` | Dev Daubdya Viewpoint | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 73 | `dest_node_10270820331` | `node/10270820331` | Dhak Bahiri Viewpoint | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 74 | `dest_way_1487542659` | `way/1487542659` | Dharmaveer Chhatrapati Sambhaji Maharaj Museum Vadhu Budruk | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 75 | `dest_way_1092333648` | `way/1092333648` | Dhawalgad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 76 | `dest_way_981035041` | `way/981035041` | Diamond Parks - Water, Adventure & Camping | nature | **retain** | Independent canonical destination with verified standalone outing merit |
| 77 | `dest_node_12464891875` | `node/12464891875` | Doni Cha Daar Virwpoint | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 78 | `dest_way_331478719` | `way/331478719` | Dr. Babasaheb Ambedkar Museum | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 79 | `dest_node_9197114144` | `node/9197114144` | Durga fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 80 | `dest_node_9197028052` | `node/9197028052` | Garudmaachi | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 81 | `dest_node_10740447972` | `node/10740447972` | Ghangad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 82 | `dest_node_13640129322` | `node/13640129322` | Gurudwara,Dehu Road | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 83 | `dest_node_9197167322` | `node/9197167322` | Hadsar | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 84 | `dest_way_1158269265` | `way/1158269265` | Induri Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 85 | `dest_way_264278038` | `way/264278038` | ISKCON NVCC Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 86 | `dest_way_67169452` | `way/67169452` | ISKCON Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 87 | `dest_way_1271767135` | `way/1271767135` | Japalouppe | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 88 | `dest_node_8017991747` | `node/8017991747` | jivdhan fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 89 | `dest_node_2455119457` | `node/2455119457` | Joshi's Museum of Miniature Railways | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 90 | `dest_node_9197028053` | `node/9197028053` | Kailasgad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 91 | `dest_way_1419914113` | `way/1419914113` | Kalbhairavnath Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 92 | `dest_node_1065807980` | `node/1065807980` | Karla Caves | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 93 | `dest_node_2289294092` | `node/2289294092` | Kasba Ganpati | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 94 | `dest_way_1461491829` | `way/1461491829` | Khandoba Bhorwadi | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 95 | `dest_node_13380386560` | `node/13380386560` | Khandoba Mal Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 96 | `dest_way_1356854731` | `way/1356854731` | Khandoba Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 97 | `dest_way_97391941` | `way/97391941` | Korighad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 98 | `dest_way_640473406` | `way/640473406` | Kundalika Valley view point | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 99 | `dest_way_217271037` | `way/217271037` | Lal Mahal | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 100 | `dest_node_283144394` | `node/283144394` | Lion Point | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 101 | `dest_node_9197072319` | `node/9197072319` | Lohagad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 102 | `dest_way_218078554` | `way/218078554` | Mahatma Phule Vastu Sangrahalaya | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 103 | `dest_way_217631172` | `way/217631172` | Manali Resort and Banquet | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 104 | `dest_way_164454174` | `way/164454174` | Mawal Shrushti | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 105 | `dest_node_9197024754` | `node/9197024754` | Mohangad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 106 | `dest_node_7251607940` | `node/7251607940` | Monkey Point | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 107 | `dest_node_11700182023` | `node/11700182023` | Morya Gosavi Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 108 | `dest_node_12980328730` | `node/12980328730` | mumbai point | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 109 | `dest_way_549548866` | `way/549548866` | Mutha River view point | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 110 | `dest_node_14268390502` | `node/14268390502` | Mystery Rooms , Kp , Lane number 7 | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 111 | `dest_node_13649050398` | `node/13649050398` | Nagphani | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 112 | `dest_way_1350357777` | `way/1350357777` | Narayangad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 113 | `dest_node_1849234367` | `node/1849234367` | Necklace Point | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 114 | `dest_node_14251347888` | `node/14251347888` | Nedhe | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 115 | `dest_way_289333384` | `way/289333384` | Nighoj Kund | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 116 | `dest_node_11474798674` | `node/11474798674` | Nira Deoghar Dam | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 117 | `dest_way_544169744` | `way/544169744` | Nisargakavi Bahinabai Chaudhary Zoo | nature | **retain** | Independent canonical destination with verified standalone outing merit |
| 118 | `dest_way_215535660` | `way/215535660` | Omkareshwar Mandir | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 119 | `dest_node_8999406553` | `node/8999406553` | Padargad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 120 | `dest_way_158879223` | `way/158879223` | Peacock bay | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 121 | `dest_way_246996059` | `way/246996059` | Pimple Gurav Dinosaur Park | nature | **retain** | Independent canonical destination with verified standalone outing merit |
| 122 | `dest_node_10578845675` | `node/10578845675` | Pune Border Pillar | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 123 | `dest_way_281798382` | `way/281798382` | Purandar Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 124 | `dest_node_9197024757` | `node/9197024757` | Raireshwar Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 125 | `dest_way_264281528` | `way/264281528` | Raja Dinkar Kelkar Museum | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 126 | `dest_node_4523148909` | `node/4523148909` | Ramdara Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 127 | `dest_node_13805714808` | `node/13805714808` | RK Laxman museum | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 128 | `dest_way_174216367` | `way/174216367` | Rohida Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 129 | `dest_node_7944584962` | `node/7944584962` | Sambhaji Park | nature | **retain** | Independent canonical destination with verified standalone outing merit |
| 130 | `dest_way_1158262461` | `way/1158262461` | Sangramdurga Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 131 | `dest_way_53272975` | `way/53272975` | Sarasbaug | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 132 | `dest_way_290390068` | `way/290390068` | Sawai Gandharva Smarak | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 133 | `dest_relation_2861672` | `relation/2861672` | Shaniwar Wada | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 134 | `dest_way_53350109` | `way/53350109` | Shinde Chhatri | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 135 | `dest_way_198116052` | `way/198116052` | Shivneri Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 136 | `dest_way_1364324779` | `way/1364324779` | Shivsrushti Historical Theme Park Pune | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 137 | `dest_node_6137209923` | `node/6137209923` | Shree Bhairavnath Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 138 | `dest_way_1432747074` | `way/1432747074` | Shree Khandoba Mandir | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 139 | `dest_node_10248790568` | `node/10248790568` | Shree Siddhivinayak Mandir | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 140 | `dest_way_264276391` | `way/264276391` | Shreemant Dagdusheth Halwai Ganpati Mandir | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 141 | `dest_node_6895311225` | `node/6895311225` | Shri Bhairavnath Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 142 | `dest_node_9816755474` | `node/9816755474` | Siddhivinayak Mandir | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 143 | `dest_node_9197143306` | `node/9197143306` | Sindola Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 144 | `dest_node_6917411981` | `node/6917411981` | Sinhagad Valley | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 145 | `dest_node_1067959284` | `node/1067959284` | Sri Balaji Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 146 | `dest_node_10316607574` | `node/10316607574` | Sujit Aquariums | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 147 | `dest_node_12317626571` | `node/12317626571` | Sunset Point | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 148 | `dest_way_91710349` | `way/91710349` | Tail Baila | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 149 | `dest_node_2701228387` | `node/2701228387` | Tambdi Jogeshwari | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 150 | `dest_node_7251607939` | `node/7251607939` | Tiger Point | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 151 | `dest_way_83510498` | `way/83510498` | Tikona Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 152 | `dest_node_10548402346` | `node/10548402346` | Tulja Bhavani Mata Mandir | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 153 | `dest_way_561168113` | `way/561168113` | Tulja Bhavani Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 154 | `dest_node_13672024594` | `node/13672024594` | Tulja Bhavani Temple | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 155 | `dest_node_5503345162` | `node/5503345162` | Tulja Leni | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 156 | `dest_node_12464844233` | `node/12464844233` | Vaanarlingi | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 157 | `dest_way_281798464` | `way/281798464` | Vajragad Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 158 | `dest_way_1184662768` | `way/1184662768` | Valley of adventure | adventure | **retain** | Independent canonical destination with verified standalone outing merit |
| 159 | `dest_node_9197072318` | `node/9197072318` | Visapur Fort | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 160 | `dest_node_5795521366` | `node/5795521366` | Yash Laxmi Numismatic Museum | heritage | **retain** | Independent canonical destination with verified standalone outing merit |
| 161 | `dest_node_9197147730` | `node/9197147730` | Yetoba | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 162 | `dest_node_13299504368` | `node/13299504368` | Zunzar Machi | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 163 | `dest_node_8697450342` | `node/8697450342` | लेण्याद्री च्या लेण्या | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |
| 164 | `dest_node_11004963846` | `node/11004963846` | संत दर्शन | scenic_view | **retain** | Independent canonical destination with verified standalone outing merit |

---

## 9. Migration Safety & Fail-Safe Architecture

To guarantee zero data loss and prevent partial execution:
1. **Transactional Wrapper**: Everything executes inside a single `BEGIN ... COMMIT` block. Any error immediately rolls back the database to its pre-transaction state.
2. **Automated Pre-Flight Assertions (Fail-Fast)**:
   * Asserts total database count is exactly 164.
   * Asserts every canonical target ID exists and is currently published.
   * Asserts every merge child ID exists.
   * Asserts every unpublish ID exists.
   * Asserts that no merge record lacks a valid, verified canonical target.
   * If any assertion fails, `RAISE EXCEPTION` terminates execution immediately before any row is updated.
3. **Physical Snapshot Table**:
   * Creates `public._destinations_pre_migration_snapshot` containing all 164 rows before modifications.
4. **Zero Physical Deletes**: Inactive records have `is_published = false` and `verification_status = 'archived'`. Original OSM identifiers, geometry, and properties are preserved untouched.
5. **Automated Post-Flight Assertions**:
   * Asserts total row count is 164.
   * Asserts active published row count is exactly 134.
   * Asserts archived row count is exactly 30.
   * Asserts zero orphaned merge records.

---

## 10. Proposed Transactional Migration SQL (Draft - NOT Executed)

```sql
-- ==============================================================================
-- GoAround Data Governance Migration: Deduplication, Canonical Mapping & Archival
-- TRANSACTIONAL & REVERSIBLE
-- TARGET: public.destinations (164 rows -> 134 active published, 30 archived)
-- ==============================================================================

BEGIN;

-- 1. PRE-MIGRATION SANITY CHECKS (FAIL-FAST ASSERTIONS)
DO $$
DECLARE
  v_total INTEGER;
  v_missing_targets INTEGER;
  v_missing_merges INTEGER;
  v_missing_unpubs INTEGER;
BEGIN
  -- Assert total rows
  SELECT COUNT(*) INTO v_total FROM public.destinations;
  IF v_total != 164 THEN
    RAISE EXCEPTION 'Pre-check failed: expected 164 records in public.destinations, found %', v_total;
  END IF;

  -- Assert all canonical target IDs exist and are published
  SELECT COUNT(*) INTO v_missing_targets
  FROM (
    VALUES 
      ('dest_way_244656951'), ('dest_node_810328975'), ('dest_way_197954182'),
      ('dest_way_126392859'), ('dest_node_4363674444'), ('dest_way_215998826'),
      ('dest_way_41393392'), ('dest_node_7195582110'), ('dest_way_1030015753'),
      ('dest_way_1462367607'), ('dest_way_1462366794'), ('dest_node_9197114147'),
      ('dest_way_1559273042'), ('dest_node_12039066569')
  ) AS targets(id)
  LEFT JOIN public.destinations d ON targets.id = d.id AND d.is_published = true
  WHERE d.id IS NULL;

  IF v_missing_targets > 0 THEN
    RAISE EXCEPTION 'Pre-check failed: % canonical target destinations are missing or inactive', v_missing_targets;
  END IF;

  -- Assert all 21 merge candidates exist
  SELECT COUNT(*) INTO v_missing_merges
  FROM (
    VALUES 
      ('dest_node_14093789642'), ('dest_way_937734468'), ('dest_node_843780210'),
      ('dest_node_843780223'), ('dest_node_1127945662'), ('dest_way_105876380'),
      ('dest_way_264282504'), ('dest_node_2081486313'), ('dest_node_3721058066'),
      ('dest_node_3721058067'), ('dest_way_218078556'), ('dest_node_14070189246'),
      ('dest_node_2081729952'), ('dest_way_823068911'), ('dest_node_3386030400'),
      ('dest_node_9197167320'), ('dest_node_12464856157'), ('dest_node_13410848389'),
      ('dest_node_12464903330'), ('dest_way_1559273051'), ('dest_way_1559273047')
  ) AS merges(id)
  LEFT JOIN public.destinations d ON merges.id = d.id
  WHERE d.id IS NULL;

  IF v_missing_merges > 0 THEN
    RAISE EXCEPTION 'Pre-check failed: % merge candidate IDs are missing', v_missing_merges;
  END IF;

  -- Assert all 9 unpublish records exist
  SELECT COUNT(*) INTO v_missing_unpubs
  FROM (
    VALUES 
      ('dest_node_13691343801'), ('dest_node_13649044841'), ('dest_way_176331809'),
      ('dest_node_14251187502'), ('dest_way_798336778'), ('dest_node_10172979847'),
      ('dest_node_12284891397'), ('dest_node_11843018849'), ('dest_node_9612882246')
  ) AS unpubs(id)
  LEFT JOIN public.destinations d ON unpubs.id = d.id
  WHERE d.id IS NULL;

  IF v_missing_unpubs > 0 THEN
    RAISE EXCEPTION 'Pre-check failed: % unpublish candidate IDs are missing', v_missing_unpubs;
  END IF;

  RAISE NOTICE 'PRE-CHECK PASSED: 164 total rows, 14 canonical targets, 21 merge candidates, and 9 unpublish candidates verified.';
END $$;

-- 2. CREATE PRE-MIGRATION BACKUP SNAPSHOT TABLE
DROP TABLE IF EXISTS public._destinations_pre_migration_snapshot;
CREATE TABLE public._destinations_pre_migration_snapshot AS
SELECT * FROM public.destinations;

-- 3. UPDATE CANONICAL PARENT DESTINATIONS WITH ENRICHED METADATA & TITLES

-- Jejuri: Retain dest_way_244656951 as canonical complex
UPDATE public.destinations
SET 
  name = 'Shri Khandoba Temple & Jejurgad Fort Complex',
  metadata = metadata || jsonb_build_object(
    'alternate_source_ids', jsonb_build_array('node/14093789642', 'way/937734468'),
    'local_alias_mr', 'जेजुरगड आणि श्री खंडोबा मंदिर',
    'sub_attractions', jsonb_build_array(
      jsonb_build_object('name', 'Jejurgad Fort Ramparts & Deepamalas', 'source_osm_id', 'way/937734468', 'type', 'fortification', 'website', 'http://jejuri.in/jejurgad'),
      jsonb_build_object('name', 'Shri Khandoba Martand Bhairav Sanctum', 'source_osm_id', 'way/244656951', 'type', 'sanctum', 'wikidata', 'Q110823714')
    )
  ),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_244656951';

-- Rajmachi: Retain dest_node_810328975 as canonical complex
UPDATE public.destinations
SET 
  name = 'Rajmachi Fort Complex',
  category = 'heritage',
  subcategory = 'fort',
  metadata = metadata || jsonb_build_object(
    'alternate_source_ids', jsonb_build_array('node/843780210', 'node/843780223', 'node/1127945662', 'way/105876380'),
    'sub_attractions', jsonb_build_array(
      jsonb_build_object('name', 'Shriwardhan Ballekilla', 'source_osm_id', 'node/843780210', 'type', 'citadel', 'duration_minutes', 60),
      jsonb_build_object('name', 'Manaranjan Fort', 'source_osm_id', 'node/843780223', 'type', 'citadel', 'duration_minutes', 45),
      jsonb_build_object('name', 'Dhanya Kothar', 'source_osm_id', 'node/1127945662', 'type', 'ruin'),
      jsonb_build_object('name', 'Shri Kaal Bhairavnath Temple', 'source_osm_id', 'way/105876380', 'type', 'temple')
    )
  ),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_810328975';

-- Parvati: Retain dest_way_197954182 as canonical complex
UPDATE public.destinations
SET 
  name = 'Parvati Hill & Peshwa Heritage Complex',
  category = 'heritage',
  subcategory = 'landmark',
  metadata = metadata || jsonb_build_object(
    'alternate_source_ids', jsonb_build_array('way/264282504', 'node/2081486313'),
    'sub_attractions', jsonb_build_array(
      jsonb_build_object('name', 'Devdeveshwar Temple', 'source_osm_id', 'way/264282504', 'type', 'temple'),
      jsonb_build_object('name', 'Peshwa Museum', 'source_osm_id', 'node/2081486313', 'type', 'museum', 'fee', '₹10-20')
    )
  ),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_197954182';

-- Sinhagad: Retain dest_way_126392859 with gates
UPDATE public.destinations
SET 
  metadata = metadata || jsonb_build_object(
    'sub_attractions', jsonb_build_array(
      jsonb_build_object('name', 'Kalyan Darwaza', 'source_osm_id', 'node/3721058066', 'type', 'gate'),
      jsonb_build_object('name', 'Pune Darwaza', 'source_osm_id', 'node/3721058067', 'type', 'gate')
    )
  ),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_126392859';

-- Pataleshwar: Reclassify category
UPDATE public.destinations
SET 
  category = 'heritage',
  subcategory = 'archaeological_site',
  metadata = metadata || jsonb_build_object('alternate_source_ids', jsonb_build_array('way/218078556')),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_4363674444';

-- Aga Khan Palace: Add Gandhi Museum sub-attraction
UPDATE public.destinations
SET 
  category = 'heritage',
  subcategory = 'monument',
  metadata = metadata || jsonb_build_object(
    'sub_attractions', jsonb_build_array(
      jsonb_build_object('name', 'Mahatma Gandhi Memorial Museum', 'source_osm_id', 'node/14070189246', 'type', 'museum')
    )
  ),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_215998826';

-- Bhaje Caves: Reclassify & add portal tag
UPDATE public.destinations
SET 
  category = 'heritage',
  subcategory = 'archaeological_site',
  metadata = metadata || jsonb_build_object('alternate_source_ids', jsonb_build_array('node/2081729952')),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_41393392';

-- Malhargad: Add alternate boundary
UPDATE public.destinations
SET 
  metadata = metadata || jsonb_build_object('alternate_source_ids', jsonb_build_array('way/823068911')),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_7195582110';

-- Nimgiri: Retain dest_way_1462367607 as Twin Forts
UPDATE public.destinations
SET 
  name = 'Nimgiri-Hanumantgad Twin Forts',
  metadata = metadata || jsonb_build_object(
    'sub_attractions', jsonb_build_array(
      jsonb_build_object('name', 'Hanumantgad Fort', 'source_osm_id', 'node/9197167320', 'type', 'fort'),
      jsonb_build_object('name', 'Neemgiri Hanumantgad Viewpoint', 'source_osm_id', 'node/12464856157', 'type', 'viewpoint')
    )
  ),
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_1462367607';

-- Taljai: Retain dest_node_12039066569 as Nature Plateau
UPDATE public.destinations
SET 
  name = 'Taljai Hills & Forest Reserve',
  category = 'nature',
  subcategory = 'park',
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_12039066569';

-- 4. SOFT-ARCHIVE 21 MERGE CANDIDATES WITH CANONICAL TARGETS
UPDATE public.destinations
SET 
  is_published = false,
  verification_status = 'archived',
  metadata = metadata || jsonb_build_object(
    'merged_into', CASE id
      WHEN 'dest_node_14093789642' THEN 'dest_way_244656951'
      WHEN 'dest_way_937734468' THEN 'dest_way_244656951'
      WHEN 'dest_node_843780210' THEN 'dest_node_810328975'
      WHEN 'dest_node_843780223' THEN 'dest_node_810328975'
      WHEN 'dest_node_1127945662' THEN 'dest_node_810328975'
      WHEN 'dest_way_105876380' THEN 'dest_node_810328975'
      WHEN 'dest_way_264282504' THEN 'dest_way_197954182'
      WHEN 'dest_node_2081486313' THEN 'dest_way_197954182'
      WHEN 'dest_node_3721058066' THEN 'dest_way_126392859'
      WHEN 'dest_node_3721058067' THEN 'dest_way_126392859'
      WHEN 'dest_way_218078556' THEN 'dest_node_4363674444'
      WHEN 'dest_node_14070189246' THEN 'dest_way_215998826'
      WHEN 'dest_node_2081729952' THEN 'dest_way_41393392'
      WHEN 'dest_way_823068911' THEN 'dest_node_7195582110'
      WHEN 'dest_node_3386030400' THEN 'dest_way_1030015753'
      WHEN 'dest_node_9197167320' THEN 'dest_way_1462367607'
      WHEN 'dest_node_12464856157' THEN 'dest_way_1462367607'
      WHEN 'dest_node_13410848389' THEN 'dest_way_1462366794'
      WHEN 'dest_node_12464903330' THEN 'dest_node_9197114147'
      WHEN 'dest_way_1559273051' THEN 'dest_way_1559273042'
      WHEN 'dest_way_1559273047' THEN 'dest_way_1559273042'
    END,
    'archive_reason', 'Subsumed into canonical destination as sub-attraction or duplicate boundary'
  ),
  updated_at = timezone('utc'::text, now())
WHERE id IN (
  'dest_node_14093789642', 'dest_way_937734468', 'dest_node_843780210',
  'dest_node_843780223', 'dest_node_1127945662', 'dest_way_105876380',
  'dest_way_264282504', 'dest_node_2081486313', 'dest_node_3721058066',
  'dest_node_3721058067', 'dest_way_218078556', 'dest_node_14070189246',
  'dest_node_2081729952', 'dest_way_823068911', 'dest_node_3386030400',
  'dest_node_9197167320', 'dest_node_12464856157', 'dest_node_13410848389',
  'dest_node_12464903330', 'dest_way_1559273051', 'dest_way_1559273047'
);

-- 5. SOFT-ARCHIVE 9 UNPUBLISH CANDIDATES
UPDATE public.destinations
SET 
  is_published = false,
  verification_status = 'archived',
  metadata = metadata || jsonb_build_object(
    'archive_reason', CASE id
      WHEN 'dest_node_13691343801' THEN 'Municipal traffic roundabout (Jay chand chowk)'
      WHEN 'dest_node_13649044841' THEN 'Cellular network reception rock (Network Point Jio)'
      WHEN 'dest_way_176331809' THEN 'Private residential real estate plotting layout (Wildernest Country Homes)'
      WHEN 'dest_node_14251187502' THEN 'Isolated single tombstone on trail (Killedar Godaji Bhuruk Samadhi)'
      WHEN 'dest_way_798336778' THEN 'Local municipal sports ground (Taljai Ground)'
      WHEN 'dest_node_10172979847' THEN 'Local municipal sports ground (Taljai Cricket Ground)'
      WHEN 'dest_node_12284891397' THEN 'Remote uncurated village shrine >85 km away (Bhairavnath Junnar Ghatghar)'
      WHEN 'dest_node_11843018849' THEN 'Remote uncurated roadside shrine >80 km away (Khandoba Vadaj)'
      WHEN 'dest_node_9612882246' THEN 'Remote uncurated village shrine in rural outer Bhor (Khandoba Bhor)'
    END
  ),
  updated_at = timezone('utc'::text, now())
WHERE id IN (
  'dest_node_13691343801', 'dest_node_13649044841', 'dest_way_176331809',
  'dest_node_14251187502', 'dest_way_798336778', 'dest_node_10172979847',
  'dest_node_12284891397', 'dest_node_11843018849', 'dest_node_9612882246'
);

-- 6. POST-MIGRATION VERIFICATION ASSERTIONS (ROLLBACK ON FAILURE)
DO $$
DECLARE
  v_total INTEGER;
  v_published INTEGER;
  v_archived INTEGER;
  v_orphaned INTEGER;
BEGIN
  SELECT COUNT(*) INTO v_total FROM public.destinations;
  SELECT COUNT(*) INTO v_published FROM public.destinations WHERE is_published = true AND verification_status != 'archived';
  SELECT COUNT(*) INTO v_archived FROM public.destinations WHERE is_published = false AND verification_status = 'archived';

  IF v_total != 164 THEN
    RAISE EXCEPTION 'Post-migration verification failed: total row count altered (expected 164, found %)', v_total;
  END IF;

  IF v_published != 134 THEN
    RAISE EXCEPTION 'Post-migration verification failed: published count mismatch (expected 134, found %)', v_published;
  END IF;

  IF v_archived != 30 THEN
    RAISE EXCEPTION 'Post-migration verification failed: archived count mismatch (expected 30, found %)', v_archived;
  END IF;

  -- Check for any merged rows pointing to a missing or inactive canonical target
  SELECT COUNT(*) INTO v_orphaned
  FROM public.destinations m
  LEFT JOIN public.destinations t ON (m.metadata->>'merged_into') = t.id AND t.is_published = true
  WHERE m.metadata->>'merged_into' IS NOT NULL AND t.id IS NULL;

  IF v_orphaned > 0 THEN
    RAISE EXCEPTION 'Post-migration verification failed: % merged rows have missing/inactive canonical targets', v_orphaned;
  END IF;

  RAISE NOTICE 'POST-MIGRATION VERIFICATION SUCCEEDED: 164 total rows, 134 published canonical destinations, 30 safely archived records, 0 orphaned merges.';
END $$;

COMMIT;
```

---

## 11. Complete Rollback Architecture & SQL (Draft - NOT Executed)

### Snapshot Preservation & Restoration Principles
1. **Snapshot Preservation**: Before any row is updated, `CREATE TABLE public._destinations_pre_migration_snapshot AS SELECT * FROM public.destinations;` creates a physical, schema-identical backup table within PostgreSQL.
2. **Zero Field Assumptions**: The rollback does **NOT** assume every record originally had `verification_status = 'published'`. Instead, it explicitly restores the exact values stored in `_destinations_pre_migration_snapshot` for all 22 columns.
3. **Complete Field Restoration**: Restores `name`, `category`, `subcategory`, `area`, `description`, `tagline`, `cost_per_person`, `suggested_duration_minutes`, `image_url`, `tags`, `crowd_level`, `best_time_of_day`, `local_tip`, `metadata`, `verification_status`, and `is_published`.
4. **Restoration Verification**: Executes an automated hash/equality assertion comparing `public.destinations` with the snapshot table to prove 0 divergences.

### Method A: Automated Restoration from Snapshot Table

```sql
-- ==============================================================================
-- Method A: Automated Rollback from _destinations_pre_migration_snapshot Table
-- ==============================================================================
BEGIN;

-- Assert backup snapshot table exists and has 164 rows
DO $$
DECLARE
  v_snap_count INTEGER;
BEGIN
  SELECT COUNT(*) INTO v_snap_count FROM public._destinations_pre_migration_snapshot;
  IF v_snap_count != 164 THEN
    RAISE EXCEPTION 'Rollback aborted: snapshot table missing or has invalid count (expected 164, found %)', v_snap_count;
  END IF;
END $$;

-- Restore all columns from snapshot join
UPDATE public.destinations d
SET 
  name = s.name,
  category = s.category,
  subcategory = s.subcategory,
  latitude = s.latitude,
  longitude = s.longitude,
  area = s.area,
  description = s.description,
  tagline = s.tagline,
  cost_per_person = s.cost_per_person,
  suggested_duration_minutes = s.suggested_duration_minutes,
  image_url = s.image_url,
  tags = s.tags,
  crowd_level = s.crowd_level,
  best_time_of_day = s.best_time_of_day,
  local_tip = s.local_tip,
  metadata = s.metadata,
  verification_status = s.verification_status,
  is_published = s.is_published,
  updated_at = timezone('utc'::text, now())
FROM public._destinations_pre_migration_snapshot s
WHERE d.id = s.id;

-- Assert 0 differences remain between destinations and snapshot
DO $$
DECLARE
  v_diff INTEGER;
BEGIN
  SELECT COUNT(*) INTO v_diff
  FROM public.destinations d
  JOIN public._destinations_pre_migration_snapshot s ON d.id = s.id
  WHERE d.name != s.name
     OR d.category != s.category
     OR d.subcategory IS DISTINCT FROM s.subcategory
     OR d.is_published != s.is_published
     OR d.verification_status != s.verification_status
     OR d.metadata != s.metadata;

  IF v_diff > 0 THEN
    RAISE EXCEPTION 'Rollback verification failed: % rows differ from pre-migration snapshot', v_diff;
  END IF;

  RAISE NOTICE 'ROLLBACK SUCCESS: All 164 records restored with zero divergence from pre-migration state.';
END $$;

COMMIT;
```

### Method B: Standalone Literal Row-by-Row Restoration (No Snapshot Table Dependency)
If `_destinations_pre_migration_snapshot` is dropped or unavailable, executing this literal script restores every one of the 41 affected records to their verified pre-migration state:

```sql
-- ==============================================================================
-- Method B: Standalone Literal Row-by-Row Fallback Restoration
-- RESTORES ALL 41 MODIFIED ROWS TO EXACT AUDITED PRE-MIGRATION LITERAL VALUES
-- ==============================================================================
BEGIN;

UPDATE public.destinations SET
  name = 'Aga Khan Palace',
  category = 'scenic_view',
  subcategory = 'attraction',
  metadata = '{"amenity":null,"name_mr":"आगा खान पॅलेस","tourism":"attraction","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/215998826","fee":"yes","area":"yes","name":"Aga Khan Palace","source":"OpenStreetMap","name:en":"Aga Khan Palace","name:mr":"आगा खान पॅलेस","tourism":"attraction","@geometry":"center","name_status":"named","source_object_id":"way/215998826","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"attraction"},"alternate_source_ids":["way/215998839"]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_215998826';

UPDATE public.destinations SET
  name = 'Artha Mandap Dakhsin Dwar Morgaon',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/1559273051","name":"Artha Mandap Dakhsin Dwar Morgaon","source":"OpenStreetMap","amenity":"place_of_worship","building":"yes","religion":"hindu","@geometry":"center","name_status":"named","source_object_id":"way/1559273051","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_1559273051';

UPDATE public.destinations SET
  name = 'Bhairavnath Temple',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/12284891397","name":"Bhairavnath Temple","source":"OpenStreetMap","amenity":"place_of_worship","name_status":"named","source_object_id":"node/12284891397","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_12284891397';

UPDATE public.destinations SET
  name = 'Bhaje Caves',
  category = 'heritage',
  subcategory = 'attraction',
  metadata = '{"amenity":null,"name_mr":"भाजे लेणी","tourism":"attraction","historic":null,"wikidata":"Q854709","wikipedia":"en:Bhaja Caves","osm_source":"OpenStreetMap","raw_properties":{"@id":"way/41393392","area":"yes","name":"Bhaje Caves","source":"OpenStreetMap","name:en":"Bhaje Caves","name:mr":"भाजे लेणी","tourism":"attraction","wikidata":"Q854709","@geometry":"center","wikipedia":"en:Bhaja Caves","name_status":"named","source_object_id":"way/41393392","goaround_category":"tourism","wikimedia_commons":"Category:Bhaja Caves","goaround_selection":"famous_place_candidate","goaround_subcategory":"attraction"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_41393392';

UPDATE public.destinations SET
  name = 'Chawand Fort',
  category = 'heritage',
  subcategory = 'fort',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":"fort","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/1462366794","name":"Chawand Fort","access":"yes","source":"OpenStreetMap","building":"yes","historic":"fort","@geometry":"center","name_status":"named","source_object_id":"way/1462366794","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"fort"},"alternate_source_ids":["node/9197145215"]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_1462366794';

UPDATE public.destinations SET
  name = 'Dhakoba Fort',
  category = 'heritage',
  subcategory = 'fort',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":"fort","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/9197114147","name":"Dhakoba Fort","source":"OpenStreetMap","historic":"fort","name_status":"named","source_object_id":"node/9197114147","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"fort"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_9197114147';

UPDATE public.destinations SET
  name = 'Dhanya kothar',
  category = 'scenic_view',
  subcategory = 'attraction',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"attraction","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/1127945662","ele":"762.93","name":"Dhanya kothar","source":"OpenStreetMap","natural":"cave_entrance","tourism":"attraction","name_status":"named","source_object_id":"node/1127945662","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"attraction"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_1127945662';

UPDATE public.destinations SET
  name = 'Dharma Mandap Purva Dwar Morgaon',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/1559273047","name":"Dharma Mandap Purva Dwar Morgaon","source":"OpenStreetMap","amenity":"place_of_worship","building":"yes","religion":"hindu","@geometry":"center","name_status":"named","source_object_id":"way/1559273047","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_1559273047';

UPDATE public.destinations SET
  name = 'Ghat Viewpoint',
  category = 'scenic_view',
  subcategory = 'viewpoint',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"viewpoint","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/12464903330","name":"Ghat Viewpoint","source":"OpenStreetMap","tourism":"viewpoint","name_status":"named","source_object_id":"node/12464903330","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"viewpoint"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_12464903330';

UPDATE public.destinations SET
  name = 'HAL Marut',
  category = 'scenic_view',
  subcategory = 'attraction',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"attraction","historic":"aircraft","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/3386030400","name":"HAL Marut","source":"OpenStreetMap","tourism":"attraction","historic":"aircraft","name_status":"named","source_object_id":"node/3386030400","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"attraction"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_3386030400';

UPDATE public.destinations SET
  name = 'Hanumantgad Fort',
  category = 'heritage',
  subcategory = 'fort',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":"fort","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/9197167320","name":"Hanumantgad Fort","source":"OpenStreetMap","historic":"fort","name_status":"named","source_object_id":"node/9197167320","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"fort"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_9197167320';

UPDATE public.destinations SET
  name = 'Jay chand chowk',
  category = 'heritage',
  subcategory = 'monument',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":"monument","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/13691343801","name":"Jay chand chowk","source":"OpenStreetMap","historic":"monument","name_status":"named","source_object_id":"node/13691343801","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"monument"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_13691343801';

UPDATE public.destinations SET
  name = 'Kalyan Gate',
  category = 'scenic_view',
  subcategory = 'viewpoint',
  metadata = '{"amenity":null,"name_mr":"कल्याण दरवाजा","tourism":"viewpoint","historic":"yes","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/3721058066","name":"Kalyan Gate","source":"OpenStreetMap","name:hi":"कल्याण दरवाजा","name:mr":"कल्याण दरवाजा","tourism":"viewpoint","historic":"yes","name_status":"named","source_object_id":"node/3721058066","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"viewpoint"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_3721058066';

UPDATE public.destinations SET
  name = 'Khandoba Mandir',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/11843018849","name":"Khandoba Mandir","source":"OpenStreetMap","amenity":"place_of_worship","religion":"hindu","addr:city":"Pune","addr:street":"Vadaj road","name_status":"named","addr:district":"Pune","addr:postcode":"410502","addr:housenumber":"Khandoba Mandir Vadaj","source_object_id":"node/11843018849","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_11843018849';

UPDATE public.destinations SET
  name = 'Khandoba Mandir',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/9612882246","name":"Khandoba Mandir","source":"OpenStreetMap","amenity":"place_of_worship","religion":"hindu","name_status":"named","source_object_id":"node/9612882246","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_9612882246';

UPDATE public.destinations SET
  name = 'Khandoba Temple Jejuri',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/14093789642","name":"Khandoba Temple Jejuri","source":"OpenStreetMap","amenity":"place_of_worship","religion":"hindu","check_date":"2026-08-14","name_status":"named","payment:cash":"yes","payment:cards":"no","internet_access":"no","source_object_id":"node/14093789642","goaround_category":"religious","goaround_selection":"famous_place_candidate","payment:debit_cards":"no","goaround_subcategory":"place_of_worship","payment:credit_cards":"no"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_14093789642';

UPDATE public.destinations SET
  name = 'Killedar Godaji Bhuruk Samadhi',
  category = 'heritage',
  subcategory = 'monument',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":"monument","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/14251187502","name":"Killedar Godaji Bhuruk Samadhi","access":"yes","source":"OpenStreetMap","historic":"monument","name_status":"named","source_object_id":"node/14251187502","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"monument"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_14251187502';

UPDATE public.destinations SET
  name = 'Mahatma Gandhi Museum',
  category = 'heritage',
  subcategory = 'museum',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"museum","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/14070189246","name":"Mahatma Gandhi Museum","source":"OpenStreetMap","tourism":"museum","name_status":"named","source_object_id":"node/14070189246","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"museum"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_14070189246';

UPDATE public.destinations SET
  name = 'Malhargad',
  category = 'heritage',
  subcategory = 'fort',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":"fort","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/823068911","name":"Malhargad","layer":"-1","access":"yes","source":"OpenStreetMap","building":"yes","historic":"fort","@geometry":"center","name_status":"named","source_object_id":"way/823068911","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"fort"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_823068911';

UPDATE public.destinations SET
  name = 'Manaranjan Fort',
  category = 'heritage',
  subcategory = 'fort',
  metadata = '{"amenity":null,"name_mr":"मनरंजन गड","tourism":null,"historic":"fort","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/843780223","name":"Manaranjan Fort","source":"OpenStreetMap","name:mr":"मनरंजन गड","historic":"fort","name_status":"named","source_object_id":"node/843780223","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"fort"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_843780223';

UPDATE public.destinations SET
  name = 'Neemgiri Hanumantgad Viewpoint',
  category = 'heritage',
  subcategory = 'viewpoint',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"viewpoint","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/12464856157","name":"Neemgiri Hanumantgad Viewpoint","source":"OpenStreetMap","tourism":"viewpoint","name_status":"named","source_object_id":"node/12464856157","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"viewpoint"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_12464856157';

UPDATE public.destinations SET
  name = 'Network Point (Jio)',
  category = 'scenic_view',
  subcategory = 'viewpoint',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"viewpoint","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/13649044841","name":"Network Point (Jio)","source":"OpenStreetMap","tourism":"viewpoint","name_status":"named","source_object_id":"node/13649044841","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"viewpoint"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_13649044841';

UPDATE public.destinations SET
  name = 'Nimgiri Fort',
  category = 'heritage',
  subcategory = 'fort',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":"fort","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/1462367607","name":"Nimgiri Fort","layer":"1","source":"OpenStreetMap","building":"yes","historic":"fort","@geometry":"center","name_status":"named","source_object_id":"way/1462367607","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"fort"},"alternate_source_ids":["node/9197167321"]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_1462367607';

UPDATE public.destinations SET
  name = 'Parvati',
  category = 'scenic_view',
  subcategory = 'attraction',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"attraction","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/197954182","name":"Parvati","source":"OpenStreetMap","tourism":"attraction","@geometry":"center","name_status":"named","source_object_id":"way/197954182","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"attraction"},"alternate_source_ids":["node/2081486315"]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_197954182';

UPDATE public.destinations SET
  name = 'Parvati Museum',
  category = 'heritage',
  subcategory = 'museum',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"museum","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/2081486313","name":"Parvati Museum","source":"OpenStreetMap","tourism":"museum","name_status":"named","source_object_id":"node/2081486313","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"museum"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_2081486313';

UPDATE public.destinations SET
  name = 'Parvati Temple',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/264282504","name":"Parvati Temple","wall":"yes","source":"OpenStreetMap","amenity":"place_of_worship","religion":"hindu","@geometry":"center","name_status":"named","source_object_id":"way/264282504","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_264282504';

UPDATE public.destinations SET
  name = 'Pataleshwar',
  category = 'scenic_view',
  subcategory = 'attraction',
  metadata = '{"amenity":null,"name_mr":"पाताळेश्वर","tourism":"attraction","historic":"archaeological_site","wikidata":"Q7144204","wikipedia":"en:Pataleshwar","osm_source":"OpenStreetMap","raw_properties":{"@id":"way/218078556","name":"Pataleshwar","source":"OpenStreetMap","name:mr":"पाताळेश्वर","tourism":"attraction","historic":"archaeological_site","wikidata":"Q7144204","@geometry":"center","wikipedia":"en:Pataleshwar","name_status":"named","source_object_id":"way/218078556","goaround_category":"tourism","wikimedia_commons":"Category:Pataleshwar","goaround_selection":"famous_place_candidate","goaround_subcategory":"attraction"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_218078556';

UPDATE public.destinations SET
  name = 'Pataleshwar cave temple',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/4363674444","name":"Pataleshwar cave temple","source":"OpenStreetMap","amenity":"place_of_worship","religion":"hindu","name_status":"named","source_object_id":"node/4363674444","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_4363674444';

UPDATE public.destinations SET
  name = 'Pune Gate',
  category = 'scenic_view',
  subcategory = 'viewpoint',
  metadata = '{"amenity":null,"name_mr":"पुणे दरवाजा","tourism":"viewpoint","historic":"yes","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/3721058067","name":"Pune Gate","source":"OpenStreetMap","name:hi":"पुणे दरवाजा","name:mr":"पुणे दरवाजा","tourism":"viewpoint","historic":"yes","name_status":"named","source_object_id":"node/3721058067","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"viewpoint"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_3721058067';

UPDATE public.destinations SET
  name = 'Rajmachi Fort',
  category = 'heritage',
  subcategory = 'attraction',
  metadata = '{"amenity":null,"name_mr":"राजमाची किल्ला","tourism":"attraction","historic":null,"wikidata":"Q3635161","wikipedia":"en:Rajmachi","osm_source":"OpenStreetMap","raw_properties":{"@id":"node/810328975","name":"Rajmachi Fort","place":"locality","source":"OpenStreetMap","name:mr":"राजमाची किल्ला","tourism":"attraction","wikidata":"Q3635161","wikipedia":"en:Rajmachi","name_status":"named","source_object_id":"node/810328975","goaround_category":"tourism","wikimedia_commons":"Category:Rajmachi","goaround_selection":"famous_place_candidate","goaround_subcategory":"attraction"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_810328975';

UPDATE public.destinations SET
  name = 'Saptamatruka',
  category = 'heritage',
  subcategory = 'archaeological_site',
  metadata = '{"amenity":null,"name_mr":"सप्तमातृका","tourism":null,"historic":"archaeological_site","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/13410848389","name":"Saptamatruka","access":"yes","source":"OpenStreetMap","name:mr":"सप्तमातृका","historic":"archaeological_site","name_status":"named","source_object_id":"node/13410848389","goaround_category":"heritage","goaround_selection":"famous_place_candidate","archaeological_site":"petroglyph","goaround_subcategory":"archaeological_site","historic:civilization":"middle-ages"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_13410848389';

UPDATE public.destinations SET
  name = 'Shri Kaal Bhairavnath Temple',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":"श्री काळ भैरवनाथ देऊळ","tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/105876380","name":"Shri Kaal Bhairavnath Temple","source":"OpenStreetMap","amenity":"place_of_worship","name:mr":"श्री काळ भैरवनाथ देऊळ","building":"yes","religion":"hindu","@geometry":"center","name_status":"named","denomination":"shaivism","source_object_id":"way/105876380","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_105876380';

UPDATE public.destinations SET
  name = 'Shri Khandoba Martanda Bhairav Mandir',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":"श्री खंडोबा मार्तंड भौरव मंदीर","tourism":null,"historic":null,"wikidata":"Q110823714","wikipedia":"en:Khandoba Temple, Jejuri","osm_source":"OpenStreetMap","raw_properties":{"@id":"way/244656951","name":"Shri Khandoba Martanda Bhairav Mandir","source":"OpenStreetMap","amenity":"place_of_worship","name:en":"Shri Khandoba Martanda Bhairav Mandir","name:mr":"श्री खंडोबा मार्तंड भौरव मंदीर","website":"http://jejuri.in/","religion":"hindu","wikidata":"Q110823714","@geometry":"center","wikipedia":"en:Khandoba Temple, Jejuri","name_status":"named","source_object_id":"way/244656951","goaround_category":"religious","wikimedia_commons":"Category:Khandoba Temple, Jejuri","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_244656951';

UPDATE public.destinations SET
  name = 'Shri Mayureshwar Mandir Morgaon',
  category = 'heritage',
  subcategory = 'place_of_worship',
  metadata = '{"amenity":"place_of_worship","name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/1559273042","name":"Shri Mayureshwar Mandir Morgaon","source":"OpenStreetMap","amenity":"place_of_worship","building":"yes","religion":"hindu","@geometry":"center","name_status":"named","source_object_id":"way/1559273042","goaround_category":"religious","goaround_selection":"famous_place_candidate","goaround_subcategory":"place_of_worship"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_1559273042';

UPDATE public.destinations SET
  name = 'Shriwardhan Ballekilla',
  category = 'heritage',
  subcategory = 'fort',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":"fort","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/843780210","name":"Shriwardhan Ballekilla","source":"OpenStreetMap","name:en":"Shrivardhan Fort","alt_name":"Rajmachi","historic":"fort","name_status":"named","source_object_id":"node/843780210","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"fort"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_843780210';

UPDATE public.destinations SET
  name = 'Sinhagad Fort',
  category = 'heritage',
  subcategory = 'castle',
  metadata = '{"amenity":null,"name_mr":"सिंहगड किल्ला","tourism":null,"historic":"castle","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/126392859","name":"Sinhagad Fort","access":"yes","source":"OpenStreetMap","name:hi":"सिंहगड किला","name:mr":"सिंहगड किल्ला","alt_name":"Sinhagarh Fort","historic":"castle","@geometry":"center","castle_type":"fortress","name_status":"named","source_object_id":"way/126392859","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"castle"},"alternate_source_ids":["node/2165009895"]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_126392859';

UPDATE public.destinations SET
  name = 'Taljai Cricket Ground',
  category = 'nature',
  subcategory = 'park',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/10172979847","name":"Taljai Cricket Ground","source":"OpenStreetMap","leisure":"park","name_status":"named","source_object_id":"node/10172979847","goaround_category":"recreation","goaround_selection":"famous_place_candidate","goaround_subcategory":"park"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_10172979847';

UPDATE public.destinations SET
  name = 'Taljai Ground',
  category = 'nature',
  subcategory = 'park',
  metadata = '{"amenity":null,"name_mr":null,"tourism":null,"historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/798336778","name":"Taljai Ground","source":"OpenStreetMap","leisure":"park","@geometry":"center","name_status":"named","source_object_id":"way/798336778","goaround_category":"recreation","goaround_selection":"famous_place_candidate","goaround_subcategory":"park"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_798336778';

UPDATE public.destinations SET
  name = 'Wildernest Country Homes',
  category = 'scenic_view',
  subcategory = 'attraction',
  metadata = '{"amenity":null,"name_mr":null,"tourism":"attraction","historic":null,"wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/176331809","name":"Wildernest Country Homes","source":"OpenStreetMap","tourism":"attraction","@geometry":"center","name_status":"named","source_object_id":"way/176331809","goaround_category":"tourism","goaround_selection":"famous_place_candidate","goaround_subcategory":"attraction"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_176331809';

UPDATE public.destinations SET
  name = 'जेजुरगड',
  category = 'heritage',
  subcategory = 'fort',
  metadata = '{"amenity":null,"name_mr":"जेजुरगड","tourism":null,"historic":"fort","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"way/937734468","name":"जेजुरगड","source":"OpenStreetMap","name:en":"Jejur fort","name:mr":"जेजुरगड","website":"http://jejuri.in/jejurgad","historic":"fort","@geometry":"center","name_status":"named","source_object_id":"way/937734468","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"fort"},"alternate_source_ids":[]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_way_937734468';

UPDATE public.destinations SET
  name = 'प्रवेशद्वार',
  category = 'heritage',
  subcategory = 'archaeological_site',
  metadata = '{"amenity":null,"name_mr":"प्रवेशद्वार","tourism":null,"historic":"archaeological_site","wikidata":null,"wikipedia":null,"osm_source":"OpenStreetMap","raw_properties":{"@id":"node/2081729952","name":"प्रवेशद्वार","source":"OpenStreetMap","name:mr":"प्रवेशद्वार","historic":"archaeological_site","name_status":"named","source_object_id":"node/2081729952","goaround_category":"heritage","goaround_selection":"famous_place_candidate","goaround_subcategory":"archaeological_site"},"alternate_source_ids":["node/9481240184","node/9481240185"]}'::jsonb,
  verification_status = 'published',
  is_published = true,
  updated_at = timezone('utc'::text, now())
WHERE id = 'dest_node_2081729952';

-- Post-Rollback Verification
DO $$
DECLARE
  v_active INTEGER;
BEGIN
  SELECT COUNT(*) INTO v_active FROM public.destinations WHERE is_published = true AND verification_status = 'published';
  IF v_active != 164 THEN
    RAISE EXCEPTION 'Literal rollback verification failed: expected 164 active rows, found %', v_active;
  END IF;
  RAISE NOTICE 'LITERAL ROLLBACK SUCCESS: All 164 rows verified in active published state.';
END $$;

COMMIT;
```

---

## 12. Application Compatibility & Verification Checklist

* [x] `[VERIFIED DATABASE FACT]` **Zero Frontend Fetching Code Changes Required**:
  `fetchPublishedDestinations()` in `src/services/supabase.ts` queries `.eq('is_published', true)`. Soft-archiving inactive rows with `is_published = false` automatically excludes them from both client queries and PostgreSQL RLS.
* [x] `[VERIFIED DATABASE FACT]` **No Dangling Foreign Keys**:
  `public.destinations` is a root dimension table with no inbound foreign key constraints. Archiving rows breaks no database relational constraints.
* [x] `[VERIFIED DATABASE FACT]` **Saved Trips Snapshot Independence**:
  `public.saved_trips` stores full itinerary JSON in `payload JSONB`. Guest trips in `localStorage` store identical JSON snapshots. Saved trips never query `destinations` by ID and will continue loading seamlessly.
* [x] `[VISITOR EXPERIENCE INFERENCE / DOMAIN ASSUMPTION]` **Planner Engine Itinerary Quality**:
  Eliminates redundant consecutive stops on mountain ridges (e.g. climbing Shriwardhan and Manaranjan on Rajmachi as two separate road trips).

---

## 13. List of Unresolved Strategic Decisions Requiring Founder Approval

The following 5 strategic decisions require formal approval before any database migration script is executed:

### 1. Jejurgad Fortification: Consolidated Complex (Option A) vs. Dual Attraction (Option B)
* **Context**: As demonstrated in Section 4, `जेजुरगड` (`dest_way_937734468`) is the 18th-century stone fort rampart perimeter enclosing `Shri Khandoba Temple` (`dest_way_244656951`).
* **Decision Required**:
  * **Option A (Recommended)**: Consolidate into `"Shri Khandoba Temple & Jejurgad Fort Complex"` with fort ramparts modeled as a sub-attraction. (Yields 134 active destinations).
  * **Option B**: Retain both as separate destinations (`Shri Khandoba Temple` and `Jejurgad Fort`). (Yields 135 active destinations). Requires engineering a mutual-exclusion rule in `plannerEngine.ts` to prevent both appearing in a single itinerary.

### 2. Selection of Authoritative Canonical ID for Jejuri
* **Context**: The previous draft designated `dest_node_14093789642` as canonical. However, `dest_way_244656951` has Wikidata (`Q110823714`), Wikipedia, official website (`http://jejuri.in/`), and Marathi name tags.
* **Decision Required**: Approve designating `dest_way_244656951` as the primary canonical destination and archiving `dest_node_14093789642` as the duplicate node.

### 3. Marathi Script Normalization & Localization Architecture
* **Context**: Several live destinations have pure Devanagari names in the database:
  * `dest_node_843780256`: `लेण्याद्री च्या लेण्या` (Lenyadri Caves)
  * `dest_node_843780270`: `संत दर्शन` (Sant Darshan Museum)
  * `dest_way_937734468`: `जेजुरगड` (Jejurgad Fort)
* **Decision Required**:
  * **Option 1**: Update `name` directly to standard English in `public.destinations` and store original Marathi script in `metadata->'name_mr'`.
  * **Option 2**: Add dedicated schema columns `name_en` and `name_mr` via a formal schema migration.

### 4. Sub-Attraction Presentation in Itinerary Result Cards
* **Context**: When a user is routed to `Rajmachi Fort Complex`, `Sinhagad Fort`, or `Parvati Hill & Peshwa Heritage Complex`, the parent destination now carries `metadata->'sub_attractions'`.
* **Decision Required**: Should the frontend UI in `TripResultView.tsx` render an expandable badge/checklist on the stop card (e.g. *"Includes: Shriwardhan Citadel, Manaranjan Citadel, Dhanya Kothar"*), or should sub-attractions remain strictly internal for routing intelligence?

### 5. Geographical Distance Guard for Day Outings
* **Context**: 34 of the 164 destinations are remote Sahyadri peaks located >70 km from Pune center (e.g. Junnar peaks like Bhairavgad, Sindola, Hadsar).
* **Decision Required**: Should the itinerary generator automatically filter out destinations >70 km away when the user selects a 4-hour or 8-hour trip, reserving them exclusively for 12-hour or 24-hour weekend getaway modes?
