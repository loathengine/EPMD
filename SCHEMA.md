# Empirical Precision — Reference Database Schema

The shape of **`test_harnes/json/local-db.json`**, the curated component library EPv5 ships
and syncs into the browser. Detailed enough to write a reader or a generator against without
opening the file.

**Scope: reference data only.** `local-db.json` also carries a working copy of user tables —
`firearms`, `loads`, `sessions`, `sessionTargets`, `groups`, `shots`, `markedTargets`,
`targetImages`, `customTargets`, `monteCarloSaves`. Those are the maintainer's own range
records, they are not part of the shipped library, and they are **deliberately not documented
here.** A consumer of the reference database should ignore them.

**Sources of truth, in priority order:**
1. `test_harnes/json/local-db.json` — the data itself. Field lists, types and coverage
   percentages below were extracted from all 1,726 reference records, not transcribed.
2. `src/types/database.ts` — the TypeScript interfaces.
3. `src/db/database.ts` — the Dexie class `ReloaderDB` (`reloadingDB_v5`) and
   `syncFromMasterData`, which is what reads this file at runtime.

Where the interface and the data disagree, **this document states both**, because that gap is
what breaks a consumer written from the types alone. §5 lists every instance.

**Verified against those sources: 2026-08-05.**

---

## 1. What is in the file

| Table | Records | Holds |
|---|---:|---|
| `manufacturers` | 38 | Makers of bullets, powder, primers, brass |
| `diameters` | 9 | Caliber definitions — bore and bullet diameter |
| `grainTypes` | 5 | Propellant kernel geometries |
| `geometryProvenanceTiers` | 6 | How trustworthy a bullet's measurements are |
| `primerPockets` | 2 | Small / large pocket dimensions |
| `primers` | 14 | Primer inventory, with brisance energy |
| `cartridges` | 124 | Chamberings: case dimensions and pressure ceilings |
| `bullets` | 993 | Projectiles: geometry, material, G1/G7 BC |
| `powders` | 144 | Propellants: densities, burn parameters, kernel type |
| `brass` | 391 | Cases: capacity and pocket size, per cartridge and maker |

**There is no `tuning` blob.** EPv4 carried one here and merged it onto powder and cartridge
records at sync time. EPv5's engine resolves its fitted parameters by component id from
`public/tuning_fit.json` instead, and `syncFromMasterData` ignores any `tuning` section it
finds (WS6). Fitted values must not be written back into this file — that would create the
second source of truth CLAUDE.md §5.1 forbids.

## 2. Units

Every stored value is **SI/metric**, without exception. Grains, inches and fps exist only in
the UI. The suffix on a field name is its unit and is not decorative:

| Suffix | Unit | Suffix | Unit |
|---|---|---|---|
| `…Mm` | millimetres | `…Pa` | pascals |
| `…Grams` | grams | `…KgM3` | kg/m³ |
| `…J` | joules | `…KjKg` | kJ/kg |

`capacityH2oGrams` and `baseCapacityH2oGrams` are **grams of water**, not grains — a 6.5
Creedmoor case is ~3.3 g, which is ~51 gr H₂O.

## 3. IDs

Every record has a string `id`, and the prefix identifies the table. `syncFromMasterData`
uses these prefixes to prune stale master records: on sync it deletes any local record whose
id carries the prefix but is absent from the incoming file, so **an id is a permanent
contract.** Renaming one deletes the old record and creates a new one, orphaning every
reference to it.

| Prefix | Table | Example |
|---|---|---|
| `MAN_` | manufacturers | `MAN_BERGER_B9K2` |
| `DIA_` | diameters | `DIA_308_XBM7` |
| `CTG_` | cartridges | `CTG_65CM_cN72` |
| `BUL_` | bullets | `BUL_BERGER_308_168_VLD_VOMA` |
| `PWD_` | powders | `PWD_ACCU_2230_R1A6` |
| `PRI_` | primers | `PRI_CCI_BR4_C1R6` |
| `PKT_` | primerPockets | `PKT_SML`, `PKT_LRG` |
| `BRS_` | brass | `BRS_MAN_ADG_A4D7_CTG_65CM_cN72` |
| `GEOTIER_` | geometryProvenanceTiers | `GEOTIER_MEASURED` |
| *(none)* | grainTypes | `ball`, `flake`, `extrudedSinglePerf` |

`grainTypes` is the one table with bare, meaningful ids, because `Powder.grainType` stores
the string directly rather than a foreign key.

