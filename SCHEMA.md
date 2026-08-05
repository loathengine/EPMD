# Reference Database Schema

The shape of the curated component library: every table, every field, its type, whether it is
required, its unit, and what it means.

Scope is the reference library only. The component tables below are the whole of it. A
working copy may also carry a user's own firearms, loads, sessions, marked targets, groups,
shots, target images, custom targets and saved simulations; those are personal records, not
part of the library, and are not described here.

---

## 1. Tables

| Table | Holds |
|---|---|
| `manufacturers` | Makers of bullets, powder, primers and brass |
| `diameters` | Calibre definitions — bore and bullet diameter |
| `grainTypes` | Propellant kernel geometries |
| `primerPockets` | Pocket sizes |
| `primers` | Primers, keyed to a pocket size |
| `cartridges` | Chamberings: case dimensions and pressure ceilings |
| `bullets` | Projectiles: geometry, material and drag |
| `powders` | Propellants: densities, burn parameters and kernel type |
| `brass` | Cases, one record per manufacturer and cartridge |

## 2. Units

Every stored value is SI/metric. Imperial units exist only at the display boundary. The
suffix on a field name is its unit and is load-bearing:

| Suffix | Unit | Suffix | Unit |
|---|---|---|---|
| `…Mm` | millimetres | `…Pa` | pascals |
| `…Grams` | grams | `…KgM3` | kilograms per cubic metre |
| `…J` | joules | `…KjKg` | kilojoules per kilogram |

`capacityH2oGrams` and `baseCapacityH2oGrams` are **grams of water**, not grains.

## 3. Identifiers

Every record has a string `id`. Each table has its own prefix, so an id is self-describing
and unique across the whole library.

| Prefix | Table |
|---|---|
| `MAN_` | manufacturers |
| `DIA_` | diameters |
| `CTG_` | cartridges |
| `BUL_` | bullets |
| `PWD_` | powders |
| `PRI_` | primers |
| `PKT_` | primerPockets |
| `BRS_` | brass |
| *(none)* | grainTypes |

`grainTypes` is the exception: its ids are bare words (`ball`, `flake`,
`extrudedSinglePerf`, `extrudedMultiPerf`, `extruded`) because `powders.grainType` stores the
string itself rather than a prefixed key.

**An id is permanent.** Records are matched by id, so changing one does not rename a record —
it destroys the old one and creates a new one, orphaning every reference to it.

Any field ending `…Id` is a foreign key to the `id` of the table its name identifies.

---

## 4. Field reference

**Required** means every record carries a non-null value. **Optional** means some records do
not; absence always means *unknown*, never zero.

### 4.1 `manufacturers`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `name` | string | required | |
| `type` | string[] | optional | Which component classes this maker produces |

### 4.2 `diameters`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `metricName` | string | required | Millimetre designation |
| `imperialName` | string | required | Inch designation |
| `bulletDiameterMm` | number | required | Groove diameter — what the projectile measures |
| `boreDiameterMm` | number | required | Land-to-land diameter, always the smaller of the two |

Both are needed and are not interchangeable: bore diameter sets the area pressure acts on,
bullet diameter sets what the rifling engraves.

### 4.3 `grainTypes`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | The value stored in `powders.grainType` |
| `name` | string | required | Display name |

### 4.4 `primerPockets`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | `PKT_SML` or `PKT_LRG` |
| `name` | string | required | |

The table exists to be the target of `primerPocketId` on cartridges, brass and primers. It
holds no dimensions.

### 4.5 `primers`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `manufacturerId` | string | required | → `manufacturers` |
| `name` | string | required | |
| `primerPocketId` | string | required | → `primerPockets`. Determines which cases the primer fits. |
| `brisanceEnergyJ` | number | optional | Ignition energy. Absent means no measured figure. |

### 4.6 `cartridges`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `name` | string | required | |
| `diameterId` | string | required | → `diameters`. The only route to bore and bullet diameter. |
| `primerPocketId` | string | required | → `primerPockets`. The standard pocket for the chambering; a specific case may differ. |
| `baseCapacityH2oGrams` | number | required | Nominal case capacity, used when no specific case is known |
| `maxCaseLengthMm` | number | required | Maximum case length |
| `trimLengthMm` | number | required | Trim-to length |
| `oalMm` | number | required | Maximum cartridge overall length |
| `bodyDiameterMm` | number | required | External body diameter at the case base |
| `freeboreLengthMm` | number | required | Standard freebore. A measured chamber value supersedes it. |
| `twistRateMm` | number | required | Reference twist, millimetres per turn |
| `refTestBarrelMm` | number \| null | optional | Barrel length the reference velocity was measured in. Absent means the standard test barrel was used. |
| `maxSaamiPa` | number | required | Pressure ceiling |
| `maxCipPa` | number | optional | Alternative pressure ceiling, where one is published |
| `cipTestBarrelMm` | number | optional | Test-barrel length for that ceiling |

### 4.7 `bullets`

