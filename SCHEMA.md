# Empirical Precision — Database Schema Reference

This document is the canonical technical reference for all IndexedDB tables and the `local-db.json` / `master-db.json` structure.

**Convention:** All field names use **camelCase**. All IDs are strings. Optional fields are marked with `?`.

Tables marked **[master-db]** are seeded from the open-source library. Tables marked **[user-only]** are created locally and never appear in `master-db.json`. Tables marked **[user — IndexedDB only]** exist solely in IndexedDB and are never serialised to JSON.

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
| `<SLUG>` | Compact cartridge name, uppercase, no spaces. Max 12 chars |
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
| `<MAN>` | 3–6 char manufacturer slug |
| `<DIA>` | Bullet diameter in thousandths. E.g. `224`, `308`, `264` |
| `<WT>` | Advertised weight in grains as integer |
| `<NAME>` | 2–4 char abbreviation of bullet model |
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
| `<NAME>` | Powder brand name slug, uppercase, no spaces or hyphens |
| `<HASH>` | 4-char unique suffix. Convention: manufacturer initial + sequential digits |

**Examples:**
- `PWD_HODG_VARGET_H1G1` → Hodgdon Varget
- `PWD_VIHT_N140_V1H6` → Vihtavuori N140

---

#### Primers → `PRI_<MAN>_<NAME>_<HASH>`

**Example:** `PRI_FED_210M_P1H1` → Federal GM210M

---

#### Brass → `BRS_<MAN>_<CTG>_<HASH>`

**Example:** `BRS_LAPUA_308WIN_B1H1` → Lapua .308 Win brass

---

#### Loads → `LOAD_<HEX16>`

```
LOAD_71b6d8c00511a4f3
```

Generate with: `'LOAD_' + crypto.randomUUID().replace(/-/g, '').slice(0, 16)`

---

#### Primer Pockets → `PKT_<SIZE>`

Fixed values only.

| ID | Meaning |
|----|---------|
| `PKT_SML` | Small primer pocket |
| `PKT_LRG` | Large primer pocket |

---

#### User-Created Records

