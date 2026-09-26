# Reference Database Schema

The shape of the curated component library: every table, every field, its type, whether it is
required, its unit, and what it means.

Sections 1–4 are the reference library: the curated component tables, which are the whole of
what ships. Section 5 is the other half of the same database — the user's own firearms,
loads, marked targets, sessions, groups, shots, chronograph strings and saved scenarios.
Those are personal records rather than library data, they are never shipped and never
arrive by sync, but they live in the same store and reference the library by id, so they
are described here too.

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

The user's own tables sit in the same Dexie store and are described in §5.

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

## 5. Session data

Everything the user creates: rifles, recipes, the paper they marked, the string they
chronographed, the session that ties those together, and the scenarios they saved. It shares
the database with the library and points into it by id, but it is not library data — the
master payload carries none of these records and a sync never writes one.

### 5.1 Tables

| Table | Holds |
|---|---|
| `firearms` | The user's rifles |
| `loads` | Recipes — a cartridge, a bullet, and for a handload the charge and seating |
| `targetImages` | Photographs or scans of shot paper, plus their binary |
| `customTargets` | Printable target presets — geometry for the target generator, no shot data |
| `markedTargets` | One marking sitting: the paper as it was scored |
| `sessionTargets` | One image placed in a sitting, with the scale that turns pixels into inches |
| `groups` | One group marked on one of those images |
| `shots` | One round fired — marked, chronographed, or both |
| `sessions` | A range session: rifle + load + marked target + environment |
| `chronoSessions` | An imported chronograph string, header only |
| `monteCarloSaves` | Saved hit-probability scenarios |

`analysisQueues` is **not** a table. The analysis page keeps its queues in `localStorage`
under `ep_analysis_queues`; the `AnalysisQueue` type is declared beside the others but never
reaches Dexie.

### 5.2 Identifiers

Library ids carry the prefixes in §3. Session ids mostly do not: a record the user creates
gets a bare `generateUniqueId()` with no prefix at all, and that absence is load-bearing.
`syncFromMasterData` prunes a table by deleting the ids that start with that table's prefix
and are missing from the payload, so an unprefixed id is what makes the user's data
unreachable by a sync.

| Prefix | Table |
|---|---|
| `CHRN_` | chronoSessions |
| `MCS_` | monteCarloSaves |
| *(none)* | firearms, loads, targetImages, customTargets, markedTargets, sessionTargets, groups, shots |

`MCS_` is a prefix the user's own records carry, which is exactly why `monteCarloSaves` is
absent from the prune list — pruning on it would delete the scenarios it was meant to
protect. Anything given a prefix later must be kept out of that list for the same reason.

### 5.3 Units — the session side is not metric

§2 governs the library. It does not govern here. Session records store the number the user
typed, in the unit the field was labelled with, and convert on the way out:

| Field | Stored as |
|---|---|
| `shots.velocity` | feet per second |
| `firearms.velocityOffsetFps`, `velocityOffsetSd` | feet per second |
| `sessions.temp` | degrees Fahrenheit |
| `sessions.pressure` | inches of mercury |
| `sessions.altitude` | feet |
| `sessions.targetDistance`, `markedTargets.targetDistance` | whatever `distanceUnits` says |
| `shots.x`, `shots.y` | whatever the owning `sessionTargets.scale.units` says |
| `monteCarloSaves.params.*` | the simulator's own units — fps, inches, yards, MOA, mph, °F, inHg, ft |
| `customTargets` dimensions | inches |

`firearms` and `loads` are the exception and follow §2: they are `…Mm` and `…Grams` in
storage and convert at the form. The split is between records that describe a component or a
rifle, which are metric, and records of an occasion, which keep the observed figure.

Converting happens at the consumer. `chronoSubmission` is the clearest case — it turns fps
into mm/s, °F into millicelsius, inHg into pascals and feet into metres as it builds the
payload, and nothing upstream of it has done any of that.

### 5.4 How the records connect

Three tables carry a field named `sessionId` that does **not** hold a `sessions.id`:

