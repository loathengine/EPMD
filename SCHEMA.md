# Empirical Precision — Database Reference

Authoritative, code-derived reference for **both** data stores in EPv4. It is detailed enough to
write read/write code against either store without inspecting live data. Every table, field, type,
nullability, unit, meaning, index, and relationship is listed.

**Sources of truth for this document** (do not diverge from these):
- `src/types/database.ts` — all TypeScript entity interfaces.
- `src/db/database.ts` — the Dexie class `ReloaderDB` (DB name `reloadingDB_v2`), the `version(1..5)`
  schema blocks, `repairImportData`, `syncFromMasterData`, `restoreUnifiedDatabase`,
  `exportUnifiedDatabase`, `clearDatabase`.
- `test_harnes/json/local-db.json` — the shipped reference database + `tuning` blob (structure inspected;
  field names/types below are taken from real records, values are not reproduced).

---

## 0. The Two Storage Layers

| | **A. IndexedDB** | **B. `local-db.json`** |
|---|---|---|
| Engine | Dexie over browser IndexedDB, DB name **`reloadingDB_v2`** | A single JSON file (also served at runtime as `master-db.json`) |
| Lives | In the user's browser | In the repo / on GitHub Pages, fetched by the app |
| Holds | Everything: shipped reference data **merged with** the user's local firearms, loads, sessions, marking, chrono, etc. | The curated component library (cartridges, bullets, powders, brass, diameters, manufacturers, primers, primer pockets, grain types) **plus** a `tuning` blob of calibrated engine parameters |
| Written by | The app UI, and by `syncFromMasterData` / `restoreUnifiedDatabase` on import | `calibrateV4.ts` (tuning only) and direct/scripted edits (physical data) |
| Authority | The user's live working copy | **The source of truth** for component/geometry/tuning data (CLAUDE.md §3.1) |

At runtime the app **fetches `local-db.json`** and calls `syncFromMasterData(data)`, which merges the
`tuning` blob back onto the physical records and writes them into IndexedDB. So the two layers share the
same record shapes for the component tables; IndexedDB additionally carries user-only tables that never
appear in `local-db.json`, and `local-db.json` additionally carries the `tuning` blob that is not a Dexie
table (it is stashed in the `meta` table under id `local-db`).

---

## 1. Entity-Relationship Overview

```
manufacturers ─┬─< bullets >─── diameters
               ├─< powders
               ├─< primers >──── primerPockets
               └─< brass
diameters ──────< cartridges >── primerPockets

cartridges ──┬─< brass
             ├─< loads >── bullets, powders, primers, brass, cartridges
             └─< firearms

firearms ──┐
loads ─────┼─< sessions ── markedTargets
markedTargets ─┬─< sessionTargets ── targetImages ── (customTargets)
               ├─< groups ──< shots
               └─ (shots also reference groups & targets)

chronoSessions ──(shots[].linkedGroupId)──> groups     (soft link only)
monteCarloSaves         (standalone)
grainTypes  <── powders.grainType (string id reference)
```

