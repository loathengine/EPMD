# Empirical Precision — Database Schema Reference

This document is the canonical technical reference for all IndexedDB tables and the `local-db.json` / `master-db.json` structure.

**Convention:** All field names use **camelCase**. All IDs are strings. Optional fields are marked with `?`.

Tables marked **[master-db]** are seeded from the open-source library. Tables marked **[user-only]** are created locally and never appear in `master-db.json`.

---

## ID Naming Conventions

### Rules for All Master-DB IDs

IDs in `local-db.json` and `master-db.json` follow a strict structured compound key pattern. **The ID is the source of truth — never the name.** IDs must never change after creation.

#### General Rules

| Rule | Description |
|------|-------------|
| **ALL CAPS** | All segments are uppercase except the 4-char hash suffix (mixed case allowed) |
| **Underscore separator** | Segments are separated by `_` |
| **No spaces** | Spaces are replaced with nothing (omit) or an underscore |
| **No special characters** | Strip all punctuation, slashes, periods, hyphens from name segments |
| **Max segment length** | Manufacturer code: 4–6 chars. Name slug: 4–12 chars. Hash: 4 chars |
| **No leading zeros** | Numeric segments use raw digits, e.g. `308` not `0308` |

#### 4-Character Hash Suffix

Every master-db ID ends in a **4-character alphanumeric hash** to guarantee uniqueness even when the name slug collides. The suffix is **permanent and never reassigned**.

Format options (pick one, be consistent per entity type):

| Style | Example | When to use |
|-------|---------|-------------|
| Sequential within manufacturer | `H1G1`, `H1G2`, `H1G3` | Powders grouped by manufacturer |
| Mixed-case random 4-char | `cR85`, `XBM7`, `2B3E` | Cartridges, diameters, other entities |

**Generating a new hash:** Generate a random 4-character string from `[A-Z0-9a-z]` and verify it does not already exist in the file for that table. Short sequential codes like `H1G1` → `H1G2` are acceptable for batches of records from the same manufacturer.

---

### Format by Entity Type

#### Manufacturers → `MAN_<SLUG>_<HASH>`

| Segment | Rule | Example |
|---------|------|---------|
| `MAN` | Fixed prefix | — |
| `<SLUG>` | 3–6 uppercase chars from manufacturer name | `HODG` (Hodgdon), `VIHT` (Vihtavuori), `ALLI` (Alliant), `SHOO` (Shooters World) |
| `<HASH>` | 4-char unique suffix | `M1G1` |

**Example:** `MAN_HODG_M1G1` → Hodgdon

---

#### Diameters → `DIA_<IMPERIAL>_<HASH>`

| Segment | Rule |
|---------|------|
| `DIA` | Fixed prefix |
| `<IMPERIAL>` | Imperial caliber name, uppercase, no dots. E.g. `308WIN`, `264LBC`, `224VAL` |
| `<HASH>` | 4-char unique suffix |

**Example:** `DIA_308WIN_XBM7` → .308 Win / 7.62mm

---

#### Cartridges → `CTG_<SLUG>_<HASH>`

| Segment | Rule |
|---------|------|
| `CTG` | Fixed prefix |
| `<SLUG>` | Compact cartridge name, uppercase, no spaces. Abbreviate common suffixes: `WIN`=Winchester, `REM`=Remington, `MAG`=Magnum, `BEE`=Bee. Max 12 chars |
| `<HASH>` | 4-char unique suffix. Convention: lowercase `c` prefix (e.g. `cR85`) |

**Examples:**
- `CTG_308WIN_cC82` → .308 Winchester
- `CTG_22HORNET_cR80` → .22 Hornet
- `CTG_65CREEDMR_cN12` → 6.5 Creedmoor

---

#### Bullets → `BUL_<MAN>_<DIA>_<WT>_<NAME>_<HASH>`

| Segment | Rule |
|---------|------|
| `BUL` | Fixed prefix |
| `<MAN>` | 3–6 char manufacturer slug (same as used in `MAN_` ID) |
| `<DIA>` | Bullet diameter in thousandths. E.g. `224`, `308`, `264` |
| `<WT>` | Advertised weight in grains as integer. E.g. `175`, `77`, `140` |
| `<NAME>` | 2–4 char abbreviation of bullet model. E.g. `HPBT`, `ELD`, `TMK`, `SP` |
| `<HASH>` | 4-char unique suffix |