> `sessionTargets.sessionId`, `groups.sessionId` and `shots.sessionId` all hold a
> **`markedTargets.id`**. Marking data is keyed by the sitting that produced it, and a
> sitting exists whether or not a session was ever recorded against it. The link runs the
> other way: `sessions.markedTargetId` points at the sitting.

The rest:

```
markedTargets ──< sessionTargets ──> targetImages
      ^               ^
      │               │ (targetId)
      │           groups ──< shots
      │
sessions ──> firearms, loads
   └──> chronoSessions ──< shots   (shots.chronoSessionId)
```

A `shot` has two independent memberships and needs at least one. Marked-only (an impact you
did not chronograph) and chronographed-only (a fouler at the head of a string) are both
normal, so `groupId`/`targetId` and `chronoSessionId` are each optional. There is one
velocity, on the shot, and it is not copied onto the string.

### 5.5 Field reference

#### 5.5.1 `firearms`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `nickname` | string | required | Auto-generated from the cartridge when left blank |
| `cartridgeId` | string | required | → `cartridges` |
| `barrelLengthMm` | number | optional | Entered in inches |
| `twistRateMm` | number | optional | Millimetres per turn. Overrides the cartridge's reference twist. |
| `sightOverBoreMm` | number | optional | Scope centre above bore axis |
| `magCoalMm` | number | optional | Longest round the magazine accepts |
| `freeboreMm` | number | optional | Measured freebore for **this** barrel; supersedes `cartridges.freeboreLengthMm` |

Plus the trued velocity offset, written by the ignition simulator when a chronographed
session is used to correct that rifle's prediction. All eight fields are set together or not
at all:

| Field | Type | | Meaning |
|---|---|---|---|
| `velocityScaleFactor` | number | optional | measured ÷ predicted |
| `velocityOffsetFps` | number | optional | measured − predicted, fps |
| `velocityOffsetShotCount` | number | optional | Shots the offset was derived from |
| `velocityOffsetSd` | number | optional | SD of those shots, fps |
| `velocityOffsetDate` | string | optional | ISO date the offset was saved |
| `velocityOffsetSessionId` | string | optional | → `sessions` |
| `velocityOffsetTuningStamp` | string | optional | `tuning-db.json`'s `generated_at` at true-ing time. Compared against the loaded fit's; a mismatch means the offset was measured against a different model and is stale. |
| `velocityOffsetFlag` | string | optional | Free text, e.g. "large offset - verify inputs" |

The offset belongs to the rifle rather than to the fit, so it is stored, never fitted. It is
deliberately excluded from a contribution payload.

#### 5.5.2 `loads`

One recipe. The app writes handloads; the factory-load fields are declared and currently
carried by no record.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `name` | string | optional | Display name |
| `handloadName` | string | optional | The generated name, regenerated when the charge changes |
| `cartridgeId` | string | required | → `cartridges` |
| `bulletId` | string | required | → `bullets` |
| `powderId` | string | optional | → `powders`. Required for a handload. |
| `primerId` | string | optional | → `primers` |
| `brassId` | string | optional | → `brass` |
| `chargeWeightGrams` | number | optional | Grams. Entered in grains. |
| `coalMm` | number | optional | Cartridge overall length |
| `cbtoMm` | number | optional | Base to ogive |
| `cbtoComp` | string | optional | Which comparator the CBTO was measured with — the number means nothing without it |
| `shoulderMm` | number | optional | Shoulder bump |
| `shoulderComp` | string | optional | Comparator used for that measurement |
| `firings` | number | optional | Times this brass has been fired |
| `bulletLot` | string | optional | |
| `powderLot` | string | optional | |
| `primerLot` | string | optional | |
| `brassLot` | string | optional | |
| `notes` | string | optional | |

Factory-load fields, declared but unpopulated: `manufacturerId`, `partNumber`, `lot`,
`velocityMps` (advertised velocity, read by `hitProbabilityAnalysis` when no chronograph
figure exists), `isCommercial` and `loadTypeId`. `lot` is read once, as a fallback source for
`bulletLot`.