**Legend:** `A ─< B` means one A has many B (B holds A's id as a foreign key).

**CRITICAL keying quirk** (see §4.4): in `sessionTargets`, `groups`, and `shots`, the field named
`sessionId` **does not hold a `Session.id`** — it holds a **`MarkedTarget.id`**. Marking data is keyed by
marked target, not by session. `Session` links to its marked target via `Session.markedTargetId`.

---

## 2. Units Convention

- **Internal / storage is SI-metric.** Engines, calibrator, and every persisted component/load field use
  metric: grams, millimeters, meters/second, Pascals, kg/m³, kJ/kg, degrees C, degrees, Joules.
- **Imperial only at the UI/IO boundary.** Conversions (inches, grains, fps, °F, inHg, feet, yards)
  happen in the UI or importer, never inside engines or storage. Never let imperial units flow into
  storage for component/load records.
- **Documented exceptions** — a handful of *user-facing convenience* records intentionally store imperial
  because they mirror UI inputs directly, and are flagged inline below:
  - `MonteCarloSave.params.*` — stored in imperial (fps, inches, yards, °F, inHg, feet, mph, MOA).
  - `Session` environment fields (`temp` °F, `altitude` ft, `pressure` inHg).
  - `Shot.velocity` and `Shot.x/y` — fps and inches/cm (`units` field says which).
  - `ChronoShot.velocityFps` — fps (normalized from device m/s or mm/s on import).
  - `Bullet.advertisedWeightGrains`, `Diameter.imperialName` — display/label fields only.

---

## 3. ID Conventions

IDs are always strings and are the source of truth (never the name). Two families:

### 3.1 Curated master-db IDs (structured, permanent)
`<PREFIX>_<...>_<HASH>`, ALL CAPS, `_` separators, no spaces/punctuation, ending in a 4-char hash suffix.

| Table | Prefix | Pattern | Example |
|-------|--------|---------|---------|
| manufacturers | `MAN_` | `MAN_<SLUG>_<HASH>` | `MAN_HODG_M1G1` |
| diameters | `DIA_` | `DIA_<IMPERIAL>_<HASH>` | `DIA_308WIN_XBM7` |
| cartridges | `CTG_` | `CTG_<SLUG>_c<HASH>` | `CTG_65CREEDMR_cN12` |
| bullets | `BUL_` | `BUL_<MAN>_<DIA>_<WT>_<NAME>_<HASH>` | `BUL_SIERRA_224_50_BK_2B3E` |
| powders | `PWD_` | `PWD_<MAN>_<NAME>_<HASH>` | `PWD_HODG_VARGET_H1G1` |
| primers | `PRI_` | `PRI_<MAN>_<NAME>_<HASH>` | `PRI_FED_210M_P1H1` |
| primer pockets | `PKT_` | `PKT_<SIZE>` (fixed) | `PKT_SML`, `PKT_LRG` |
| brass | `BRS_` | `BRS_<MAN>_<CTG>_<HASH>` | `BRS_LAPUA_308WIN_B1H1` |
| loads | `LOAD_` / `LOD_` | `LOAD_<HEX16>` | `LOAD_71b6d8c00511a4f3` |
| monte carlo saves | `MCS_` | `MCS_<SLUG>` | `MCS_HDES_HIGH_DESERT` |

The prefix list above is exactly the `tablePrefixes` map in `syncFromMasterData` (`MAN_ DIA_ BUL_ PWD_
PRI_ PKT_ BRS_ CTG_ LOAD_ MCS_`). Only records whose id starts with that table's prefix are **pruned** as
stale on sync — user records (other id shapes) are never auto-pruned. `grainTypes` uses bare semantic ids
(`ball`, `flake`, `extrudedSinglePerf`, `extrudedMultiPerf`, `extruded`).

### 3.2 User-created IDs
`generateUniqueId()` (`src/utils/id.ts`) returns **`Date.now().toString(36) + Math.random().toString(36).slice(2)`**
— a base-36 timestamp concatenated with a random base-36 string (NOT a UUID). Any imported record missing
an `id` is assigned one via this function on sync/restore. Some legacy seed records use UUID-v4-shaped ids;
both forms are valid opaque strings — never parse them.

---

# LAYER A — IndexedDB (Dexie `reloadingDB_v2`)

## 4. Dexie Schema

`ReloaderDB extends Dexie` declares 22 tables. Every table uses a **string primary key `id`** (outbound —
Dexie does not auto-generate it; the app supplies it). Store strings list the primary key first, then
indexed properties.

### 4.1 Store strings (current = v5)

| Table (Dexie `Table<T>`) | Store string (`id` = PK) |
|---|---|
| `manufacturers` | `id` |
| `diameters` | `id` |
| `bullets` | `id, manufacturerId, diameterId` |
| `powders` | `id, manufacturerId` |
| `primers` | `id, manufacturerId` |
| `primerPockets` | `id` |
| `brass` | `id, cartridgeId, manufacturerId` |
| `cartridges` | `id, diameterId` |
| `loads` | `id, cartridgeId, bulletId, powderId` |
| `firearms` | `id, cartridgeId` |
| `customTargets` | `id` |
| `targetImages` | `id` |
| `markedTargets` | `id` |
| `sessions` | `id, firearmId, loadId, markedTargetId, [firearmId+loadId]` |
| `sessionTargets` | `id, sessionId` |
| `groups` | `id, sessionId` |
| `shots` | `id, groupId, sessionId, targetId` |
| `monteCarloSaves` | `id` |
| `chronoSessions` | `id` |
| `meta` | `id` |
| `grainTypes` | `id` |

Note: the indexed `sessionId` on `sessionTargets`/`groups`/`shots` actually indexes a **markedTargetId**
value (§4.4). The compound `[firearmId+loadId]` index on `sessions` supports "all sessions for this exact
firearm+load" queries.

### 4.2 Version history (v1 → v5)

Each `this.version(n).stores({...})` block re-declares the full store set (Dexie requirement). Schema
deltas per version:

| Version | Change |
|---|---|
| **v1** | Base schema: `manufacturers, diameters, bullets, powders, primers, primerPockets, brass, cartridges, loads, firearms, customTargets, targetImages, markedTargets, sessions, sessionTargets, groups, shots`. Marking tables already keyed as in v5. |
| **v2** | **+ `monteCarloSaves`** (`id`). |
| **v3** | **+ `chronoSessions`** (`id`). |
| **v4** | **+ `meta`** (`id`) — runtime store for the `tuning` blob (id `local-db`). |
| **v5** | **+ `grainTypes`** (`id`) — grain-geometry lookup referenced by `Powder.grainType`. |

No indexes were dropped or renamed across versions; every version is purely additive, so no data-migration
callback (`.upgrade(...)`) is attached to any version. The only data reshaping is `repairImportData`,
applied at **import time** (not as a Dexie upgrade) — see §4.5.

### 4.3 IndexedDB-only tables

`chronoSessions` and `meta` exist only in IndexedDB and are **never serialized to `local-db.json`**.
`meta` holds a single conventional row `{ id: 'local-db', tuning: {...} }`.

### 4.4 CRITICAL quirk — marking data is keyed by `markedTargetId`, not `sessionId`

Historically, marking data (targets/groups/shots) was keyed by the session id. The app later split
"a marked target" (the physical annotated target sheet) from "a session" (a range outing). To avoid a
destructive rename, the **field is still named `sessionId`** on `sessionTargets`, `groups`, and `shots`,
but it now stores a **`MarkedTarget.id`**.

- `Session.markedTargetId` → `MarkedTarget.id` is the real session→target link.
- `SessionTarget.sessionId`, `Group.sessionId`, `Shot.sessionId` → all hold a **`MarkedTarget.id`**.
- To load a session's marking data: read `session.markedTargetId`, then query
  `groups.where('sessionId').equals(markedTargetId)` (likewise `shots`, `sessionTargets`). Querying with
  the actual session id returns nothing.
- `Shot.groupId → Group.id` and `Shot.targetId / Group.targetId → SessionTarget.id` are ordinary links.

### 4.5 `repairImportData(data)` migration (import-time)

Run by both `syncFromMasterData` and `restoreUnifiedDatabase` on the incoming JSON, in-place, before any
write. Two jobs:

1. **Blob rehydration for `targetImages`:** if an item has a `dataUrl` string and no `imageBlob`, convert
   the Base64 data URL back to a `Blob` (`dataUrlToBlob`) into `imageBlob` and delete `dataUrl`. (The
   inverse — `imageBlob → dataUrl` — happens in `exportUnifiedDatabase`; `imageBlob` is a binary `Blob` in
   IndexedDB and cannot be JSON-serialized directly.)
2. **Legacy marking re-key (pre-split data):** for each `session` that has **no `markedTargetId`** but does
   have marking data (any `group`/`shot`/`sessionTarget` whose `sessionId === session.id`):
   - Create a `markedTargets` record with id derived from the session id: `SES_...` → `MKT_...`, else
     `MKT_<sessionId>`. Name = session name with "Session" replaced by "Marked Target". Copies
     `targetDistance`, `distanceUnits` (default `'yards'`), `createdAt = Date.now()`.
   - Set `session.markedTargetId = mtId`.
   - **Re-key** every matching `group.sessionId`, `shot.sessionId`, `sessionTarget.sessionId` from the old
     session id to the new `mtId`, enforcing the §4.4 invariant on legacy data.

### 4.6 Sync / restore / export / clear semantics

- **`syncFromMasterData(data)`** (used for the shipped reference DB): merges `data.tuning.powders[]` and
  `data.tuning.cartridges[]` onto matching physical records by id (`Object.assign`), runs
  `repairImportData`, then in one `rw` transaction: for each incoming array table, **prunes** stale
  master records (existing keys that start with the table's prefix but are absent from the incoming id
  set) via `bulkDelete`, `put`s every incoming item (assigning an id if missing), and finally writes the
  whole `tuning` blob to `meta` as `{ id: 'local-db', tuning }`. Returns `false` if no valid arrays.
- **`syncFromMaster(url?)`**: `fetch` the URL (default the EPMD GitHub raw `master-db.json`) → `json()` →
  `syncFromMasterData`.
- **`restoreUnifiedDatabase(jsonData)`** (user backup restore): `repairImportData`, then per table `put`s
  items — **except** for the "master tables" set `{manufacturers, diameters, bullets, powders, primers,
  primerPockets, brass, cartridges}`, where it only inserts records that **don't already exist** (so a
  stale snapshot inside a session-export file cannot clobber calibrated master data). No pruning. Does not
  touch `meta`/tuning.