## 4. How the tables relate

```
manufacturers ─┬─< bullets >──── diameters
               ├─< powders
               ├─< primers >──── primerPockets
               └─< brass >──┬─── cartridges >─── diameters
                            └─── primerPockets

bullets >─── geometryProvenanceTiers      (geometryTierId)
bullets >─── bullets                      (geometryDonorBulletId — geometry borrowed from another bullet)
powders >─── grainTypes                   (grainType, by bare id)
cartridges >─ primerPockets
```

Every `…Id` field is a foreign key to the `id` of the named table. Nothing in the reference
data is nullable-by-reference: where an id field is present it always resolves.

---

## 5. Where the types and the data disagree

Read this section before writing anything against `src/types/database.ts`. Each row is a
field the interface declares that **no record in the shipped file carries**. A consumer that
assumes they exist will read `undefined`.

| Interface | Field declared but absent | Why it is absent |
|---|---|---|
| `Cartridge` | `bulletDiameterMm`, `boreDiameterMm` | Both live on the **`diameters`** record instead. Resolve via `cartridge.diameterId`. `IgnitionSimulator` reads the cartridge field first and falls back to the diameter record, which is why nothing has broken. |
| `Cartridge` | `transducerScaleFactor`, `gradientBetaScale` | EPv4 calibration outputs. Dead under ib2 — these are `tuning_fit.json`'s concern now. |
| `Powder` | `burnAreaCoeff`, `burnAreaFillSlope`, `burnAreaBoreSlope`, `burnAreaExpansionSlope`, `energyScaleFactor` | Same: EPv4 fitted values, superseded by `tuning_fit.json` keyed on powder id. **Do not repopulate them.** |
| `Powder` | `cartridgeOverrides` | EPv4's per-cartridge burn-rate override list. ib2 fits a (cartridge, powder) pair factor instead. |
| `Powder` | `heatConvention` | Never populated; every record's `heatOfExplosionKjKg` is on one convention. |
| `Bullet.physis` | `meplatDiameterMm` | Typed `number \| null` and carried by no record. The engine does not read it. |
| `Brass` | `primerHole` | Flash-hole diameter. Typed, never populated. |
| `Primer` | `type` | Superseded by `primerPocketId`, which every record does carry. |
| `Manufacturer` | `displayName` | The UI falls back to `name`, which is why the fallback is written `displayName || name` everywhere. |

And one gap the other way:

| Table | Problem |
|---|---|
| `geometryProvenanceTiers` | **Has no TypeScript interface.** Six records with `id`, `name`, `rank`, `description`, referenced by `Bullet.geometryTierId` — which *is* typed, as a bare `string`. A reader must treat the tier table as untyped reference data. |

`diameters`, `grainTypes`, `primerPockets` and `bullets` match their interfaces exactly.

---

## 6. Table reference

Coverage is the share of records carrying a non-null value. **100% means required in
practice**, whatever the interface says is optional.

### 6.1 `manufacturers` — 38 records

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `MAN_` |
| `name` | string | 100% | `Berger`, `Sierra`, `Hodgdon` |
| `type` | string[] | 94.7% | What they make. Two records omit it entirely. |

### 6.2 `diameters` — 9 records

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `DIA_` |
| `metricName` | string | 100% | `5.56`, `6mm`, `7.62` |
| `imperialName` | string | 100% | `.224`, `.243`, `.308` — typed optional, always present |
| `bulletDiameterMm` | number | 100% | Groove/bullet diameter — what the projectile measures |
| `boreDiameterMm` | number | 100% | Land-to-land — always smaller than `bulletDiameterMm` |

Both diameters matter and are not interchangeable: bore diameter sets the area pressure acts
on, bullet diameter sets what is engraved.

### 6.3 `grainTypes` — 5 records

| Field | Type | Coverage |
|---|---|---:|
| `id` | string | 100% |
| `name` | string | 100% |

Ids are bare and meaningful: `ball`, `flake`, `extrudedSinglePerf`, `extrudedMultiPerf`,
`extruded`. `Powder.grainType` holds one of these strings directly.

### 6.4 `geometryProvenanceTiers` — 6 records

How much a bullet's stored geometry can be trusted. **No TypeScript interface — see §5.**

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `GEOTIER_MEASURED`, `GEOTIER_WEB_VERIFIED`, … |
| `name` | string | 100% | `Measured`, `Web-verified` |
| `rank` | number | 100% | 1 = best. Sort on this, not on the id. |
| `description` | string | 100% | What the tier means |