#### 5.5.3 `targetImages`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `name` | string | required | Filename with the extension stripped, de-duplicated with a `(n)` suffix |
| `imageBlob` | Blob | required | The image itself |
| `size` | string | required | Human-readable size, e.g. `"482 KB"`. Display text, not a number. |
| `firearmId` | string | optional | → `firearms` |
| `loadId` | string | optional | → `loads` |
| `customTargetConfig` | CustomTarget | optional | Embedded copy of the preset the paper was printed from |

`imageBlob` is the one field JSON cannot carry. Export writes it as a Base64 `dataUrl` and
import converts it back; a record on disk has `dataUrl` and no `imageBlob`, and a record in
the database has the reverse.

#### 5.5.4 `customTargets`

A printable target preset. Geometry only — no shots, no session. Dimensions are inches.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `name` | string | required | |
| `paperSize` | string | optional | |
| `orientation` | string | optional | |
| `gridEnabled` | boolean | optional | |
| `gridSize` | number | optional | |
| `gridColor` | string | optional | CSS colour |
| `rows`, `cols` | number | optional | Bullseyes down and across the sheet |
| `marginX`, `marginY` | number | optional | |
| `shape` | string | optional | `circle` and friends |
| `diameter` | number | optional | Bullseye diameter |
| `numRings` | number | optional | |
| `bullseyeColor`, `ringColorA`, `ringColorB` | string | optional | |
| `labelText`, `labelPosition`, `labelSize`, `labelMargin` | | optional | Printed caption |

#### 5.5.5 `markedTargets`

The sitting itself, and almost nothing else. Its substance is in `sessionTargets`, `groups`
and `shots`, all of which point back at its id.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | The value the three child tables store in their `sessionId` |
| `name` | string | optional | |
| `targetDistance` | number | optional | Required by the UI before a sitting can be saved — MOA, MIL and every trajectory figure depend on it |
| `distanceUnits` | string | optional | `yards` or `meters` |
| `createdAt` | number | optional | Epoch milliseconds |

#### 5.5.6 `sessionTargets`

One image placed in one sitting, and the scale for it. Re-saving a sitting deletes and
rewrites every row that belongs to it.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `sessionId` | string | required | → `markedTargets` (see §5.4) |
| `targetImageId` | string | required | → `targetImages` |
| `scale.p1`, `scale.p2` | `{x, y}` \| null | required | The two pixel points the user clicked on a known distance |
| `scale.distance` | number \| null | required | Real distance between them |
| `scale.units` | string | required | `in` or `mm`. Becomes `shots.units`. |
| `scale.pixelsPerUnit` | number \| null | required | Derived from the three above. Null means unscaled, and a sitting will not save while any target is. |
| `transform.scale` | number | required | Canvas zoom. A view setting, not a measurement. |

#### 5.5.7 `groups`

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `sessionId` | string | required | → `markedTargets` (see §5.4) |
| `targetId` | string | required | → `sessionTargets` |
| `groupNum` | number | optional | Numbered across the whole sitting, not per image |
| `poa` | `{x, y}` \| null | required | Point of aim, **in pixels**. Every impact is measured from it, so a group without one is dropped at save. |
| `color` | string | required | Cycled from a fixed palette so groups stay distinguishable |

#### 5.5.8 `shots`

One round fired. Marked, chronographed, or both — see §5.4.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `velocity` | number \| null | required | Feet per second, or null. The only copy; the chronograph string does not hold one. |

Marked-target membership, absent when the shot was not marked:

| Field | Type | | Meaning |
|---|---|---|---|
| `groupId` | string | optional | → `groups` |
| `sessionId` | string | optional | → `markedTargets` (see §5.4) |
| `targetId` | string | optional | → `sessionTargets` |
| `shotNumber` | number | optional | Order the impacts were marked in |
| `x`, `y` | number | optional | Offset from the group's POA in `units`. **Y is inverted at save** so positive is up. |
| `px`, `py` | number | optional | Raw pixel position on the image. Kept so the mark can be redrawn without the scale. |
| `units` | string | optional | Copied from the target's `scale.units` |

Chronograph membership, absent when the shot was not chronographed:

| Field | Type | | Meaning |
|---|---|---|---|
| `chronoSessionId` | string | optional | → `chronoSessions` |
| `firingOrder` | number | optional | Position in the string as the device recorded it |
| `timestamp` | string | optional | As reported by the device |