- **`exportUnifiedDatabase(tableName?)`**: dumps all tables (or one) to pretty JSON; for `targetImages`
  converts `imageBlob → dataUrl` Base64.
- **`clearDatabase(tableName?)`**: `clear()` all tables or one.

---

## 5. IndexedDB Table Reference (per entity)

Types are from `src/types/database.ts`. `?` = optional/absent-allowed. "null" in Type means the field may
be explicitly `null`. Component tables (manufacturers…grainTypes) are shared with Layer B and detailed in
§7; below are the definitions with foreign keys and indexes noted. User-only tables are fully detailed
here.

### 5.1 `firearms` — user-only. Index: `id, cartridgeId`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`generateUniqueId`). |
| `nickname` | string | — | User label (required). |
| `cartridgeId` | string | — | FK → `cartridges.id` (required). |
| `barrelLengthMm`? | number | mm | Barrel length. |
| `twistRateMm`? | number | mm/turn | Rifling twist (203.2 = 1:8"). |
| `sightOverBoreMm`? | number | mm | Scope centerline above bore. |
| `magCoalMm`? | number | mm | Magazine max COAL constraint. |
| `freeboreMm`? | number | mm | Measured freebore for this barrel; overrides cartridge SAAMI spec. |
| `velocityScaleFactor`? | number | ratio | `sessionMeanVelocityFps / predictedVelocityFps`. |
| `velocityOffsetFps`? | number | fps | `sessionMeanVelocityFps − predictedVelocityFps`. |
| `velocityOffsetShotCount`? | number | count | Shots used to derive the offset. |
| `velocityOffsetSd`? | number | fps | SD of those shots. |
| `velocityOffsetDate`? | string | ISO date | When offset was saved. |
| `velocityOffsetSessionId`? | string | — | Session the offset came from. |
| `velocityOffsetTuningStamp`? | string | — | `tuning.generatedAt` at true-ing time; mismatch vs current tuning ⇒ stale factor. |
| `velocityOffsetFlag`? | string | — | Warning, e.g. `"large offset - verify inputs"`. |

### 5.2 `loads` — master-db + user. Index: `id, cartridgeId, bulletId, powderId`
Full field table in §7.9 (identical shape in both layers).

### 5.3 `markedTargets` — user-only. Index: `id`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. Referenced by `sessionTargets/groups/shots.sessionId` and `Session.markedTargetId`. |
| `name`? | string | — | Label. |
| `targetDistance`? | number | value in `distanceUnits` | Distance to target. |
| `distanceUnits`? | string | — | `"yards"` or `"meters"`. |
| `createdAt`? | number | ms epoch | Creation timestamp. |

### 5.4 `sessions` — user-only. Index: `id, firearmId, loadId, markedTargetId, [firearmId+loadId]`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. |
| `name`? | string | — | Label. |
| `markedTargetId`? | string | — | FK → `markedTargets.id` (the real session→marking link). |
| `firearmId`? | string | — | FK → `firearms.id`. |
| `loadId`? | string | — | FK → `loads.id`. |
| `targetDistance`? | number | value in `distanceUnits` | Distance. |
| `distanceUnits`? | string | — | `"yards"` / `"meters"`. |
| `temp`? | number | **°F** | Environment temperature (imperial — UI mirror). |
| `altitude`? | number | **feet** | Local altitude. |
| `pressure`? | number | **inHg** | Barometric/station pressure. |
| `pressureType`? | string | — | `"station"` or `"sea"`. |

### 5.5 `sessionTargets` — user-only. Index: `id, sessionId`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. |
| `sessionId` | string | — | **Holds a `MarkedTarget.id`** (not a session id — §4.4). |
| `targetImageId` | string | — | FK → `targetImages.id`. |
| `scale` | object | — | Pixel-to-physical calibration (below). |
| `scale.p1` | `{x,y}` \| null | px | First scale reference point. |
| `scale.p2` | `{x,y}` \| null | px | Second scale reference point. |
| `scale.distance` | number \| null | value in `scale.units` | Physical distance p1↔p2. |
| `scale.units` | string | — | e.g. `"in"`, `"cm"`. |
| `scale.pixelsPerUnit` | number \| null | px/unit | Derived scale factor. |
| `transform` | `{scale:number}` | ratio | View transform (zoom). |

### 5.6 `groups` — user-only. Index: `id, sessionId`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. Referenced by `shots.groupId` and `ChronoShot.linkedGroupId`. |
| `sessionId` | string | — | **Holds a `MarkedTarget.id`** (§4.4). |
| `targetId` | string | — | FK → `sessionTargets.id`. |
| `groupNum`? | number | index | Group order within the marked target. |
| `poa` | `{x,y}` \| null | px | Point-of-aim pixel coords on the image. |
| `color` | string | — | Hex color for rendering (required). |

### 5.7 `shots` — user-only. Index: `id, groupId, sessionId, targetId`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. |
| `groupId` | string | — | FK → `groups.id`. |
| `sessionId` | string | — | **Holds a `MarkedTarget.id`** (§4.4). |
| `targetId` | string | — | FK → `sessionTargets.id`. |
| `shotNumber` | number | index | Shot sequence number. |
| `x` | number | value in `units` | Horizontal offset from POA. |
| `y` | number | value in `units` | Vertical offset from POA (positive = up). |
| `units` | string | — | `"in"` or `"cm"` (imperial/metric per target). |
| `velocity` | number \| null | **fps** | Muzzle velocity (manual entry or chrono import); `null` if unrecorded. |
| `px` | number | px | Raw pixel X on canvas. |
| `py` | number | px | Raw pixel Y on canvas. |

### 5.8 `chronoSessions` — IndexedDB-only. Index: `id`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. |
| `name` | string | — | Import label. |
| `deviceType` | enum string | — | `'labradar' \| 'magnetospeed' \| 'garmin_xero' \| 'fit' \| 'generic'`. |
| `importedAt` | number | ms epoch | Import time. |
| `shots` | `ChronoShot[]` | — | Embedded array (below). |

`ChronoShot`:

| Field | Type | Units | Meaning |
|---|---|---|---|
| `shotNumber` | number | index | Sequence number from export file. |
| `velocityFps` | number | **fps** | Muzzle velocity (normalized from device units on import). |
| `timestamp`? | string | seconds string | Original time value if present. |
| `linkedGroupId`? | string \| null | — | Soft FK → `groups.id` this chrono shot is tied to. |
| `linkedShotIndex`? | number \| null | 0-based index | Position within the linked group's shot list. |

### 5.9 `monteCarloSaves` — user-only. Index: `id`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`MCS_...`). |
| `name` | string | — | Scenario label. |
| `createdAt` | number | ms epoch | Save time. |
| `params` | object | **imperial** | External-ballistics scenario inputs (all imperial — direct UI mirror). |

`params` fields — every value is `number \| null` unless noted:
`mv` (fps), `bulletWeight` (grains), `bulletDiam` (inches), `bc`, `bcType` (`'G1'\|'G7'`), `twist`
(inches/turn), `sightHeight` (inches), `zeroDist` (yards), `zeroOffsetX`/`zeroOffsetY` (in),
`cantDegrees` (deg), `temp` (°F), `pressure` (inHg), `humidity` (%), `altitude` (ft), `windSpeed` (mph),
`windDir` (deg), `mvSd`, `bcSd`, `windSpeedSd`, `windDirSd`, `windEstimateSd`, `rangeErrorSd`,
`precisionMoa` (MOA), `cantSd` (deg), `latitude` (deg), `azimuth` (deg),
`targetShape` (`'circle'\|'rectangle'\|'ipsc'`), `targetWidth`/`targetHeight` (in), `numRuns` (count),
`rangeMax` (yards), `rangeStep` (yards).

### 5.10 `targetImages` — user-only. Index: `id`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. |
| `name` | string | — | Label. |
| `imageBlob` | Blob | binary | Stored as native `Blob` in IndexedDB; serialized to/from a Base64 `dataUrl` at the export/import boundary. May be absent on config-only target records. |
| `size` | string | — | Human-readable size, e.g. `"245 KB"`. |
| `firearmId`? | string | — | FK → `firearms.id`. |
| `loadId`? | string | — | FK → `loads.id`. |
| `customTargetConfig`? | `CustomTarget` | — | Embedded target-template config (§5.11 shape). |

### 5.11 `customTargets` — user-only. Index: `id`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. |
| `name` | string | — | Label. |
| `paperSize`? | string | — | e.g. `"letter"`. |
| `orientation`? | string | — | `"portrait"` / `"landscape"`. |
| `gridEnabled`? | boolean | — | Draw grid. |
| `gridSize`? | number | inches | Grid cell size. |
| `gridColor`? | string | — | Hex. |
| `rows`?, `cols`? | number | count | Target layout grid. |
| `marginX`?, `marginY`? | number | inches | Page margins. |
| `shape`? | string | — | e.g. `"circle"`. |
| `diameter`? | number | inches | Bullseye/target diameter. |
| `numRings`? | number | count | Scoring rings. |
| `bullseyeColor`?, `ringColorA`?, `ringColorB`? | string | — | Hex colors. |
| `labelText`? | string | — | Caption. |
| `labelPosition`? | string | — | e.g. `"bottom"`. |
| `labelSize`? | number | pt | Caption font size. |
| `labelMargin`? | number | inches | Caption margin. |

### 5.12 `meta` — IndexedDB-only. Index: `id`
Runtime scratch table. One conventional row: `{ id: 'local-db', tuning: <tuning blob> }` (see §8). Typed
`Table<any, string>`; read via `db.meta.get('local-db')`.

---

# LAYER B — `local-db.json` (Reference Database + Tuning)

## 6. Top-Level Structure

`local-db.json` is a single JSON object. Component arrays share record shapes with the IndexedDB tables of
the same name; the `tuning` object is unique to this layer.

| Key | Type | Count (shipped) | Notes |
|---|---|---|---|
| `grainTypes` | array | 5 | Grain-geometry lookup. |
| `diameters` | array | 9 | Caliber definitions. |
| `cartridges` | array | 126 | Cartridge geometry + pressure ceilings. |
| `powders` | array | 152 | Powder physical constants (calibrated fields live in `tuning`). |
| `bullets` | array | 1008 | Projectiles + geometry + ballistics. |
| `brass` | array | 201 | Case capacity per manufacturer/cartridge. |
| `firearms` | array | 3 | (User records may ride along in exports.) |
| `loads` | array | 5 | Recipes. |
| `customTargets` | array | 2 | Target templates. |
| `targetImages` | array | 8 | (Config-only in the shipped file.) |
| `sessions` | array | 6 | |
| `sessionTargets` | array | 7 | |
| `groups` | array | 19 | |
| `shots` | array | 95 | |
| `manufacturers` | array | 38 | Brands. |
| `primerPockets` | array | 2 | `PKT_SML`, `PKT_LRG`. |
| `primers` | array | 13 | |
| `markedTargets` | array | 6 | |
| `monteCarloSaves` | array | 3 | |
| `tuning` | object | — | **Not a Dexie table.** Calibrated engine parameters (§8). |

**Runtime sync separation:** physical component records carry only physical/geometry fields; all
*calibrated* fields (`burnAreaCoeff`, `transducerScaleFactor`, etc.) live under `tuning`. On load,
`syncFromMasterData` `Object.assign`s `tuning.powders[i]` onto the matching powder and
`tuning.cartridges[i]` onto the matching cartridge (by `id`), so the rest of the app reads e.g.
`powder.burnAreaCoeff` directly from IndexedDB. The full `tuning` blob (including `velocityCorrection`,
`densityRefs`, `levelRefs`, `generatedAt`) is stored verbatim in `meta['local-db'].tuning`.

---

## 7. Component Table Reference (shared shapes)

### 7.1 `manufacturers`. Index: `id`

| Field | Type | Meaning |
|---|---|---|
| `id` | string | PK (`MAN_...`). |
| `name` | string | Brand name. |
| `displayName`? | string | Optional shorthand (interface field). |
| `type`? | string[] | Category tags. Observed values: `"bullet"`, `"powder"`, `"primer"`, `"brass"`, `"ammo"`. |
| `country`? | string | ISO country code (e.g. `"US"`). Present in data; not in the TS interface. |

### 7.2 `diameters`. Index: `id`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`DIA_...`). |
| `metricName` | string | — | Metric display string (e.g. `"7.62mm"`). |
| `imperialName`? | string | — | Imperial display string (e.g. `".308"`). |
| `bulletDiameterMm` | number | mm | Nominal groove/bullet diameter. |
| `boreDiameterMm` | number | mm | Land-to-land bore diameter. |

### 7.3 `grainTypes`. Index: `id`

| Field | Type | Meaning |
|---|---|---|
| `id` | string | Semantic id: `ball`, `flake`, `extrudedSinglePerf`, `extrudedMultiPerf`, `extruded`. Referenced by `Powder.grainType`. |
| `name` | string | Human label (e.g. `"Spherical / Ball"`). |

### 7.3b `geometryProvenanceTiers` (reference-DB only; not a Dexie table)

Ranked confidence tiers for bullet geometry, referenced by `Bullet.geometryTierId`. Lives only
in `local-db.json`; the app's sync ignores it (`parseImportBundle` skips non-Dexie keys).

| Field | Type | Meaning |
|---|---|---|
| `id` | string | `GEOTIER_MEASURED` (rank 1) → `GEOTIER_WEB_VERIFIED` (2) → `GEOTIER_WEB_BESTFIT` (3) → `GEOTIER_SPLIT` (4) → `GEOTIER_SYNTHESIZED` (5) → `GEOTIER_ESTIMATED` (6). |
| `rank` | number | Lower = higher confidence; enables "all bullets below tier X" queries. |
| `name` / `description` | string | Human label and definition (SPLIT = known OAL, donor-scaled ogive/bearing split). |

### 7.4 `primerPockets`. Index: `id`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK. Fixed set: `PKT_SML`, `PKT_LRG`. |
| `name` | string | — | `"SMALL"` / `"LARGE"`. |
| `pocketDiameterMinMm`? / `pocketDiameterMaxMm`? | number | mm | SAAMI/CIP reamer diameter tolerance. |
| `pocketDepthMinMm`? / `pocketDepthMaxMm`? | number | mm | SAAMI/CIP pocket depth tolerance. |

### 7.5 `primers`. Index: `id, manufacturerId`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`PRI_...`). |
| `manufacturerId` | string | — | FK → `manufacturers.id`. |
| `name` | string | — | Model name. |
| `type`? | string | — | Size/type description (interface field; not populated in shipped data). |
| `primerPocketId`? | string | — | FK → `primerPockets.id`. |
| `brisanceEnergyJ`? | number | Joules | Ignition spark energy. Typical: small rifle ~8, large rifle ~14, magnum ~20. |

### 7.6 `cartridges`. Index: `id, diameterId`
All lengths/diameters in **mm**, pressures in **Pa**, capacity in **grams H₂O**, angles in **degrees**.

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`CTG_...`). |
| `name` | string | — | Cartridge name. |
| `diameterId` | string | — | FK → `diameters.id`. |
| `primerPocketId`? | string | — | FK → `primerPockets.id`. |
| `maxCaseLengthMm`? | number | mm | Max case length. |
| `trimLengthMm`? | number | mm | Trim-to length. |
| `oalMm`? | number | mm | Nominal cartridge overall length (SAAMI). |
| `maxSaamiPa`? | number | Pa | SAAMI max average pressure. **Authoritative engine ceiling.** |
| `maxCipPa`? | number | Pa | C.I.P. max pressure — a *different standard*, not a unit variant of `maxSaamiPa`; both kept. |
| `baseCapacityH2oGrams`? | number | g H₂O | Case water capacity at the base. |
| `boreDiameterMm`? | number | mm | Land-to-land bore diameter. |
| `bulletDiameterMm`? | number | mm | Nominal groove/bullet diameter. |
| `flashHoleDiameterMm`? | number | mm | SAAMI/CIP flash hole diameter. |
| `shoulderAngleDeg`? | number | deg | SAAMI/CIP shoulder half-angle. |
| `bodyDiameterMm`? | number | mm | External body diameter at base (P1). |
| `freeboreLengthMm`? | number | mm | The single cartridge-side freebore source; maps to engine `throatFreetravelMm`. |
| `twistRateMm`? | number | mm/turn | The live twist field the engine reads (1:8" = 203.2). |
| `cipTestBarrelMm`? | number | mm | CIP regulatory test barrel length (600/650); sparse (~33 cartridges). |
| `refTestBarrelMm`? | number | mm | Reference test barrel assumed when a load source omits the barrel (omission ⇒ standard barrel used). Derived per cartridge from cited barrels; replaces the ingester's old blunt 24" default. See `scratch/populate_ref_barrel.py`. |
| `refTestBarrelSource`? | string | — | Provenance of `refTestBarrelMm` (publisher-level vote): `EMPIRICAL_MODE` (≥2/3 supermajority) \| `EMPIRICAL_LOW` (plurality; genuinely multi-barrel) \| `EMPIRICAL_SINGLE` (one publisher) \| `CIP` \| `DEFAULT_24`. |
| `refTestBarrelAgreement`? | number | — | Confidence of `refTestBarrelMm` = winning votes / total publisher votes (0–1). |
| `externalDimensions`? | object | mm | Case geometry for the volume solver (below). |
| `wallThicknessProfile`? | object | mm | Wall-thickness profile (below). |

`externalDimensions` (all 7 required when the object is present, all **mm**):
`baseDiameterP1Mm`, `bodyDiameterAtShoulderP2Mm`, `shoulderNeckJunctionDiameterH1Mm`,
`neckDiameterAtMouthH2Mm`, `lengthToShoulderStartL1Mm`, `lengthToShoulderNeckJunctionL2Mm`,
`totalCaseLengthL3Mm`.

`wallThicknessProfile` (**mm**): `webThicknessMm`, `wallThicknessAtBaseMm`, `wallThicknessAtMidBodyMm`?,
`wallThicknessAtShoulderMm`, `wallThicknessAtNeckMm` (mid-body optional).

### 7.7 `bullets`. Index: `id, manufacturerId, diameterId`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`BUL_...`). |
| `manufacturerId` | string | — | FK → `manufacturers.id`. |
| `diameterId` | string | — | FK → `diameters.id`. |
| `name` | string | — | Model name. |
| `advertisedWeightGrains`? | number | **grains** | Labeled weight (display; present in data, not in TS interface). |
| `physis` | object | — | Physical geometry (below). |
| `ballistics` | object | — | BC / form factor (below). |

`physis`:

| Field | Type | Units | Meaning |
|---|---|---|---|
| `weightGrams` | number | g | Bullet mass (metric; grains ÷ 15.4324). |
| `overallLengthMm` | number \| null | mm | Total length. |
| `ogiveLengthMm` | number \| null | mm | Ogive (nose) length. **Nullable.** |
| `boatTailLengthMm` | number \| null | mm | Boat-tail length. **Nullable.** |
| `tipLengthMm` | number \| null | mm | Plastic tip length (excluded from aerodynamic metal length). |
| `meplatDiameterMm` | number \| null | mm | Meplat (nose flat) diameter. |
| `bearingSurfaceMm`? | number \| null | mm | Bearing surface length. **Nullable.** |
| `materialType`? | enum string | — | `MAT_JACKETED_LEAD`, `MAT_MONOLITHIC_COPPER`, `MAT_CAST_LEAD`, `MAT_RELIEF_GROOVED_COPPER_MONO` (legacy bare `jacketed_lead`/`monolithic_copper`/`cast_lead` deprecated). |
| `engravingPressurePa`? | number | Pa | Engraving-resistance pressure for the internal ballistics free-travel model (typical 32e6 = 32 MPa). |
| `verificationFlag`? | string | — | Free-text QA note flagging suspect/phantom records. In data, not in TS interface. |

Geometry provenance (bullet-level, **outside** `physis` — it is metadata, not a dimension;
replaced the old free-text `physis.geometryProvenance` on 2026-07-24):

| Field | Type | Meaning |
|---|---|---|
| `geometryTierId` | string | FK → `geometryProvenanceTiers.id`. Confidence tier of the geometry data. |
| `geometryDonorBulletId`? | string | FK → `bullets.id`. Only on `GEOTIER_SPLIT` records: the (measured) bullet whose ogive/bearing proportions were scaled to this bullet's known OAL. Never a split-tier record itself. |
| `geometryNote`? | string | Optional detail tail (sources, corroboration, or `donor(unresolved, deleted upstream): <old id>` where a donor record no longer exists). |

`ballistics`:

| Field | Type | Meaning |
|---|---|---|
| `g1BC` | number \| null | G1 ballistic coefficient. |
| `g7BC` | number \| null | G7 ballistic coefficient. |
| `g1FF`? | number \| null | G1 form factor. |
| `g7FF`? | number \| null | G7 form factor. |
| `sectionalDensity`? | number | Sectional density (present in data, not in TS interface). |
| `preferredModel`? | `'G1'\|'G7'` | Interface field; not populated in shipped data. |

### 7.8 `powders`. Index: `id, manufacturerId`
Root holds **physical constants only**. Calibrated fields (`burnAreaCoeff`, slopes, `energyScaleFactor`,
and calibrated `burnExponent`) live in `tuning.powders[]` and are merged on at runtime — **never store
them on the powder root.**

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`PWD_...`). |
| `manufacturerId` | string | — | FK → `manufacturers.id`. |
| `name` | string | — | Powder name. |
| `heatOfExplosionKjKg`? | number | kJ/kg | Heat of explosion. Physical — do not calibrate. Single-base ~3580–3750; double-base ~3950. |
| `heatConvention`? | `'vapor'\|'liquid'` | — | Water-state convention for the heat value. |
| `kCoeff`? | number | — | Noble-Abel adiabatic exponent ratio (~1.23 single-base, ~1.24–1.255 double-base). Physical. |
| `grainType`? | enum string | — | FK → `grainTypes.id` (`ball`/`flake`/`extrudedSinglePerf`/`extrudedMultiPerf`/`extruded`). |
| `propellantDensityKgM3`? | number | kg/m³ | Solid propellant density (Noble-Abel EOS). Physical. |
| `bulkDensityKgM3`? | number | kg/m³ | Poured bulk density (for fill %). Physical. |
| `burnExponent`? | number | — | Vieille's-law pressure exponent (default 0.65; typical 0.55–0.85). **Calibrated** — canonical value is in `tuning.powders[]`. |
| `tempSensitivity`? | number | /°C | Temperature sensitivity coefficient. |
| `ignitionBp`? / `ignitionZ1`? / `ignitionZ2`? | number | — | Multi-stage ignition/burn profile parameters. |
| `ignitionProvenance`? | string | — | Source tag for ignition params (e.g. `grt-curvefit-2021-03-17`). In data, not in TS interface. |
| `burnAreaCoeff`? | number | — | **Calibrated** (canonical in `tuning`); interface allows it on root only as the merge target. |
| `cartridgeOverrides`? | `{cartridgeId,burnAreaCoeff}[]` | — | Per-cartridge `burnAreaCoeff` overrides fitted by `--calibrate-cartridge-overrides`. |