**Examples:**
- `BUL_SIERRA_224_50_BK_2B3E` → Sierra 50gr BlitzKing .224
- `BUL_BERGER_308_175_HYBRID_A3F1` → Berger 175gr Hybrid .308

---

#### Powders → `PWD_<MAN>_<NAME>_<HASH>`

| Segment | Rule |
|---------|------|
| `PWD` | Fixed prefix |
| `<MAN>` | 4–6 char manufacturer slug |
| `<NAME>` | Powder brand name slug, uppercase, no spaces or hyphens. `H4350`, `VARGET`, `RETUMBO`, `N140` |
| `<HASH>` | 4-char unique suffix. Convention: manufacturer initial + sequential digits (e.g. `H1G1`, `V1H3`) |

**Examples:**
- `PWD_HODG_VARGET_H1G1` → Hodgdon Varget
- `PWD_VIHT_N140_V1H6` → Vihtavuori N140
- `PWD_SHOO_THE_CREED_8NG4` → Shooters World The Creed

---

#### Primers → `PRI_<MAN>_<NAME>_<HASH>`

| Segment | Rule |
|---------|------|
| `PRI` | Fixed prefix |
| `<MAN>` | Manufacturer slug |
| `<NAME>` | Model name slug |
| `<HASH>` | 4-char unique suffix |

**Example:** `PRI_FED_210M_P1H1` → Federal GM210M

---

#### Brass → `BRS_<MAN>_<CTG>_<HASH>`

| Segment | Rule |
|---------|------|
| `BRS` | Fixed prefix |
| `<MAN>` | Manufacturer slug |
| `<CTG>` | Cartridge slug (matches `CTG_` name segment) |
| `<HASH>` | 4-char unique suffix |

**Example:** `BRS_LAPUA_308WIN_B1H1` → Lapua .308 Win brass

---

#### Loads → `LOAD_<HEX16>`

Loads use a 16-character lowercase hex string (no structured prefix segments beyond `LOAD_`):

```
LOAD_71b6d8c00511a4f3
```

Generate with: `'LOAD_' + crypto.randomUUID().replace(/-/g, '').slice(0, 16)`

---

#### Primer Pockets → `PKT_<SIZE>`

Fixed values only. Do not add new entries without updating the engine.

| ID | Meaning |
|----|---------|
| `PKT_SML` | Small primer pocket |
| `PKT_LRG` | Large primer pocket |

---

#### User-Created Records (Firearms, Sessions, Groups, Shots, etc.)

User-created records use **UUID v4** strings generated by the app at creation time:

```
8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a
```

These are always lowercase hyphenated UUIDs. Never apply the structured compound key format to user records.

---

#### Quick Reference Table

| Table | Pattern | Real Example |
|-------|---------|--------------|
| Manufacturers | `MAN_<SLUG>_<HASH>` | `MAN_HODG_M1G1` |
| Diameters | `DIA_<IMPERIAL>_<HASH>` | `DIA_308WIN_XBM7` |
| Cartridges | `CTG_<SLUG>_c<HASH>` | `CTG_22HORNET_cR80` |
| Bullets | `BUL_<MAN>_<DIA>_<WT>_<NAME>_<HASH>` | `BUL_SIERRA_224_50_BK_2B3E` |
| Powders | `PWD_<MAN>_<NAME>_<HASH>` | `PWD_HODG_VARGET_H1G1` |
| Primers | `PRI_<MAN>_<NAME>_<HASH>` | `PRI_FED_210M_P1H1` |
| Brass | `BRS_<MAN>_<CTG>_<HASH>` | `BRS_LAPUA_308WIN_B1H1` |
| Loads | `LOAD_<HEX16>` | `LOAD_71b6d8c00511a4f3` |
| Primer Pockets | `PKT_<SIZE>` | `PKT_LRG` |
| Firearms | UUID v4 | `8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a` |
| Sessions / Groups / Shots | UUID v4 | `c8b3d6f1-4e8a-4d7a-8b9c-2d3e4f5a6b7c` |

---

## Manufacturers [master-db]

```json
{
  "id": "MAN_HODG_M1G1",
  "name": "Hodgdon",
  "displayName": "Hodgdon",
  "type": ["powder"]
}
```

* `displayName`? — optional shorthand display name string
* `type`? — array of strings. Valid values: `bullet`, `powder`, `primer`, `brass`, `ammo`

Dexie index: `id`

---

## Diameters [master-db]

```json
{
  "id": "DIA_308WIN_XBM7",
  "imperial": ".308",
  "metric": "7.62mm"
}
```

Dexie index: `id`

---

## Cartridges [master-db]