`firingOrder` and `shotNumber` are two different orderings and nothing may assume they line
up: one is the order rounds left the barrel, the other the order impacts were clicked.

Two type guards narrow the union — `isMarkedShot` (has a group and coordinates) and
`isChronographedShot` (has a string and a numeric velocity). Query by group or by string and
every row satisfies the matching guard by construction, but the type cannot know that, so
filter once at the boundary rather than null-checking downstream.

#### 5.5.9 `sessions`

What was fired, out of what, at what, in what weather.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | |
| `name` | string | optional | |
| `markedTargetId` | string | optional | → `markedTargets`. The paper this session produced. |
| `firearmId` | string | optional | → `firearms` |
| `loadId` | string | optional | → `loads` |
| `targetDistance` | number | optional | Copied from the marked target at save |
| `distanceUnits` | string | optional | `yards` or `meters` |
| `temp` | number | optional | °F |
| `altitude` | number | optional | Feet |
| `pressure` | number | optional | inHg |
| `pressureType` | string | optional | `station` (absolute) or `sea` (corrected to sea level). The number is unusable without it. |
| `chronoSessionId` | string | optional | → `chronoSessions`. One string measuring this session's load. |
| `roundCount` | number | optional | Approximate rounds through the barrel when this was fired |

`chronoSessionId` is the direct association: without it, velocity could reach a session only
through marked paper, leaving a chronographed load with nothing marked unable to carry its
own numbers. One string per session matches how the devices work — a Garmin, LabRadar or
MagnetoSpeed file is normally one charge weight. A ladder fired as a single uninterrupted
string is the case it does not cover.

`roundCount` sits on the session and not on the firearm on purpose. "This string was fired at
~640 rounds" is a fact about an occasion and never changes; "this rifle has 640 rounds" is a
moving number that would re-hash every measurement that rifle ever produced each time it
ticked.

The `[firearmId+loadId]` compound index exists because "every session of this combination" is
the query the analysis and contribution paths are built on.

#### 5.5.10 `chronoSessions`

A header, and nothing else.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | `CHRN_`-prefixed |
| `name` | string | required | |
| `deviceType` | string | required | `labradar`, `magnetospeed`, `garmin_xero`, `fit` or `generic` |
| `importedAt` | number | required | Epoch milliseconds |

Its shots are rows in `shots` carrying this id, not an array inside it. Embedding them made
the velocity a second copy of the one on the shot, and every defect that followed — a
hand-edited value diverging from the copy the simulator actually read, links stored as an
ordinal into a sorted group, an auto-match assuming the two sequences aligned — came from
that duplication.

#### 5.5.11 `monteCarloSaves`

A saved hit-probability scenario: the whole input form, verbatim, in the simulator's own
units.

| Field | Type | | Meaning |
|---|---|---|---|
| `id` | string | required | `MCS_`-prefixed |
| `name` | string | required | Matched case-insensitively on save, so re-saving under an existing name overwrites it |
| `createdAt` | number | required | Epoch milliseconds |
| `params` | object | required | ~35 fields, every one `number \| null` unless noted |

`params` groups into projectile (`mv`, `bulletWeight`, `bulletDiam`, `bc`, `bcType` as
`'G1' | 'G7'`, `twist`), sight and zero (`sightHeight`, `zeroDist`, `zeroOffsetX/Y`,
`cantDegrees`), atmosphere (`temp`, `pressure`, `humidity` as 0–100, `altitude`, `windSpeed`,
`windDir`), the dispersion terms the simulation samples (`mvSd`, `bcSd`, `windSpeedSd`,
`windDirSd`, `windEstimateSd`, `rangeErrorSd`, `precisionMoa`, `cantSd`), Coriolis
(`latitude`, `azimuth`), the target (`targetShape` as `'circle' | 'rectangle' | 'ipsc'`,
`targetWidth`, `targetHeight`) and the run (`numRuns`, `rangeMax`, `rangeStep`).

Null means the field was left blank, which is not zero. `mv` and `rangeMax` are refused as
blank at run time rather than defaulted.