### 7.9 `loads`. Index: `id, cartridgeId, bulletId, powderId`
**All mass/length metric** (grams/mm), velocity in m/s.

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`LOAD_`/`LOD_...`). |
| `name`? | string | — | Display name (factory/commercial loads). |
| `handloadName`? | string | — | Handload nickname. |
| `cartridgeId` | string | — | FK → `cartridges.id` (required). |
| `bulletId` | string | — | FK → `bullets.id` (required). |
| `powderId`? | string | — | FK → `powders.id`. |
| `primerId`? | string | — | FK → `primers.id`. |
| `brassId`? | string | — | FK → `brass.id`. |
| `manufacturerId`? | string | — | FK → `manufacturers.id` (factory ammo). |
| `partNumber`? | string \| null | — | SKU. |
| `lot`? | string \| null | — | Lot number. |
| `chargeWeightGrams`? | number | g | Propellant charge (grains × 0.0647989). |
| `coalMm`? | number | mm | Cartridge overall length. |
| `cbtoMm`? | number | mm | Cartridge base-to-ogive length. |
| `velocityMps`? | number | m/s | Recorded average MV (fps × 0.3048). |
| `loadTypeId`? | string | — | *Deprecated — not populated.* |
| `isCommercial`? | boolean | — | *Deprecated — not populated.* |
| `notes`? | string | — | Free text. |
| `bulletLot`? | string | — | Bullet lot number (handload bookkeeping; `lot` is its legacy fallback on read). |
| `powderLot`? | string | — | Powder lot number. |
| `primerLot`? | string | — | Primer lot number. |
| `brassLot`? | string | — | Brass lot number. |
| `cbtoComp`? | string | — | Comparator/tool used for the `cbtoMm` measurement. |
| `shoulderMm`? | number | mm | Shoulder bump (stored from an inches input). |
| `shoulderComp`? | string | — | Comparator/tool used for the shoulder measurement. |
| `firings`? | number | — | Times the brass in this load has been fired. |