```json
{
  "id": "CTG_308WIN_cC82",
  "name": ".308 Winchester",
  "diameterId": "DIA_308WIN_XBM7",
  "minCaseLengthMm": 50.80,
  "maxCaseLengthMm": 51.18,
  "trimLengthMm": 50.92,
  "oalMm": 71.12,
  "maxSaamiPa": 427474952.17,
  "baseCapacityH2oGrams": 3.6252,
  "boreDiameterMm": 7.62,
  "bulletDiameterMm": 7.82,
  "flashHoleDiameterMm": 2.03,
  "shoulderAngleDeg": 20,
  "bodyDiameterMm": 11.96,
  "throatFreetravelMm": 3.43,
  "transducerScaleFactor": 1.0,
  "freeboreLengthMm": 3.43
}
```

* All length and diameter fields are in **millimeters**
* `minCaseLengthMm`? — minimum case length in millimeters
* `maxCaseLengthMm`? — maximum case length in millimeters
* `trimLengthMm`? — case trim-to length in millimeters
* `oalMm`? — nominal overall cartridge length in millimeters
* `maxSaamiPa`? — SAAMI maximum average pressure in Pascals (Pa)
* `baseCapacityH2oGrams`? — case water capacity in grams of H₂O (grams, not grains)
* `boreDiameterMm`? — land-to-land bore diameter in mm (NOT groove/bullet diameter). Source: SAAMI spec sheets
* `bulletDiameterMm`? — nominal projectile groove/bullet diameter in millimeters
* `flashHoleDiameterMm`? — SAAMI/CIP spec flash hole diameter in millimeters
* `shoulderAngleDeg`? — SAAMI/CIP shoulder half-angle in degrees
* `bodyDiameterMm`? — SAAMI/CIP external body diameter at base (P1) in millimeters
* `throatFreetravelMm`? — freebore distance before rifling engagement in millimeters
* `transducerScaleFactor`? — conformal piezo transducer scaling factor (default 1.0). Aligns breech pressure output with SAAMI copper crusher equivalents. Cartridge-specific; calibrated from published data
* `freeboreLengthMm`? — alias for `throatFreetravelMm` used by some code paths

Dexie index: `id, diameterId`

---

## Bullets [master-db]

```json
{
  "id": "BUL_SIERRA_308_175_HPBT_A3F1",
  "manufacturerId": "MAN_SIER_M1G1",
  "diameterId": "DIA_308WIN_XBM7",
  "name": "175gr MatchKing HPBT",
  "advertisedWeightGrains": 175,
  "physis": {
    "weightGrams": 11.34,
    "overallLengthMm": 35.0,
    "ogiveLengthMm": 17.97,
    "boatTailLengthMm": 4.7,
    "tipLengthMm": null,
    "meplatDiameterMm": 1.5,
    "bearingSurfaceMm": 12.33,
    "materialType": "jacketed_lead"
  },
  "ballistics": {
    "preferredModel": "G7",
    "g1BC": 0.505,
    "g7BC": 0.258,
    "g1FF": null,
    "g7FF": null
  },
  "stability": {
    "ix": null,
    "iy": null
  }
}
```

* `advertisedWeightGrains`? — bullet weight in grains as labeled by manufacturer
* All length and diameter fields in `physis` are in **millimeters**
* `physis.weightGrams` — bullet weight in **grams** (convert: grains ÷ 15.4324 = grams)
* `physis.bearingSurfaceMm`? — length of the bullet bearing surface in millimeters
* `physis.tipLengthMm`? — for tipped bullets: the plastic tip length only in millimeters (not included in aerodynamic metal length)
* `physis.materialType`? — bullet jacket/construction material. Valid values: `"jacketed_lead"`, `"monolithic_copper"`, `"cast_lead"`. Used to scale engraving resistance in internal ballistics simulation
* `ballistics.g1FF`? / `ballistics.g7FF`? — form factors relative to G1/G7 standard projectile
* `stability.ix`? / `stability.iy`? — moments of inertia (kg·m²), used by stability calculator when available
* `ballistics.preferredModel`? — `"G1"` or `"G7"`. G7 is preferred for long-range boat-tail rifle bullets

Dexie index: `id, manufacturerId, diameterId`

---

## Powders [master-db]

Powder records in `local-db.json` separate **physical constants** (stored at the root of the powder object, source of truth from manufacturer/chemistry literature) from **calibrated optimization values** (stored in the top-level `optimizations.powders[]` array and merged at load time).