User-created records use UUID v4 strings: `8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a`

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
  "id": "DIA_224_L5X8",
  "imperialName": ".224",
  "metricName": "5.56",
  "bulletDiameterMm": 5.6896,
  "boreDiameterMm": 5.5626
}
```

* `imperialName` — imperial caliber display string (e.g. `".308"`)
* `metricName` — metric display string (e.g. `"7.62mm"`)
* `bulletDiameterMm` — nominal groove/bullet diameter in millimeters
* `boreDiameterMm` — land-to-land bore diameter in millimeters

> **Note:** Field names changed from `imperial`/`metric` (old schema) to `imperialName`/`metricName`. The old names are no longer valid.

Dexie index: `id`

---

## Cartridges [master-db]

```json
{
  "id": "CTG_22ARC_cK12",
  "name": "22 ARC",
  "diameterId": "DIA_224_L5X8",
  "primerPocketId": "PKT_SML",
  "baseCapacityH2oGrams": 2.242,
  "capacitySource": "SAAMI spec sheet",
  "trimLengthMm": 38.481,
  "maxCaseLengthMm": 38.735,
  "oalMm": 57.404,
  "maxSaamiPa": 358527379,
  "flashHoleDiameterMm": 1.6,
  "shoulderAngleDeg": 30,
  "bodyDiameterMm": 11.2,
  "bulletDiameterMm": 5.69,
  "boreDiameterMm": 5.563,
  "freeboreLengthMm": 1.5,
  "freeboreDiameterMm": 5.72,
  "throatFreetravelMm": 1.5,
  "standardTwistMm": 203.2,
  "cipTestBarrelMm": 609.6,
  "standards": [
    {
      "standard": "saami",
      "maxPressurePsi": 52000,
      "nominalVelocityFps": 2800,
      "referenceSource": "ANSI/SAAMI Standards"
    },
    {
      "standard": "cip",
      "maxPressureBar": 3585,
      "referenceSource": "C.I.P. TDCC Sheet 22 ARC"
    }
  ],
  "externalDimensions": {
    "baseDiameterP1Mm": 11.2,
    "bodyDiameterAtShoulderP2Mm": 11.1,
    "shoulderNeckJunctionDiameterH1Mm": 6.4,
    "neckDiameterAtMouthH2Mm": 6.2,
    "lengthToShoulderStartL1Mm": 30.0,
    "lengthToShoulderNeckJunctionL2Mm": 35.8,
    "totalCaseLengthL3Mm": 38.735
  },
  "wallThicknessProfile": {
    "webThicknessMm": 1.35,
    "wallThicknessAtBaseMm": 0.95,
    "wallThicknessAtMidBodyMm": 0.42,
    "wallThicknessAtShoulderMm": 0.38,
    "wallThicknessAtNeckMm": 0.34
  }
}
```

### Root Fields

* All length and diameter fields are in **millimeters**
* `diameterId` — references `diameters` collection
* `primerPocketId`? — references `primerPockets` collection (`PKT_SML` or `PKT_LRG`)
* `minCaseLengthMm`? — minimum allowable case length in mm
* `maxCaseLengthMm`? — maximum allowable case length in mm
* `trimLengthMm`? — recommended case trim-to length in mm
* `oalMm`? — nominal cartridge overall length (SAAMI) in mm
* `maxSaamiPa`? — SAAMI maximum average pressure in Pascals. **Authoritative engine value**
* `baseCapacityH2oGrams`? — case water capacity at the base in grams of H₂O
* `capacitySource`? — provenance string for `baseCapacityH2oGrams` (e.g. `"SAAMI spec sheet"`, `"case weigh/fill test"`)
* `boreDiameterMm`? — land-to-land bore diameter in mm. Source: SAAMI spec sheets
* `bulletDiameterMm`? — nominal projectile groove/bullet diameter in millimeters
* `flashHoleDiameterMm`? — SAAMI/CIP spec flash hole diameter in millimeters
* `shoulderAngleDeg`? — SAAMI/CIP shoulder half-angle in degrees
* `bodyDiameterMm`? — SAAMI/CIP external body diameter at base (P1) in millimeters
* `throatFreetravelMm`? — freebore from cartridge spec (SAAMI/CIP) in mm
* `freeboreLengthMm`? — freebore length (same physical dimension as `throatFreetravelMm`)
* `freeboreDiameterMm`? — freebore throat diameter in mm
* `standardTwistMm`? — SAAMI standard twist rate in mm per turn (e.g. `203.2` = 1:8")
* `cipTestBarrelMm`? — CIP test barrel length in mm
* `twistRateInchesPerTurn`? — *(legacy, prefer `standardTwistMm`)* twist rate in inches per turn
* `maxSaamiPressurePsi`? — *(legacy, prefer `maxSaamiPa`)* pressure in PSI; only used for display

### `standards[]` Array

Replaces the old `standards.saami` / `standards.cip` nested object. Each element:

| Field | Description |
|-------|-------------|
| `standard` | `"saami"` or `"cip"` |
| `maxPressurePsi`? | SAAMI max pressure in PSI (saami only) |
| `maxPressureBar`? | CIP max pressure in bar (cip only) |
| `nominalVelocityFps`? | Published reference velocity in fps |
| `referenceSource`? | Source document string |

### `externalDimensions` Object (required for volume solver)

| Field | Description |
|-------|-------------|
| `baseDiameterP1Mm` | Case head diameter (P1) in mm |
| `bodyDiameterAtShoulderP2Mm` | Body diameter at shoulder start (P2) in mm |
| `shoulderNeckJunctionDiameterH1Mm` | Diameter at shoulder/neck junction (H1) in mm |
| `neckDiameterAtMouthH2Mm` | External neck diameter at case mouth (H2) in mm |
| `lengthToShoulderStartL1Mm` | Case length from base to shoulder start (L1) in mm |
| `lengthToShoulderNeckJunctionL2Mm` | Case length from base to shoulder/neck junction (L2) in mm |
| `totalCaseLengthL3Mm` | Total case length (L3) in mm |

### `wallThicknessProfile` Object

| Field | Description |
|-------|-------------|
| `webThicknessMm` | Solid brass web thickness at the base in mm |
| `wallThicknessAtBaseMm` | Wall thickness just above the web in mm |
| `wallThicknessAtMidBodyMm`? | Wall thickness at mid-body in mm |
| `wallThicknessAtShoulderMm` | Wall thickness at the shoulder in mm |
| `wallThicknessAtNeckMm` | Wall thickness at the neck in mm |

Dexie index: `id, diameterId`

---

## Bullets [master-db]

```json
{
  "id": "BUL_BERGER_277_145_FB_EH45",
  "name": "Flat Base Target",
  "manufacturerId": "MAN_BERGER_B9K2",
  "diameterId": "DIA_277_K9L0",
  "advertisedWeightGrains": 145,
  "isMatchGrade": true,
  "sierraPartNum": null,
  "notes": null,
  "physis": {
    "weightGrams": 9.3958,
    "overallLengthMm": 35.2552,
    "ogiveLengthMm": 17.818,
    "boatTailLengthMm": 5.2354,
    "bearingSurfaceMm": 12.1877,
    "materialType": "MAT_JACKETED_LEAD",
    "engravingPressurePa": 32000000
  },
  "ballistics": {
    "g1BC": 0.503,
    "g7BC": 0.25,
    "g1FF": 0.536,
    "g7FF": 1.078
  }
}
```

* `advertisedWeightGrains`? — bullet weight in grains as labeled by manufacturer
* `isMatchGrade`? — boolean; `true` for match-grade competition bullets
* `sierraPartNum`? — Sierra part number string (null for non-Sierra bullets)
* `notes`? — free-text notes string
* All length and diameter fields in `physis` are in **millimeters**
* `physis.weightGrams` — bullet weight in **grams** (convert: grains ÷ 15.4324 = grams)
* `physis.bearingSurfaceMm`? — bearing surface length in millimeters
* `physis.tipLengthMm`? — *(optional)* plastic tip length in mm (not included in aerodynamic metal length)
* `physis.meplatDiameterMm`? — *(optional)* meplat diameter in mm
* `physis.materialType`? — Valid `MAT_` prefixed values: `"MAT_JACKETED_LEAD"`, `"MAT_MONOLITHIC_COPPER"`, `"MAT_CAST_LEAD"`, `"MAT_RELIEF_GROOVED_COPPER_MONO"`. Legacy bare strings (`"jacketed_lead"`, etc.) are accepted but deprecated
* `physis.engravingPressurePa`? — engraving resistance pressure in Pascals. Used in the internal ballistics engine to model the bullet's initial free-travel resistance force. Typical value: `32_000_000` Pa (32 MPa)
* `ballistics.g1BC`? / `ballistics.g7BC`? — ballistic coefficients for G1/G7 drag models
* `ballistics.g1FF`? / `ballistics.g7FF`? — form factors relative to G1/G7 standard projectile
* `ballistics.preferredModel`? — *(TypeScript interface field, not currently populated)* `"G1"` or `"G7"`

Dexie index: `id, manufacturerId, diameterId`

---

## Powders [master-db]

Powder records separate **physical constants** (root-level) from **calibrated optimization values** (in `tuning.powders[]`, merged at runtime). **Never store calibrated fields on the powder root.**

### Physical Constants (root-level)

```json
{
  "id": "PWD_HODG_VARGET_H1G1",
  "manufacturerId": "MAN_HODG_M1G1",
  "name": "Varget",
  "heatOfExplosionKjKg": 3750,
  "heatSource": "Propellant Manufacturer Technical Data",
  "grainType": "extrudedSinglePerf",
  "propellantDensityKgM3": 1620,
  "bulkDensityKgM3": 920,
  "kCoeff": 1.2409,
  "burnExponent": 0.5502,
  "tempSensitivity": 0.0027,
  "ignitionBp": 0.2256,
  "ignitionZ1": 0.5689,
  "ignitionZ2": 0.5812,
  "_calibrationNote": "Initial baseline from Hodgdon data sheet"
}
```

* `heatOfExplosionKjKg`? — heat of explosion in kJ/kg. **Do not calibrate.** Single-base NC ≈ 3750; double-base NC+NG ≈ 3950–4200
* `heatSource`? — provenance string (e.g. `"Propellant Manufacturer Technical Data"`, `"GRT"`, `"hodgdon-adc"`)
* `grainType`? — grain geometry. Values: `"ball"`, `"flake"`, `"extrudedSinglePerf"`, `"extrudedMultiPerf"`, `"extruded"`
* `propellantDensityKgM3`? — solid propellant density in kg/m³. Physical property — do not calibrate
* `bulkDensityKgM3`? — bulk (poured) density in kg/m³. Used to compute case fill percentage. Physical property
* `kCoeff`? — adiabatic exponent ratio for Noble-Abel EOS. Typical: `1.23` (single-base), `1.24`–`1.255` (double-base). Physical constant — do not calibrate
* `burnExponent`? — Vieille's law pressure exponent. **Calibrated.** Historically stored on powder root. Typical range 0.55–0.85
* `tempSensitivity`? — temperature sensitivity coefficient in /°C. Physical constant
* `ignitionBp`? / `ignitionZ1`? / `ignitionZ2`? — multi-stage burn profile parameters. **Calibrated.** Historically stored on powder root
* `_calibrationNote`? — internal annotation string; not used by engine

> **Separation rule:** `burnAreaCoeff`, `burnAreaFillSlope`, `burnAreaBoreSlope`, `burnAreaExpansionSlope`, `energyScaleFactor`, `calibrationConfidence`, and `calibrationFlags` are **never** stored on the powder root. They live exclusively in `tuning.powders[]`.

### Calibrated Optimization Values (`tuning.powders[]`)

```json
{
  "id": "PWD_HODG_VARGET_H1G1",
  "burnAreaCoeff": 0.2285,
  "burnAreaFillSlope": 0.04,
  "burnAreaBoreSlope": 0.0,
  "burnAreaExpansionSlope": 0.0,
  "energyScaleFactor": 1.18,
  "calibrationConfidence": "HIGH",
  "calibrationFlags": []
}
```

| Field | Description |
|-------|-------------|
| `burnAreaCoeff` | Base burn rate coefficient at 50% reference fill fraction. Output of `calibrateV3.ts --all` |
| `burnAreaFillSlope` | Linear fill-fraction slope. Controls burn rate scaling with loading density. Only populated when ≥10 loads span ≥8% fill range; otherwise `0` |
| `burnAreaBoreSlope` | Bore-diameter correction slope. At reference bore (7.62 mm) factor = 1.0. Default `0` |
| `burnAreaExpansionSlope` | Gas expansion slope correction during barrel travel phase. Default `0` |
| `energyScaleFactor` | Engine-level energy efficiency multiplier. **Not a physical constant.** Values persistently above 1.25 indicate suspect data |
| `calibrationConfidence` | `"HIGH"` (≥20 loads, R²>0.85), `"MEDIUM"` (10–19 loads or R²>0.70), `"LOW"` (<10 loads or R²≤0.70) |
| `calibrationFlags` | Array of warning strings from the calibrator. e.g. `["SPARSE_DATA"]`, `["FILL_SLOPE_CLAMPED"]`. Empty = no flags |

Dexie index: `id, manufacturerId`

---

## Primers [master-db]

```json
{
  "id": "PRI_CCI_NO41_C1R7",
  "manufacturerId": "MAN_CCI_C2W7",
  "name": "No. 41 Small Rifle Military",
  "primerPocketId": "PKT_SML",
  "brisanceEnergyJ": 8
}
```

* `primerPocketId`? — references `primerPockets` collection
* `brisanceEnergyJ`? — initial ignition spark energy in Joules. Typical: Small rifle `8.0`, Large rifle `14.0`, Magnum `20.0`
* `type`? — *(TypeScript interface field, not currently populated in local-db.json)* primer size/type description string

Dexie index: `id, manufacturerId`

---

## Primer Pockets [master-db]

```json
{
  "id": "PKT_SML",
  "name": "SMALL",
  "pocketDiameterMinMm": 4.394,
  "pocketDiameterMaxMm": 4.432,
  "pocketDepthMinMm": 2.972,
  "pocketDepthMaxMm": 3.124
}
```

* `pocketDiameterMinMm`? / `pocketDiameterMaxMm`? — SAAMI/CIP primer pocket reamer diameter tolerance in mm
* `pocketDepthMinMm`? / `pocketDepthMaxMm`? — SAAMI/CIP primer pocket depth tolerance in mm

Fixed values. Valid IDs: `PKT_SML` (small), `PKT_LRG` (large).

Dexie index: `id`

---

## Brass [master-db]

```json
{
  "id": "BRS_MAN_ADG_A4D7_CTG_65CM_cN72_TR53",
  "manufacturerId": "MAN_ADG_A4D7",
  "cartridgeId": "CTG_65CM_cN72",
  "primerPocketId": "PKT_SML",
  "capacityH2oGrams": 3.33714
}
```

* `primerPocketId`? — references `primerPockets` collection
* `capacityH2oGrams`? — case overflow water capacity in **grams** of H₂O
* `primerHole`? — *(TypeScript interface field, not currently populated)* flash hole diameter in mm

Dexie index: `id, cartridgeId, manufacturerId`

---

## Loads [master-db + user]

### Units Convention

> **All mass and length fields in loads are stored in metric (grams/mm).** The scraper and importer handle unit conversion. The engine and calibrator read metric values directly.

| Field | Unit |
|-------|------|
| `chargeWeightGrams` | grams |
| `velocityMps` | meters per second |
| `coalMm` | millimeters |
| `cbtoMm` | millimeters |

> **Deprecated fields:** `barrelLenMm`, `pressurePsi`, `loadTypeId`, `isCommercial`, and `quarantine` are **no longer populated** in `local-db.json`. Do not add them to new records.

### Example

```json
{
  "id": "LOD_HDES_65PRC_147ELD",
  "handloadName": "147gr Hornady ELD-M / H4350",
  "cartridgeId": "CTG_65PRC_cP02",
  "bulletId": "BUL_HORNADY_264_147_147_ELDM",
  "powderId": "PWD_HODG_H4350_H1G2",
  "primerId": "PRI_FED_210M_F1D3",
  "brassId": "BRS_MAN_ALPHA_A1L2_CTG_65PRC_cP02_LRP_OCD",
  "chargeWeightGrams": 2.9029912,
  "coalMm": 75.057,
  "velocityMps": 886.97,
  "notes": "44.8gr H4350, COAL 2.955\", 2910 fps avg."
}
```

* `handloadName`? — user-facing handload nickname string
* `name`? — alternative display name (used for factory/commercial loads)
* `coalMm`? — Cartridge Overall Length in **millimeters**
* `cbtoMm`? — Cartridge Base-to-Ogive length in **millimeters**
* `chargeWeightGrams`? — propellant charge weight in **grams** (convert from grains: × 0.0647989)
* `velocityMps`? — recorded average muzzle velocity in **meters per second** (convert from fps: × 0.3048)
* `notes`? — free-text string

Dexie index: `id, cartridgeId, bulletId, powderId`

---

## Tuning Block [master-db — not a Dexie table]

The `tuning` key is a top-level object in `local-db.json` that stores all calibrated engine parameters. It is **not** a Dexie table — `loadDb()` merges it into the relevant entity objects at runtime.

```json
{
  "tuning": {
    "powders": [ ... ],
    "cartridges": [ ... ],
    "powderCartridgeCalibrations": [],
    "velocityCorrection": { ... }
  }
}
```

### `tuning.powders[]`

See the **Powders** section above for the full field description.

### `tuning.cartridges[]`

Per-cartridge calibrated scalars. Merged onto the cartridge object at runtime by `loadDb()`.

```json
{
  "id": "CTG_22ARC_cK12",
  "transducerScaleFactor": 0.71715,
  "gradientBetaScale": 1.0
}
```

| Field | Description |
|-------|-------------|
| `transducerScaleFactor` | Conformal piezo transducer scaling factor. Aligns breech pressure output with SAAMI copper crusher equivalents. Default `1.0`. Calibrated from published pressure data per cartridge |
| `gradientBetaScale`? | Beta scaling factor for the pressure gradient ODE. Adjusts the shape of the pressure curve during barrel travel. Default `1.0`. Only populated when the transducer scale alone gives a poor curve fit |

### `tuning.powderCartridgeCalibrations[]`

Reserved for future per-powder-per-cartridge calibration overrides. Currently empty (`[]`).

### `tuning.velocityCorrection`

A spline-based velocity correction model that maps expansion ratio to a velocity scale factor. Applied after ODE integration to correct for gas-dynamics effects not captured by Noble-Abel.

```json
{
  "modelVersion": "v1.0",
  "globalKnots": [
    { "expansionRatio": 3.8,  "factor": 0.996225 },
    { "expansionRatio": 9.8,  "factor": 1.0088 },
    { "expansionRatio": 11.3, "factor": 1.0114 },
    { "expansionRatio": 12.8, "factor": 1.0226 },
    { "expansionRatio": 14.3, "factor": 1.0405 },
    { "expansionRatio": 15.8, "factor": 1.05 }
  ],
  "cartridgeOverrides": {
    "CTG_223REM_cR82": {
      "factor": 0.9573,
      "confidence": "HIGH",
      "fittedLoads": 1721
    }
  },
  "fittedAt": "2026-07-09T00:00:00Z",
  "validationR2": 0.94
}
```

| Field | Description |
|-------|-------------|
| `modelVersion` | Schema version tag for the velocity correction model |
| `globalKnots[]` | Array of `{ expansionRatio, factor }` knot points for the global piecewise-linear correction curve. `expansionRatio` = barrel volume / chamber volume. `factor` = velocity multiplier |
| `cartridgeOverrides` | Map of `cartridgeId → { factor, confidence, fittedLoads }`. Flat scalar override per cartridge for systematic offsets not explained by expansion ratio alone |
| `cartridgeOverrides[id].factor` | Scalar velocity multiplier for this cartridge |
| `cartridgeOverrides[id].confidence` | `"HIGH"` / `"MEDIUM"` / `"LOW"` — data quality flag |
| `cartridgeOverrides[id].fittedLoads` | Number of loads used to fit this override |
| `fittedAt`? | ISO 8601 timestamp when the model was last fitted |
| `validationR2`? | Global hold-out R² validation score at time of fitting |

---

## Firearms [user-only]

```json
{
  "id": "8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a",
  "nickname": "6.5 Creedmoor Precision Rifle",
  "cartridgeId": "CTG_65CREEDMR_cN12",
  "barrelLengthMm": 660.4,
  "twistRateMm": 203.2,
  "sightOverBoreMm": 48.26,
  "magCoalMm": null,
  "freeboreMm": null,
  "velocityScaleFactor": null,
  "velocityOffsetFps": null,
  "velocityOffsetShotCount": null,
  "velocityOffsetSd": null,
  "velocityOffsetDate": null,
  "velocityOffsetSessionId": null,
  "velocityOffsetFlag": null
}
```

* `barrelLengthMm`? — barrel length in millimeters
* `twistRateMm`? — rifling twist rate in mm per turn (e.g. `203.2` = 1:8", `254.0` = 1:10")
* `sightOverBoreMm`? — scope centerline height above bore centerline in millimeters
* `magCoalMm`? — magazine maximum COAL constraint in mm
* `freeboreMm`? — actual measured freebore for this specific barrel in mm; overrides the SAAMI cartridge spec
* `velocityScaleFactor`? — fitted velocity scale: `sessionMeanVelocityFps / predictedVelocityFps`
* `velocityOffsetFps`? — fitted velocity offset: `sessionMeanVelocityFps − predictedVelocityFps`
* `velocityOffsetShotCount`? — number of shots used to derive the velocity offset
* `velocityOffsetSd`? — standard deviation of shots used for the offset
* `velocityOffsetDate`? — ISO 8601 date string when the offset was saved
* `velocityOffsetSessionId`? — session ID from which the offset was derived
* `velocityOffsetFlag`? — warning string, e.g. `"large offset - verify inputs"`

Dexie index: `id, cartridgeId`

---

## Monte Carlo Saves [user-only]

```json
{
  "id": "MCS_HDES_HIGH_DESERT",
  "name": "High Desert Southwest 6.5 PRC Mule Deer",
  "createdAt": 1750500000000,
  "params": {
    "mv": 2910,
    "bulletWeight": 147,
    "bulletDiam": 0.264,
    "bc": 0.351,
    "bcType": "G7",
    "twist": 8,
    "sightHeight": 1.75,
    "zeroDist": 100,
    "zeroOffsetX": 0,
    "zeroOffsetY": 0,
    "cantDegrees": 0,
    "temp": 78,
    "pressure": 24.71,
    "humidity": 15,
    "altitude": 5500,
    "windSpeed": 8,
    "windDir": 90,
    "mvSd": 4.2,
    "bcSd": 0.5,
    "windSpeedSd": 2,
    "windDirSd": 8,
    "windEstimateSd": 1.5,
    "rangeErrorSd": 1,
    "precisionMoa": 0.35,
    "cantSd": 0.3,
    "latitude": 34.5,
    "azimuth": 45,
    "targetShape": "circle",
    "targetWidth": 18,
    "targetHeight": 18,
    "numRuns": 3000,
    "rangeMax": 500,
    "rangeStep": 50
  }
}
```

* `createdAt` — Unix timestamp in milliseconds
* `params.mv` — muzzle velocity in **fps**
* `params.bulletDiam` — bullet diameter in **inches**
* `params.twist` — twist rate in **inches** per turn
* `params.sightHeight` — scope height above bore in **inches**
* `params.zeroDist` — zero distance in **yards**
* `params.temp` — temperature in **°F**
* `params.pressure` — station pressure in **inHg**
* `params.altitude` — altitude in **feet**
* `params.windSpeed` — wind speed in **mph**
* `params.targetShape` — `"circle"`, `"rectangle"`, or `"ipsc"`
* `params.targetWidth` / `params.targetHeight` — target dimensions in **inches**
* `params.rangeMax` — maximum range in **yards**

Dexie index: `id`

---

## Marked Targets [user-only]

```json
{
  "id": "MKT_HDES_100YD",
  "name": "High Desert 100yd Paper",
  "targetDistance": 100,
  "distanceUnits": "yards",
  "createdAt": 1750500000000
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
  "markedTargetId": "MKT_HDES_100YD",
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

* `markedTargetId`? — references `markedTargets` collection
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
  "sessionId": "MKT_HDES_100YD",
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
* `scale.p1` / `scale.p2` — pixel coordinates of the two scale reference points
* `scale.distance` — physical distance value between the two points
* `scale.pixelsPerUnit` — derived scale factor

Dexie index: `id, sessionId`

---

## Groups [user-only]

```json
{
  "id": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "sessionId": "MKT_HDES_100YD",
  "targetId": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "groupNum": 1,
  "poa": { "x": 450.5, "y": 312.0 },
  "color": "#ef4444"
}
```

* **Note on sessionId**: Holds the ID of the linked **markedTargets** record
* `groupNum`? — order index of group within the marked target
* `poa` — pixel coordinates of the point of aim on the target image
* `color` — hex color string used to render this group

Dexie index: `id, sessionId`

---

## Shots [user-only]

```json
{
  "id": "d5c4b3a2-9e8d-7c6b-5a4b-3c2d1e0f9a8b",
  "groupId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "sessionId": "MKT_HDES_100YD",
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
* `x` / `y` — physical offset from POA in `units` (inches or cm). Y positive = up
* `px` / `py` — raw pixel coordinates on the canvas
* `velocity` — muzzle velocity in **fps**, written from either the Marking page manual input or the Chrono import workflow. `null` if not recorded

Dexie index: `id, groupId, sessionId, targetId`

---

## Target Images [user-only]

```json
{
  "id": "f2b4c6d8-0e2a-4b6c-8d0e-2a4b6c8d0e2a",
  "name": "100yd Target",
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

## Chrono Sessions [user — IndexedDB only]

Chronograph import records created by the Chrono page. **Never serialised to `local-db.json`.**

```json
{
  "id": "CHRN_lf7q3x9p2k",
  "name": "Range Day — LabRadar Series 4",
  "deviceType": "labradar",
  "importedAt": 1781546283422,
  "shots": [
    {
      "shotNumber": 1,
      "velocityFps": 2743.25,
      "timestamp": "0.000000",
      "linkedGroupId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
      "linkedShotIndex": 0
    },
    {
      "shotNumber": 2,
      "velocityFps": 2738.10,
      "linkedGroupId": null,
      "linkedShotIndex": null
    }
  ]
}
```

* `deviceType` — detected chronograph type. Values: `"labradar"`, `"magnetospeed"`, `"garmin_xero"`, `"generic"`
* `importedAt` — Unix timestamp in milliseconds
* `shots[].shotNumber` — shot sequence number from the chrono export file
* `shots[].velocityFps` — muzzle velocity in **fps** (normalized from m/s or mm/s during import)
* `shots[].timestamp`? — original time value from chrono CSV if available (seconds elapsed string)
* `shots[].linkedGroupId`? — ID of the marking `Group` record this chrono shot is associated with
* `shots[].linkedShotIndex`? — 0-based index within the group's shot list

Dexie index: `id`

---

*For the mathematical derivations behind the simulation engines, see [MATHS.md](MATHS.md).*
*For the user-facing guide, see [README.md](README.md).*