### 7.10 `brass`. Index: `id, cartridgeId, manufacturerId`

| Field | Type | Units | Meaning |
|---|---|---|---|
| `id` | string | — | PK (`BRS_...`). |
| `manufacturerId` | string | — | FK → `manufacturers.id`. |
| `cartridgeId` | string | — | FK → `cartridges.id`. |
| `primerPocketId`? | string | — | FK → `primerPockets.id`. |
| `primerHole`? | number \| null | mm | Flash hole diameter (interface field; not populated). |
| `capacityH2oGrams`? | number \| null | g H₂O | Case overflow water capacity. |

---

## 8. The `tuning` Blob

`local-db.json.tuning` is an object (not a Dexie table) stored verbatim in `meta['local-db'].tuning`.
Top-level keys:

| Key | Type | Meaning |
|---|---|---|
| `powders` | array | Per-powder calibrated parameters (merged onto powder records). |
| `cartridges` | array | Per-cartridge calibrated scalars (merged onto cartridge records). |
| `powderCartridgeCalibrations` | array | Reserved for per-powder-per-cartridge overrides; currently empty `[]`. |
| `velocityCorrection` | object | Post-integration velocity correction model. |
| `densityRefs` | object (map) | `"<CTG_id>|<PWD_id>" → number` reference loading density (kg/m³) per cartridge+powder pair. |
| `levelRefs` | object (map) | `"<CTG_id>|<PWD_id>" → number` reference fill level per cartridge+powder pair. |
| `cellDataProvenance` | object (map) | `"<CTG_id>|<PWD_id>" → {pub, share, n}` dominant publisher + share per cell (calibrateV4 post-pass). App surfaces share ≥ 0.9 as a single-test-rifle velocity caution — transparency, never silent correction. |
| `bulletPressureFactors` | object (map) | `"<BUL_id>" → number` per-bullet pressure REPORTING factor (calibrateV4 post-pass): TSF-analogue on the bullet axis, EB-shrunken, \|t\|≥2-guarded, per-bullet p30 tail-guard, clamped [0.90, 1.08]. Applied via `inputs.bulletPressureFactor` at consumption boundaries only (run_all, app); never during fitting. |
| `generatedAt` | string | ISO 8601 timestamp of the calibration run (also used as the tuning stamp on firearm true-ing). |