### Physical Constants (root-level, source of truth)

```json
{
  "id": "PWD_HODG_VARGET_H1G1",
  "manufacturerId": "MAN_HODG_M1G1",
  "name": "Varget",
  "heatOfExplosionKjKg": 3750,
  "grainType": "extrudedSinglePerf",
  "propellantDensityKgM3": 1620,
  "bulkDensityKgM3": 920
}
```

* `heatOfExplosionKjKg`? — heat of explosion in kJ/kg. Physical constant from chemistry literature. **Do not calibrate this field.** Reference values: single-base NC ≈ 3750; double-base NC+NG ≈ 3950–4200. Source of truth — never overwrite with a calibrated value
* `grainType`? — physical geometry of powder grains. Allowed values: `"ball"`, `"flake"`, `"extrudedSinglePerf"`, `"extrudedMultiPerf"`, `"extruded"`. Affects form factor model in burn rate ODE
* `propellantDensityKgM3`? — solid density of the powder material in kg/m³. Used to compute fill fraction and flame spread. Physical property — do not calibrate. Typical values: ball 1590–1640, extruded 1600–1640
* `bulkDensityKgM3`? — bulk (poured) density of the powder in kg/m³. Used to compute case fill percentage. Typically 0.55–0.65 × solid density. Physical property

### Calibrated Optimization Values (`optimizations.powders[]`)

The following fields are stored in the `optimizations` top-level key, **not** on the powder root. `loadDb()` merges them into the powder object at runtime. These are outputs of `calibrateV3.ts` and must never be treated as physical constants.

```json
{
  "optimizations": {
    "powders": [
      {
        "id": "PWD_HODG_VARGET_H1G1",
        "burnAreaCoeff": 0.2285,
        "burnAreaFillSlope": 0.04,
        "burnAreaBoreSlope": 0.0,
        "burnExponent": 0.65,
        "energyScaleFactor": 1.18,
        "ignitionBp": 0.15,
        "ignitionZ1": 0.50,
        "ignitionZ2": 0.82
      }
    ]
  }
}
```

* `burnAreaCoeff` — base burn rate coefficient at the 50% reference fill fraction. Calibrated by `calibrateV3.ts --all`. Replaces legacy `baCoeff`
* `burnAreaFillSlope` — linear fill-fraction slope for `burnAreaCoeff`. Controls how burn rate scales with loading density above/below the 50% reference. Replaces legacy `baFillSlope`. Calibrated by `calibrateV3.ts --fill-slope`. Only populated when ≥10 loads span ≥8% fill range; otherwise `0`
* `burnAreaBoreSlope` — bore-diameter correction slope for `burnAreaCoeff`. Applies a non-linear area-ratio correction for small/large bore diameters: `ba_effective *= (1 + burnAreaBoreSlope × (1 − (7.62 / boreDiaMm)²))`. At the reference bore (7.62 mm), factor = 1.0 (zero correction). Negative values boost small-bore efficiency. Default `0` (no bore correction)
* `burnExponent` — pressure exponent in Vieille's burn law. Calibrated by `calibrateV3.ts --burn-exp`. Typical range 0.55–0.85
* `energyScaleFactor` — engine-level energy efficiency multiplier (dimensionless). Absorbs model efficiency deficits not captured by physical heat values. Calibrated by `calibrateV3.ts`. Values persistently above 1.25 indicate a powder with incomplete combustion data or an unusual grain geometry. **Not a physical constant**
* `ignitionBp`? / `ignitionZ1`? / `ignitionZ2`? — optional multi-stage burn profile parameters for the grain form factor model. Calibrated when multi-stage behavior is detected

Dexie index: `id, manufacturerId`

---

## Primers [master-db]

```json
{
  "id": "PRI_FED_210M_P1H1",
  "manufacturerId": "MAN_FED_M1G1",
  "name": "Federal GM210M",
  "type": "large_rifle_match",
  "primerPocketId": "PKT_LRG",
  "brisanceEnergyJ": 14.0
}
```

* `type`? — primer size/type description string
* `primerPocketId`? — references `primerPockets` collection. Valid values: `PKT_SML`, `PKT_LRG`
* `brisanceEnergyJ`? — initial ignition spark energy in Joules (J). Used in internal ballistics simulation. Typical values: Small rifle: `8.0`, Large rifle: `14.0`, Magnum: `20.0`

Dexie index: `id, manufacturerId`

---

## Primer Pockets [master-db]

```json
{
  "id": "PKT_LRG",
  "name": "LARGE"
}
```