### 6.5 `primerPockets` — 2 records

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `PKT_SML`, `PKT_LRG` |
| `name` | string | 100% | `SMALL`, `LARGE` |
| `pocketDiameterMinMm` | number | 100% | SAAMI tolerance band |
| `pocketDiameterMaxMm` | number | 100% | |
| `pocketDepthMinMm` | number | 100% | SAAMI tolerance band |
| `pocketDepthMaxMm` | number | 100% | |

All four dimension fields are typed optional and are always present.

### 6.6 `primers` — 14 records

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `PRI_` |
| `manufacturerId` | string | 100% | → `manufacturers` |
| `name` | string | 100% | `BR-4 Small Rifle Benchrest` |
| `primerPocketId` | string | 100% | → `primerPockets`. Filters which primers fit a case. |
| `brisanceEnergyJ` | number | 92.9% | Ignition energy in joules. **One record lacks it**; the simulator falls back to a pocket-size default and says so on screen rather than hiding the substitution. |

### 6.7 `cartridges` — 124 records

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `CTG_` |
| `name` | string | 100% | `6.5 Creedmoor` |
| `diameterId` | string | 100% | → `diameters`. **The only route to bullet/bore diameter** — see §5. |
| `primerPocketId` | string | 100% | → `primerPockets`. The cartridge's SAAMI default; specific brass may differ. |
| `baseCapacityH2oGrams` | number | 100% | Nominal case capacity, grams of water. Used when no brass is selected. |
| `maxCaseLengthMm` | number | 100% | SAAMI maximum |
| `trimLengthMm` | number | 100% | Trim-to length; the app prefers this over `maxCaseLengthMm` |
| `oalMm` | number | 100% | SAAMI maximum cartridge overall length |
| `bodyDiameterMm` | number | 100% | External body diameter at the base (P1) |
| `freeboreLengthMm` | number | 100% | SAAMI freebore. A measured firearm value overrides it. |
| `twistRateMm` | number | 100% | Reference twist, mm/turn (1:8″ = 203.2) |
| `refTestBarrelMm` | number | 100% | Barrel length the reference velocity was measured in |
| `maxSaamiPa` | number | 100% | **The pressure ceiling the Ignition safety audit measures against** |
| `maxCipPa` | number | 95.2% | CIP ceiling. Absent on the 6 cartridges CIP does not list. |
| `cipTestBarrelMm` | number | 26.6% | CIP test-barrel length. Present only where CIP publishes one. |

### 6.8 `bullets` — 993 records

The largest table, and the one whose geometry drives stability, seating and case fill.

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `BUL_` |
| `manufacturerId` | string | 100% | → `manufacturers` |
| `diameterId` | string | 100% | → `diameters` |
| `name` | string | 100% | `VLD Hunting`, `Tipped TSX FB` |
| `physis` | object | 100% | Physical geometry — below |
| `ballistics` | object | 100% | Drag — below |
| `geometryTierId` | string | 99.7% | → `geometryProvenanceTiers`. **3 records have none.** |
| `geometryNote` | string | 18.2% | How the geometry was arrived at: `bc-oal-pass`, `web+grt-verified` |
| `geometryDonorBulletId` | string | 4.2% | → `bullets`. This bullet's geometry was **borrowed** from another. Treat those dimensions as an estimate. |

**`physis` — present on every record:**

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `weightGrams` | number | 100% | Never null |
| `overallLengthMm` | number \| null | key 100%, value 99.7% | Tip to base |
| `ogiveLengthMm` | number \| null | key 100%, value 99.7% | |
| `boatTailLengthMm` | number \| null | key 100%, value 99.7% | `0` for a flat base — distinct from `null` for unknown |
| `bearingSurfaceMm` | number \| null | key 100%, value 99.7% | Full-diameter shank. When null the engine defaults to **half the overall length**, and its correction factors were fitted with that default in place — see WS9 before "improving" it. |
| `tipLengthMm` | number | 16.6% | Polymer tip only. Absent means no tip. |
| `materialType` | string | 100% | `MAT_JACKETED_LEAD`, `MAT_MONOLITHIC_COPPER`, `MAT_CAST_LEAD`, `MAT_RELIEF_GROOVED_COPPER_MONO`. The interface also permits three legacy lowercase spellings; no record uses them. |
| `engravingPressurePa` | number | 100% | Pressure needed to engrave the rifling |