### 8.1 `tuning.powders[]`
Merged by `id` onto the matching powder.

| Field | Type | Meaning |
|---|---|---|
| `id` | string | FK → `powders.id`. |
| `burnAreaCoeff` | number \| null | Base burn-rate coefficient at the reference fill fraction. `null` for unmodeled powders. |
| `burnAreaFillSlope` | number | Linear fill-fraction slope (0 when insufficient data). |
| `burnAreaBoreSlope` | number | Bore-diameter correction slope (0 at reference bore). |
| `burnAreaExpansionSlope` | number | Gas-expansion slope during barrel travel (default 0). |
| `energyScaleFactor` | number | Engine energy-efficiency multiplier (not physical; persistently >1.25 ⇒ suspect data). |
| `burnExponent` | number | Calibrated Vieille's-law exponent (canonical copy; also mirrored to the powder root). |

### 8.2 `tuning.cartridges[]`
Merged by `id` onto the matching cartridge.

| Field | Type | Meaning |
|---|---|---|
| `id` | string | FK → `cartridges.id`. |
| `transducerScaleFactor` | number | Piezo→SAAMI-equivalent breech-pressure scaling. Default 1.0; fitted per cartridge from published pressure. |
| `gradientBetaScale`? | number | Beta scale for the pressure-gradient ODE (default 1.0; populated only when the transducer scale alone fits poorly). |