Fixed values. Valid IDs: `PKT_SML` (small), `PKT_LRG` (large).

Dexie index: `id`

---

## Brass [master-db]

```json
{
  "id": "BRS_LAPUA_308WIN_B1H1",
  "manufacturerId": "MAN_LAPU_M1G1",
  "cartridgeId": "CTG_308WIN_cC82",
  "primerPocketId": "PKT_LRG",
  "primerHole": 2.03,
  "capacityH2oGrams": 3.6287
}
```

* `primerHole`? — physical flash hole diameter in millimeters (typical Small: `1.60`, Large: `2.06`)
* `capacityH2oGrams`? — case overflow water capacity in **grams** of H₂O

Dexie index: `id, cartridgeId, manufacturerId`

---

## Loads [master-db + user]

Commercial and handload records share the same table. Fields differ by type.

### Units Convention for loads.json

> **All mass fields in loads.json are stored in metric (grams/mm).** The scraper and importer are responsible for unit conversion before writing. The engine and calibrator read metric values directly.

| Field | Unit |
|-------|------|
| `chargeWeightGrams` | grams |
| `velocityMps` | meters per second |
| `coalMm` | millimeters |
| `cbtoMm` | millimeters |
| `barrelLenMm` | millimeters |
| `pressurePsi` | PSI (exception — raw sensor units from manufacturer data) |

### Handload Example
```json
{
  "id": "LOAD_71b6d8c00511a4f3",
  "loadTypeId": "LT_HAND",
  "name": "308 Win / 175gr SMK / Varget 44gr",
  "handloadName": "Ref Handload",
  "cartridgeId": "CTG_308WIN_cC82",
  "bulletId": "BUL_SIERRA_308_175_HPBT_A3F1",
  "powderId": "PWD_HODG_VARGET_H1G1",
  "primerId": "PRI_FED_210M_P1H1",
  "brassId": "BRS_LAPUA_308WIN_B1H1",
  "chargeWeightGrams": 2.8512,
  "coalMm": 71.12,
  "cbtoMm": 67.20,
  "velocityMps": 870.0,
  "barrelLenMm": 660.4,
  "pressurePsi": 58500,
  "isCommercial": false,
  "quarantine": false,
  "notes": ""
}
```

* `loadTypeId`? — references load type definition (`"LT_COMM"` or `"LT_HAND"`)
* `isCommercial`? — boolean flag distinguishing commercial factory ammunition from handloads
* `handloadName`? — specific custom user-facing handload nickname string
* `coalMm`? — Cartridge Overall Length in **millimeters**
* `cbtoMm`? — Cartridge Base-to-Ogive length in **millimeters**
* `chargeWeightGrams`? — propellant charge weight in **grams** (convert from grains: × 0.0647989)
* `velocityMps`? — recorded average muzzle velocity in **meters per second** (convert from fps: × 0.3048)
* `barrelLenMm`? — test barrel length in **millimeters** (required for calibration — loads without this field are skipped by `calibrateV3.ts`)
* `pressurePsi`? — recorded peak chamber pressure in **PSI** (as published by manufacturer; raw transducer values)
* `quarantine`? — boolean. When `true`, the load is excluded from calibration and simulation. Used for loads that cannot yet be modeled correctly (e.g., subsonic loads, pistol loads pending data correction)

Dexie index: `id, cartridgeId, bulletId, powderId`

---

## Firearms [user-only]

```json
{
  "id": "8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a",
  "nickname": "6.5 Creedmoor Precision Rifle",
  "cartridgeId": "CTG_65CREEDMR_cN12",
  "barrelLengthMm": 660.4,
  "twistRateMm": 203.2,
  "sightOverBoreMm": 48.26
}
```

* `barrelLengthMm`? — barrel length in millimeters
* `twistRateMm`? — rifling twist rate in mm per turn (e.g., `203.2` = 1:8", `254.0` = 1:10")
* `sightOverBoreMm`? — scope centerline height above bore centerline in millimeters

Dexie index: `id, cartridgeId`

---

## Marked Targets [user-only]

```json
{
  "id": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "name": "100yd Group — Session 1",
  "targetDistance": 100,
  "distanceUnits": "yards",
  "createdAt": 1781546283422
}
```

* `targetDistance`? — target distance value
* `distanceUnits`? — `"yards"` or `"meters"`
* `createdAt`? — Unix timestamp in milliseconds

Dexie index: `id`

---

## Sessions [user-only]