The four `number \| null` fields share one trap: the **key is always there**, the value may be
`null`. `if ('overallLengthMm' in physis)` is true for the 3 records that know nothing. Those
same 3 records are the ones missing `geometryTierId` — a bullet with no provenance has no
geometry either.

**`ballistics` — present on every record:**

| Field | Key present | Non-null | Notes |
|---|---:|---:|---|
| `g1BC` | 87.3% | **87.2%** | Flat-base / round-nose reference |
| `g7BC` | 72.7% | **44.3%** | Boat-tail reference — **prefer this** for modern match bullets |

**Read those two columns separately — this is the sharpest trap in the file.** Unlike
`physis`, the `ballistics` keys are *not* always present: `g7BC` is missing outright on 27%
of records and explicitly `null` on another 28%, so **only 44% of bullets have a usable G7
number.** A `'g7BC' in ballistics` test passes for 722 records but yields a value for 440.

**127 bullets (12.8%) have neither BC** and cannot be used for trajectory work at all. A
consumer must handle that case rather than assume a fallback chain always terminates.

### 6.9 `powders` — 144 records

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `PWD_` |
| `manufacturerId` | string | 100% | → `manufacturers` |
| `name` | string | 100% | `H4350`, `Varget`, `2230` |
| `grainType` | string | 100% | One of the `grainTypes` ids, stored directly |
| `propellantDensityKgM3` | number | 100% | Density of the solid propellant, ~1540–1650 |
| `bulkDensityKgM3` | number | 100% | Poured density including air — what decides whether a charge fits |
| `heatOfExplosionKjKg` | number | 100% | Energy per kg, ~3500–4000 |
| `burnExponent` | number | 99.3% | Vieille's-law pressure exponent. **One record lacks it**; the engine's default is 0.65. |
| `burnSurfacePeakGain` | number | 94.4% | Burning-surface evolution — how much area grows before it starts shrinking |
| `burnFractionAtSurfacePeak` | number | 94.4% | Burn fraction at that peak |
| `burnFractionAtSliverStart` | number | 94.4% | Burn fraction where the kernel breaks into slivers |
| — | | | **No burn-rate coefficient.** It is a fitted parameter and lives in `tuning_fit.json`, keyed by this `id`. See §5. |

The three `burnFraction…` / `burnSurface…` fields travel together: 8 records have none of the
three, 136 have all three. Treat them as one optional block.

### 6.10 `brass` — 391 records

One record per (manufacturer, cartridge) pair — the id encodes both.

| Field | Type | Coverage | Notes |
|---|---|---:|---|
| `id` | string | 100% | `BRS_{manufacturerId}_{cartridgeId}` |
| `manufacturerId` | string | 100% | → `manufacturers` |
| `cartridgeId` | string | 100% | → `cartridges` |
| `primerPocketId` | string | 100% | → `primerPockets`. **May differ from the cartridge's default** — this is how small-primer .308 and 6.5 CM brass is represented, and why selecting brass can invalidate a primer choice. |
| `capacityH2oGrams` | number | **33.5%** | Measured water capacity. **Two-thirds of records do not have it.** |

`capacityH2oGrams` is the lowest-coverage field in the reference database and one of the most
consequential — case capacity is a first-order pressure input. Absence means *unknown*, never
*zero* (WS36): a consumer must fall back to the cartridge's `baseCapacityH2oGrams` rather than
treat a missing value as an empty case.

---

## 7. How the app reads this file

`syncFromMasterData(data)` in `src/db/database.ts`:

1. `parseImportBundle` picks out the recognised table arrays. Anything else — including a
   `tuning` section left over from EPv4 — is ignored.
2. `repairImportData` normalises known legacy shapes.
3. For each table with an id prefix (§3), any **local** record whose id starts with that
   prefix but is missing from the incoming file is deleted. This is the pruning path, and it
   is why ids are a permanent contract.
4. Remaining records are `put` by id, so a user's own records — which carry no master prefix
   — survive untouched.

The Dexie database is **`reloadingDB_v5`**, currently at `version(2)`. The reference tables
are indexed as:

```
manufacturers  id
diameters      id
grainTypes     id
primerPockets  id
primers        id, manufacturerId
cartridges     id, diameterId
bullets        id, manufacturerId, diameterId
powders        id, manufacturerId
brass          id, cartridgeId, manufacturerId
```

`geometryProvenanceTiers` is **not a Dexie table.** It exists only in this file, as a lookup
for `Bullet.geometryTierId`.