### 8.3 `tuning.velocityCorrection`
Spline + per-cartridge/per-powder correction applied after ODE integration.

| Field | Type | Meaning |
|---|---|---|
| `modelVersion` | string | Schema tag (e.g. `"v4.5-pressure-ramp"`). |
| `globalKnots` | `{expansionRatio,factor}[]` | Piecewise-linear global curve. `expansionRatio` = barrel volume / chamber volume; `factor` = velocity multiplier. |
| `pressureRampSlope` | number | Global pressure-ramp slope term. |
| `cartridgeOverrides` | object (map) | `cartridgeId → override` (below). |
| `rifleEffects` | object (map) | `"<Manufacturer>|<CTG_id>" → {factor, devPct, n}` — per powder-brand×cartridge velocity bias (`factor` multiplier, `devPct` % deviation, `n` sample count). |
| `fittedAt` | string | ISO 8601 fit timestamp. |
| `validationR2`? | number | Hold-out R² (when present). |

`cartridgeOverrides[cartridgeId]`:

| Field | Type | Meaning |
|---|---|---|
| `factor` | number | Scalar velocity multiplier for the cartridge. |
| `confidence` | string | `"HIGH"`/`"MEDIUM"`/`"LOW"`. |
| `weightSlope`? | number | Charge-weight sensitivity slope. |
| `refWeightGrams`? | number | Reference charge weight (g) for the slope. |
| `weightFactorMin`? / `weightFactorMax`? | number | Clamp bounds for the weight-adjusted factor. |
| `meanSimPressurePsi`? / `stdSimPressurePsi`? | number | Fit diagnostics (sim pressure mean/SD in PSI). |
| `fittedLoads`? | number | Load count used to fit. |
| `powderOverrides`? | object (map) | `powderId → { factor, meanSimPressurePsi, stdSimPressurePsi, weightSlope?, refWeightGrams?, weightFactorMin?, weightFactorMax? }` — per-powder refinement within the cartridge. |