```json
{
  "id": "c8b3d6f1-4e8a-4d7a-8b9c-2d3e4f5a6b7c",
  "name": "Session 1",
  "markedTargetId": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "firearmId": "8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a",
  "loadId": "LOAD_71b6d8c00511a4f3",
  "targetDistance": 100,
  "distanceUnits": "yards",
  "temp": 59.0,
  "altitude": 1000.0,
  "pressure": 29.92,
  "pressureType": "station"
}
```

* `temp`? — environmental temperature during session in **Fahrenheit**
* `altitude`? — local altitude in **feet**
* `pressure`? — station or barometric pressure in **inches of mercury** (inHg)
* `pressureType`? — `"station"` or `"sea"`

Dexie index: `id, firearmId, loadId, markedTargetId, [firearmId+loadId]`

The compound index `[firearmId+loadId]` enables efficient queries like "all sessions for this exact firearm+load combination."

---

## Session Targets [user-only]

```json
{
  "id": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "sessionId": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "targetImageId": "f2b4c6d8-0e2a-4b6c-8d0e-2a4b6c8d0e2a",
  "scale": {
    "p1": { "x": 100, "y": 200 },
    "p2": { "x": 300, "y": 200 },
    "distance": 2.0,
    "units": "in",
    "pixelsPerUnit": 100.0
  },
  "transform": { "scale": 1.0 }
}
```

* **Note on sessionId**: Holds the ID of the linked **markedTargets** record (not the session record)
* `scale.p1` / `scale.p2` — pixel coordinates of the two scale reference points on the canvas
* `scale.distance` — physical distance value between points
* `scale.pixelsPerUnit` — derived from the p1/p2 distance and the known physical distance

Dexie index: `id, sessionId`

---

## Groups [user-only]

```json
{
  "id": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "sessionId": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "targetId": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "groupNum": 1,
  "poa": { "x": 450.5, "y": 312.0 },
  "color": "#ef4444"
}
```

* **Note on sessionId**: Holds the ID of the linked **markedTargets** record (not the session record)
* `groupNum`? — order index of group
* `poa` — pixel coordinates of the point of aim on the target image
* `color` — hex color used to render this group on the composite plot

Dexie index: `id, sessionId`

---

## Shots [user-only]

```json
{
  "id": "d5c4b3a2-9e8d-7c6b-5a4b-3c2d1e0f9a8b",
  "groupId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "sessionId": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "targetId": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "shotNumber": 1,
  "x": 0.152,
  "y": -0.218,
  "units": "in",
  "velocity": 2705,
  "px": 465.7,
  "py": 333.8
}
```

* **Note on sessionId**: Holds the ID of the linked **markedTargets** record
* `x` / `y` — physical offset from POA in `units` (inches or cm)
* `px` / `py` — pixel coordinates on the canvas
* `velocity` — muzzle velocity in **fps**, or `null` if not recorded

Dexie index: `id, groupId, sessionId, targetId`

---

## Target Images [user-only]

```json
{
  "id": "f2b4c6d8-0e2a-4b6c-8d0e-2a4b6c8d0e2a",
  "name": "100yd Target",
  "timestamp": "2026-06-01T14:30:00.000Z",
  "imageBlob": "<Blob>",
  "size": "245 KB",
  "firearmId": "8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a",
  "loadId": "LOAD_71b6d8c00511a4f3"
}
```

* `imageBlob` — stored as a native binary `Blob` in IndexedDB
* **JSON export/import:** `imageBlob` is automatically converted to/from a Base64 `dataUrl` string at the export/import boundary

Dexie index: `id`

---

## Custom Targets [user-only]

```json
{
  "id": "b7c9e1d3-4f0a-4b6c-8d2e-4a6b8c0d2e4f",
  "name": "100yd Precision Target",
  "paperSize": "letter",
  "orientation": "portrait",
  "gridEnabled": true,
  "gridSize": 1,
  "gridColor": "#cccccc",
  "rows": 1,
  "cols": 1,
  "marginX": 0.5,
  "marginY": 0.5,
  "shape": "circle",
  "diameter": 3.0,
  "numRings": 5,
  "bullseyeColor": "#000000",
  "ringColorA": "#000000",
  "ringColorB": "#ffffff",
  "labelText": "100 yards",
  "labelPosition": "bottom",
  "labelSize": 12,
  "labelMargin": 0.25
}
```

Dexie index: `id`

---

*For the mathematical derivations behind the simulation engines, see [MATHS.md](MATHS.md).*
*For the user-facing guide, see [README.md](README.md).*