A bullet record holds measurements and nothing about where they came from. There is no
quality grade, no provenance note, and no record of one bullet's numbers being derived from
another's — nothing to rank or filter on.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `manufacturerId` | string | required | → `manufacturers` |
| `diameterId` | string | required | → `diameters` |
| `name` | string | required | |
| `physis` | object | required | Physical geometry — §4.7.1 |
| `ballistics` | object | required | Drag — §4.7.2 |

#### 4.7.1 `physis`

| Field | Type | | Meaning |
|---|---|---|---|
| `weightGrams` | number | required | |
| `materialType` | string | required | `MAT_JACKETED_LEAD`, `MAT_MONOLITHIC_COPPER`, `MAT_CAST_LEAD` or `MAT_RELIEF_GROOVED_COPPER_MONO` |
| `engravingPressurePa` | number \| null | optional | Pressure required to engrave the rifling. Absent on most records, where the engine applies its own default. |
| `overallLengthMm` | number \| null | optional | Tip to base |
| `ogiveLengthMm` | number \| null | optional | |
| `boatTailLengthMm` | number \| null | optional | `0` means a flat base, which is not the same as `null` for unknown |
| `bearingSurfaceMm` | number \| null | optional | Full-diameter shank length |
| `tipLengthMm` | number | optional | Polymer tip only; absent means the bullet has no tip |

The four `number | null` fields carry the key on every record even when the value is unknown,
so testing for the key's presence does not tell you whether there is a measurement. Test the
value.

#### 4.7.2 `ballistics`

| Field | Type | | Meaning |
|---|---|---|---|
| `g1BC` | number \| null | optional | Drag coefficient against the G1 reference |
| `g7BC` | number \| null | optional | Drag coefficient against the G7 reference |

Unlike `physis`, these keys are **not always present**, and where present may still be
`null`. A test for the key is not a test for a value, and for `g7BC` the two differ widely.
Some records have neither coefficient and cannot be used for trajectory work at all.

Only the drag coefficients are stored. Sectional density and form factor are derived from
weight, diameter and the coefficient, so storing them would be storing a value that is then
recomputed anyway.

### 4.8 `powders`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `manufacturerId` | string | required | → `manufacturers` |
| `name` | string | required | |
| `grainType` | string | required | → `grainTypes`, stored as the bare id |
| `propellantDensityKgM3` | number | required | Density of the solid propellant |
| `bulkDensityKgM3` | number | required | Poured density including the air between kernels; this is what decides whether a charge fits |
| `heatOfExplosionKjKg` | number | required | Energy released per kilogram |
| `burnExponent` | number | optional | Pressure exponent of the burn-rate law |
| `burnSurfacePeakGain` | number | optional | How far the burning surface grows before it begins to shrink |
| `burnFractionAtSurfacePeak` | number | optional | Burn fraction at that peak |
| `burnFractionAtSliverStart` | number | optional | Burn fraction at which the kernel breaks into slivers |

The last three describe the same curve and are populated together: a record has all three or
none of them.

**No burn-rate coefficient is stored here.** It is a fitted parameter and belongs with the
rest of the fitted parameters, keyed by powder id.

**Every powder in the library has a fitted entry.** A powder without one cannot be simulated,
so it does not belong here. Adding a powder means adding load data for it and refitting, not
adding a record and hoping.

### 4.9 `brass`

One record per manufacturer and cartridge.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | Encodes the manufacturer and cartridge it belongs to |
| `manufacturerId` | string | required | → `manufacturers` |
| `cartridgeId` | string | required | → `cartridges` |
| `primerPocketId` | string | required | → `primerPockets`. **May differ from the cartridge's standard pocket** — this is how small-pocket variants of large-pocket chamberings are represented. |
| `capacityH2oGrams` | number | optional | Measured water capacity. Frequently absent, and one of the most consequential values in the library: absence means unknown, and a consumer must fall back to the cartridge's nominal capacity rather than read it as an empty case. |

---

## 5. Fields declared but never populated

The type model declares several fields that no record carries. They are listed so that code
written from the types alone knows to expect nothing.

| Table | Field | |
|---|---|---|
| `cartridges` | `bulletDiameterMm`, `boreDiameterMm` | Both live on the `diameters` record; resolve through `diameterId` |
| `cartridges` | `transducerScaleFactor`, `gradientBetaScale` | Fitted parameters, held with the fit rather than with the component |
| `powders` | `burnAreaCoeff`, `burnAreaFillSlope`, `burnAreaBoreSlope`, `burnAreaExpansionSlope`, `energyScaleFactor` | Fitted parameters, held with the fit |
| `powders` | `cartridgeOverrides` | Superseded by a fitted cartridge-and-powder pair factor |
| `powders` | `heatConvention` | Every record uses one convention |
| `bullets` | `physis.meplatDiameterMm` | Not measured |
| `brass` | `primerHole` | Not measured |
| `primers` | `type` | Superseded by `primerPocketId` |
| `manufacturers` | `displayName` | Consumers fall back to `name` |

Populating any of the fitted-parameter fields above would put a second copy of a calibrated
value into the library, which is the one thing the reference database must never hold.