---

## 9. Worked Examples

### 9.1 Add a handload (write path)

```ts
import { db } from '../db/database';

const id = 'LOAD_' + crypto.randomUUID().replace(/-/g, '').slice(0, 16);
await db.loads.put({
  id,
  handloadName: '147gr ELD-M / H4350',
  cartridgeId: 'CTG_65PRC_cP02',   // FK → cartridges.id (required)
  bulletId:    'BUL_HORNADY_264_147_147_ELDM', // FK → bullets.id (required)
  powderId:    'PWD_HODG_H4350_H1G2',
  primerId:    'PRI_FED_210M_F1D3',
  brassId:     'BRS_MAN_ALPHA_A1L2_CTG_65PRC_cP02_LRP_OCD',
  chargeWeightGrams: 44.8 * 0.0647989,  // grains → grams at the boundary
  coalMm:            75.06,
  velocityMps:       2910 * 0.3048,     // fps → m/s at the boundary
  notes: '44.8gr H4350, 2910 fps avg',
});
```
Rules: store metric; supply all required FKs; the id starts with `LOAD_` so a later master sync could prune
it — user loads normally use non-prefixed ids from `generateUniqueId()` to avoid that.

### 9.2 Read a session's shots + velocities (read path)

```ts
const session = await db.sessions.get(sessionId);
if (!session?.markedTargetId) return [];      // no marking yet

// §4.4: groups/shots are keyed by markedTargetId, stored in the field named `sessionId`.
const mtId = session.markedTargetId;
const shots = await db.shots.where('sessionId').equals(mtId).sortBy('shotNumber');

const velocities = shots
  .filter(s => s.velocity != null)            // fps, may be null
  .map(s => s.velocity as number);
```
Querying `db.shots.where('sessionId').equals(session.id)` returns nothing — that is the classic bug this
quirk causes.

### 9.3 Read a powder with its calibrated burn rate

After `syncFromMasterData`, `tuning.powders[i]` has been `Object.assign`ed onto the powder record, so:
```ts
const powder = await db.powders.get('PWD_HODG_VARGET_H1G1');
const coeff  = powder?.burnAreaCoeff;   // merged from tuning at sync time
// Full tuning (velocityCorrection, densityRefs, generatedAt) if needed:
const meta = await db.meta.get('local-db');
const vc   = meta?.tuning?.velocityCorrection;
```

---

*Companion docs: `MATHS.md` (engine derivations), `README.md` (user guide). This file supersedes the prior
schema draft and is generated from `database.ts` + `local-db.json` structure.*
