# Empirical Precision — Database Schema Reference

This document is the canonical technical reference for all IndexedDB tables and the `master-db.json` structure.

**Convention:** All field names use **camelCase**. All IDs are strings. Optional fields are marked with `?`.

Tables marked **[master-db]** are seeded from the open-source library. Tables marked **[user-only]** are created locally and never appear in `master-db.json`.

---

## Manufacturers [master-db]

```json
{
  "id": "MAN_HODGDON_H7X2",
  "name": "Hodgdon",
  "type": ["powder"]
}
```

- `type` — array of strings. Valid values: `bullet`, `powder`, `primer`, `brass`, `ammo`

Dexie index: `id`

---

## Diameters [master-db]

```json
{
  "id": "DIA_308_XBM7",
  "imperial": ".308",
  "metric": "7.62mm"
}
```

Dexie index: `id`

---

## Cartridges [master-db]

```json
{
  "id": "CTG_17HRN_N1H3",
  "name": "17 Hornet",
  "diameterId": "DIA_172_QW2R",
  "maxCaseLengthMm": 34.29,
  "trimLengthMm": 34.036,
  "oalMm": 43.7642,
  "maxSaamiPa": 344737864.66,
  "baseCapacityH2oGrams": 0.77758692,
  "boreDiameterMm": 4.2672,
  "bulletDiameterMm": 4.3688,
  "burnRateMultiplier": 0.1,
  "primerPocketId": "PKT_SML"
}
```

- All length and diameter fields are in millimeters
- `maxSaamiPa` — SAAMI maximum average pressure in Pascals (Pa)
- `baseCapacityH2oGrams` — case water capacity in grams of H2O
- `boreDiameterMm` — land-to-land bore diameter in millimeters (NOT groove/bullet diameter). Source: SAAMI spec sheets
- `bulletDiameterMm` — nominal projectile diameter in millimeters
- `burnRateMultiplier` — scaling factor for internal ballistics simulator burn rates
- `primerPocketId` — references `primerPockets` collection. `"PKT_SML"` for small rifle, `"PKT_LRG"` for large rifle, or omitted/null for rimfire

Dexie index: `id, diameterId`

---

## Bullets [master-db]

```json
{
  "id": "BUL_SIERRA_308_175_HPBT_A2D1",
  "manufacturerId": "MAN_SIERRA_S6Y3",
  "diameterId": "DIA_308_XBM7",
  "name": "HPBT MatchKing",
  "physis": {
    "weightGrams": 11.34,
    "overallLengthMm": 35.0,
    "ogiveLengthMm": 17.97,
    "boatTailLengthMm": 4.7,
    "tipLengthMm": null,
    "meplatDiameterMm": 1.5,
    "bearingSurfaceMm": 12.33
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

- All length and diameter fields in `physis` are in millimeters
- `weightGrams` — bullet weight in grams
- `bearingSurfaceMm` — length of the bullet bearing surface in millimeters
- `tipLengthMm` — for tipped bullets: the plastic tip length only in millimeters (not included in aerodynamic metal length)
- `g1FF` / `g7FF` — form factors relative to G1/G7 standard projectile
- `ix` / `iy` — moments of inertia (kg·m²), used by stability calculator when available
- `preferredModel` — `"G1"` or `"G7"`. G7 is preferred for long-range boat-tail rifle bullets

Dexie index: `id, manufacturerId, diameterId`

---

## Powders [master-db]

```json
{
  "id": "PWD_HODG_H4350_R1A6",
  "manufacturerId": "MAN_HODGDON_H3V8",
  "name": "H4350",
  "baCoeff": 0.2285,
  "kCoeff": 1.2311,
  "heatOfExplosionKjKg": 3585,
  "grainId": "GRN_EXTRUDED_SINGLEPERF",
  "solidDensity": 1620,
  "ignitionBa": 0.4857,
  "ignitionBp": 0.1489,
  "ignitionZ1": 0.5175,
  "ignitionZ2": 0.8298,
  "tempSensFactor": 0.002
}
```

- `baCoeff` — ballistic action coefficient (internal ballistics simulator)
- `kCoeff` — burn rate shape coefficient. Single-base (nitrocellulose only): `1.23`. Double-base (NC + nitroglycerin): `1.255`
- `heatOfExplosionKjKg` — heat of explosion in kJ/kg. Single-base: `3580`. Double-base: `3950`
- `grainId` — references `grain` collection (powder grain geometry)
- `solidDensity` — optional solid density of the powder in kg/m³
- `ignitionBa` / `ignitionBp` / `ignitionZ1` / `ignitionZ2` — optional ignition phase burn rate coefficients
- `tempSensFactor` — optional temperature sensitivity factor

Dexie index: `id, manufacturerId`

---

## Grain [master-db]

```json
{
  "id": "GRN_BALL",
  "grainType": "ball"
}
```

- `grainType` — allowed values: `"ball"`, `"flake"`, `"extrudedSinglePerf"`, `"extrudedMultiPerf"`, `"extruded"`

Dexie index: `id`

---

## Primers [master-db]

```json
{
  "id": "PRI_CCI_BR2LRG_C84Z",
  "manufacturerId": "MAN_CCI_C2W7",
  "name": "CCI BR-2 Large Rifle Benchrest",
  "primerPocketId": "PKT_LRG"
}
```

Dexie index: `id, manufacturerId`

---

## Primer Pockets [master-db]

```json
{
  "id": "PKT_SML",
  "name": "SMALL"
}
```

Dexie index: `id`

---

## Primer Holes [master-db]

```json
{
  "id": "HL_059",
  "name": "0.059"
}
```

Dexie index: `id`

---

## Brass [master-db]

```json
{
  "id": "BRS_MAN_LAPUA_L5X1_CTG_308WIN_cC82_TR53",
  "manufacturerId": "MAN_LAPUA_L5X1",
  "cartridgeId": "CTG_308WIN_cC82",
  "primerPocketId": "PKT_LRG",
  "primerHoleId": "HL_080",
  "capacityH2oGrams": 3.6287
}
```

- `capacityH2oGrams` — water capacity in grams of H2O

Dexie index: `id, cartridgeId, manufacturerId`

---

## Load Types [master-db]

Reference table. Two records: `LT_COMM` (commercial) and `LT_HAND` (handload).

```json
{
  "id": "LT_COMM",
  "name": "commercial"
}
```

Dexie index: `id`

---

## Loads [master-db + user]

Commercial and handload records share the same table. Fields differ by type.

### Commercial Load Example
```json
{
  "id": "LOAD_COMM_FEDERAL_308WIN_GMATCH_09RB",
  "loadTypeId": "LT_COMM",
  "name": "Federal Gold Medal Match",
  "cartridgeId": "CTG_308WIN_cC82",
  "bulletId": "BUL_SIERRA_308_175_HPBT_A2D1",
  "manufacturerId": "MAN_FEDERAL_F4G7",
  "partNumber": "GM308M",
  "lot": "Lot2024A",
  "coalMm": 71.12,
  "velocityMps": 792.48,
  "isCommercial": true,
  "notes": "Factory match ammunition"
}
```

### Handload Example
```json
{
  "id": "LOAD_HAND_USER_308WIN_SMK175_X4Y7",
  "loadTypeId": "LT_HAND",
  "name": "308 Win - 175gr SMK - 42.0gr H4350",
  "cartridgeId": "CTG_308WIN_cC82",
  "bulletId": "BUL_SIERRA_308_175_HPBT_A2D1",
  "powderId": "PWD_HODG_H4350_R1A6",
  "primerId": "PRI_CCI_BR2LRG_C84Z",
  "brassId": "BRS_MAN_LAPUA_L5X1_CTG_308WIN_cC82_TR53",
  "chargeWeightGrams": 2.7216,
  "coalMm": 71.12,
  "cbtoMm": 56.40,
  "velocityMps": 792.48,
  "isCommercial": false,
  "notes": "Seating depth ladder, node 3"
}
```

**Commercial loads** typically have: `loadTypeId`, `name`, `cartridgeId`, `bulletId`, `manufacturerId`, `partNumber`, `lot`, `isCommercial: true`

**Handloads** typically have: `loadTypeId`, `cartridgeId`, `bulletId`, `powderId`, `primerId`, `brassId`, `chargeWeightGrams`, `coalMm`, `cbtoMm`, `velocityMps`, `isCommercial: false`

Dexie index: `id, cartridgeId, bulletId, powderId`

---

## Firearms [user-only]

```json
{
  "id": "8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a",
  "nickname": "308 Win Tactical Rifle",
  "cartridgeId": "CTG_308WIN_cC82",
  "barrelLengthMm": 609.6,
  "twistRateMm": 254.0,
  "sightOverBoreMm": 48.26
}
```

- `barrelLengthMm` — barrel length in millimeters
- `twistRateMm` — rifling twist rate in millimeters per turn (e.g., `203.2` means 1 turn in 203.2mm, which is a 1:8" twist)
- `sightOverBoreMm` — scope centerline height above bore centerline in millimeters

Dexie index: `id, cartridgeId`

---

## Sessions [user-only]

One record per range visit.

```json
{
  "id": "c8b3d6f1-4e8a-4d7a-8b9c-2d3e4f5a6b7c",
  "name": "100yd H4350 Ladder",
  "timestamp": "2026-05-30T12:00:00.000Z",
  "firearmId": "8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a",
  "loadId": "LOAD_HAND_USER_308WIN_SMK175_X4Y7",
  "targetDistance": 100,
  "distanceUnits": "yards"
}
```

Dexie index: `id, firearmId, loadId, [firearmId+loadId]`

The compound index `[firearmId+loadId]` enables efficient queries like "all sessions for this exact firearm+load combination."

---

## Session Targets [user-only]

Links a target image to a session and stores the scale calibration.

```json
{
  "id": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "sessionId": "c8b3d6f1-4e8a-4d7a-8b9c-2d3e4f5a6b7c",
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

- `p1` / `p2` — pixel coordinates of the two scale reference points on the canvas
- `pixelsPerUnit` — derived from the p1/p2 distance and the known physical distance

Dexie index: `id, sessionId`

---

## Groups [user-only]

One record per group of shots within a session.

```json
{
  "id": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "sessionId": "c8b3d6f1-4e8a-4d7a-8b9c-2d3e4f5a6b7c",
  "targetId": "e3a5f7d2-1c9b-4a8d-8e7f-3a2b1c0d9e8f",
  "groupNum": 1,
  "poa": { "x": 450.5, "y": 312.0 },
  "color": "#ef4444"
}
```

- `poa` — pixel coordinates of the point of aim on the canvas
- `color` — hex color used to render this group on the composite plot

Dexie index: `id, sessionId`

---

## Shots [user-only]

One record per individual bullet impact.

```json
{
  "id": "d5c4b3a2-9e8d-7c6b-5a4b-3c2d1e0f9a8b",
  "groupId": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
  "sessionId": "c8b3d6f1-4e8a-4d7a-8b9c-2d3e4f5a6b7c",
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

- `x` / `y` — physical offset from POA in `units` (inches or cm)
- `px` / `py` — pixel coordinates on the canvas
- `velocity` — muzzle velocity in fps, or `null` if not recorded

Dexie index: `id, groupId, sessionId, targetId`

---

## Target Images [user-only]

```json
{
  "id": "f2b4c6d8-0e2a-4b6c-8d0e-2a4b6c8d0e2a",
  "name": "100yd Target - Jun 2026",
  "timestamp": "2026-06-01T14:30:00.000Z",
  "imageBlob": "<Blob>",
  "size": "245 KB",
  "firearmId": "8f3b6c1d-2d4a-4e8a-8c7a-9b5d8f6c3e2a",
  "loadId": "LOAD_HAND_USER_308WIN_SMK175_X4Y7"
}
```

- `imageBlob` — stored as a native binary `Blob` in IndexedDB for memory efficiency
- **JSON export/import:** `imageBlob` is automatically converted to/from a Base64 `dataUrl` string at the export/import boundary. The JSON field name in exports/imports is `dataUrl`.

Dexie index: `id`

---

## Custom Targets [user-only]

Stores target generator configuration for reconstruction.

```json
{
  "id": "b7c9e1d3-4f0a-4b6c-8d2e-4a6b8c0d2e4f",
  "name": "100yd IPSC Target",
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
  "labelText": "308 Win Match",
  "labelPosition": "bottom",
  "labelSize": 12,
  "labelMargin": 0.25
}
```

Dexie index: `id`

---

## ID Naming Conventions

IDs in `master-db.json` follow a human-readable compound key pattern:

| Table | Pattern | Example |
|-------|---------|---------|
| Manufacturers | `MAN_<NAME>_<HASH>` | `MAN_HODGDON_H7X2` |
| Diameters | `DIA_<IMPERIAL>_<HASH>` | `DIA_308_XBM7` |
| Cartridges | `CTG_<NAME>_<HASH>` | `CTG_308WIN_cC82` |
| Bullets | `BUL_<MAN>_<DIA>_<WEIGHT>_<NAME>_<HASH>` | `BUL_SIERRA_308_175_HPBT_A2D1` |
| Powders | `PWD_<MAN>_<NAME>_<HASH>` | `PWD_HODG_H4350_R1A6` |
| Primers | `PRI_<MAN>_<NAME>_<HASH>` | `PRI_CCI_BR2LRG_C84Z` |
| Brass | `BRS_MAN_<MAN_HASH>_CTG_<CTG_HASH>_<HASH>` | `BRS_MAN_ADG_A4D7_CTG_65CM_cN72_TR53` |
| Loads | `LOAD_<TYPE>_<MAN>_<CTG>_<NAME>_<HASH>` | `LOAD_COMM_FEDERAL_308WIN_GMATCH_09RB` |
| Primer Pockets | `PKT_<SIZE>` | `PKT_SML`, `PKT_LRG` |
| Primer Holes | `HL_<SIZE>` | `HL_059`, `HL_080` |

User-created records (firearms, sessions, groups, shots, etc.) use UUID-style random strings generated by the app.
