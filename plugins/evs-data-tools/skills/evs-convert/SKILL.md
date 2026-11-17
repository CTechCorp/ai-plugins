---
name: evs-convert
description: >-
  Procedures for converting between EVS data file formats. Use when converting PGF to LSDV,
  LSDV to LPDV, GEO to GMF, APDV to AIDV, or vice versa. Also covers lithology-to-stratigraphy
  conversion (PGF/LSDV to GEO/GMF) including geologic hierarchy construction. Covers lossless,
  lossy, and interpretive conversions, data fidelity warnings, and post-conversion validation.
  Also use when the user asks about format differences, migration from legacy formats, or which
  conversion path to take.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Data File Conversions

Prerequisite: Load `/evs-data` first for shared rules. Before performing any conversion,
also load the **source format skill** and the **target format skill** (e.g., `/evs-pgf`
and `/evs-lsdv` for a PGF-to-LSDV conversion).

## General Conversion Rules

These rules apply to all conversions:

- **Numeric precision** — follow the precision rules in `/evs-data`. Never round source
  values except to clean floating-point artifacts.
- **Coordinate mode** — preserve `Elevation` or `Depth` from the source unless the target
  format requires a specific mode (GMF is elevation-only). If conversion between modes is
  needed, confirm with the user.
- **Coordinate units** — must match between source and target. If they differ, stop and
  ask the user.
- **Material IDs** — when the target format requires material IDs, preserve the source IDs
  if possible. If the target format needs new IDs assigned, prompt the user for ordering
  per the material ID ordering guidance in the target format skill.
- **Generation metadata comments** — required. Document the source file, conversion type,
  date, and any data lost or assumptions made.
- **No unsolicited conversions** — never suggest or offer a conversion. Only perform a
  conversion when the user explicitly requests it. You may list available conversions if
  the user asks what formats are possible from a given source.
- **No unsolicited subsetting** — never offer to subset or filter the data. Convert all
  data. If the user wants to exclude specific records, they must request it explicitly.
- **Output verification** — always show the complete output to the user before writing.

## Conversion Reference

| Source | Target | Fidelity | Key Decisions |
|--------|--------|----------|---------------|
| PGF | LSDV | Lossless | None |
| LSDV | PGF | Lossy | Gap handling, non-surface starts |
| LSDV | LPDV | Lossy | Point placement method |
| LPDV | LSDV | Assumptions | Interval derivation method |
| PGF | LPDV | Lossy | Composite (PGF to LSDV to LPDV) |
| APDV | AIDV | Assumptions | Max gap, interval derivation |
| AIDV | APDV | Lossy | Point placement method |
| LPDV | APDV | Conditional | Exactly 2 materials required, indicator 0/1 |
| LSDV | APDV | Conditional | Composite (LSDV to LPDV to APDV), 2 materials |
| PGF | APDV | Conditional | Composite (PGF to LPDV to APDV), 2 materials |
| GEO | APDV | Conditional | Composite (GEO to LSDV to LPDV to APDV), 2 materials |
| PGF | GEO | Interpretive | Hierarchy construction, simplification, user decisions |
| LSDV | GEO | Interpretive | Hierarchy construction, simplification, user decisions |
| PGF | GMF | Composite (PGF → GEO → GMF) | Same as PGF → GEO |
| LSDV | GMF | Composite (LSDV → GEO → GMF) | Same as LSDV → GEO |
| GEO | GMF | Lossless | Depth-to-elevation if needed |
| GMF | GEO | Conditional | Coverage analysis, short/missing handling |
| GEO | LSDV | Lossless | None |
| GEO | PGF | Lossless | None |
| GEO | LPDV | Lossy | Composite (GEO to LSDV to LPDV) or direct refine |

## Lithology Conversions

### PGF to LSDV (Lossless)

**Fidelity:** Lossless — all data is preserved.

**When to use:** Migrating from legacy PGF to the modern LSDV format.

**Prerequisite skills:** `/evs-pgf`, `/evs-lsdv`

**Procedure:**

1. Parse the PGF file: header message (Line 1), declaration (Line 2), row count (Line 3),
   data rows.
2. Extract coordinate mode, material definitions, and units from Line 2.
3. Build the LSDV header:
   - Same coordinate mode (`Elevation` or `Depth`)
   - Convert PGF material definitions to LSDV quoted format: `ID|Name` becomes `"ID|Name"`
   - Same units
4. Group PGF data rows by boring name.
5. For each boring, iterate through consecutive pairs of points to produce LSDV intervals:

   For pair (Point[i], Point[i+1]):
   - X Top = Point[i] X
   - Y Top = Point[i] Y
   - Z Top = Point[i] Z
   - X Bottom = Point[i+1] X
   - Y Bottom = Point[i+1] Y
   - Z Bottom = Point[i+1] Z
   - Material = Point[i+1] material ID (for i=0, Point[0] and Point[1] share the same
     material per PGF's first-two-match rule, so this is consistent)
   - Boring name = boring name from PGF
   - Ground elevation:
     - Elevation mode without column 6: use the boring's first point Z (ground surface)
     - Depth mode or Elevation mode with column 6: use column 6 value

6. A PGF boring with N points produces N-1 LSDV interval rows.
7. Validate per `/evs-lsdv` rules.

**Data lost:** None. The PGF header message (Line 1) is not structural data and is recorded
in the generation metadata comments.

**Example:**

PGF input (2 borings, Elevation mode):
```
Pregeology file
Elevation 1|Silt 2|Sand 3|Clay m
6
100.0 200.0 50.0 1 BH-01
100.0 200.0 45.0 1 BH-01
100.0 200.0 30.0 2 BH-01
100.0 200.0 20.0 3 BH-01
110.0 210.0 48.0 2 BH-02
110.0 210.0 35.0 2 BH-02
```

LSDV output:
```
Elevation "1|Silt" "2|Sand" "3|Clay" m
# Source: [path to PGF file]
# Conversion: PGF to LSDV (lossless)
# PGF header message: "Pregeology file"
100.0 200.0 50.0 100.0 200.0 45.0 1 BH-01 50.0
100.0 200.0 45.0 100.0 200.0 30.0 2 BH-01 50.0
100.0 200.0 30.0 100.0 200.0 20.0 3 BH-01 50.0
110.0 210.0 48.0 110.0 210.0 35.0 2 BH-02 48.0
```

### LSDV to PGF (Lossy)

**Fidelity:** Lossy — missing intervals and non-surface-start borings cannot be represented
in PGF.

**When to use:** Exporting to systems that only accept PGF, or for legacy workflows.

**Prerequisite skills:** `/evs-lsdv`, `/evs-pgf`

**What is lost:**
- **Missing intervals** — PGF requires continuous coverage from ground surface to the
  bottom. Gaps between LSDV intervals cannot be represented.
- **Non-surface starts** — PGF borings must start at the ground surface. LSDV borings that
  begin below the surface cannot be directly represented.

**Procedure:**

1. Parse the LSDV file.
2. Group intervals by boring name.
3. **Check for gaps** within each boring (non-contiguous intervals). If found, warn the
   user and ask how to handle:
   - Fill gaps with a user-specified material (e.g., "Unknown")
   - Extend adjacent intervals to close the gap
   - Abort the conversion
4. **Check for non-surface starts** (first interval Z Top differs from ground elevation
   in Elevation mode, or Z Top > 0 in Depth mode). If found, warn the user and ask:
   - Extend the first interval's material up to ground surface
   - Add a user-specified material from ground surface to the first interval
   - Start the boring at the start of the first material and ignore the top
   - Abort
5. Build PGF Line 1: prompt the user for a header message, or default to
   "Converted from LSDV".
6. Build PGF Line 2: same coordinate mode, convert material definitions from LSDV
   `"ID|Name"` to PGF `ID|Name` (unquoted unless name contains spaces), same units.
7. For each boring, derive PGF transition points from LSDV intervals:
   - **Point 1:** X, Y, Z from the first interval's top. Material = first interval's
     material ID.
   - **Point 2:** X, Y, Z from the first interval's bottom. Material = first interval's
     material ID. (Satisfies PGF's first-two-match rule.)
   - **Point N (N >= 3):** X, Y, Z from interval (N-1)'s bottom. Material = interval
     (N-1)'s material ID.
   - A boring with M LSDV intervals produces M+1 PGF points.
8. Count total PGF data rows across all borings, write Line 3.
9. Write data rows.
10. Validate per `/evs-pgf` rules.

### LSDV to LPDV (Lossy)

**Fidelity:** Lossy — interval extent collapses to point(s).

**When to use:** Merging lithology with other point-based datasets, or when only point
representation is needed. This is the same operation EVS performs internally when it
"refines" LSDV data for use in kriging or modeling.

**Prerequisite skills:** `/evs-lsdv`, `/evs-lpdv`

**Procedure:**

1. Parse the LSDV file.
2. **Prompt the user** for the **refine distance**. This controls how densely points are
   placed within each interval. A smaller value produces more points; a larger value
   produces fewer. The user should choose a value appropriate for their modeling resolution.
   If the user has no preference, suggest a default of **1/60 of the total Z extent** of
   the data (max Z minus min Z across all intervals), which matches the EVS default.
3. Build LPDV header: same coordinate mode, same units. Preserve the source material IDs.
   Sequential IDs starting from 0 are preferred but not required.
4. Compute `min_distance` — the minimum 3D distance between the top and bottom of any
   interval across the entire dataset. Cap it: `min_distance = MIN(refine_distance * 10,
   min_distance)`.
5. For each LSDV interval, generate LPDV points using the EVS refine algorithm:
   a. Compute the 3D distance of the interval:
      `dist = sqrt((Xb-Xt)^2 + (Yb-Yt)^2 + (Zb-Zt)^2)`
   b. Subtract edge insets: `interior = dist - (min_distance / 10)`. This reserves 5% of
      `min_distance` on each side as a buffer so points are not placed exactly on interval
      boundaries.
   c. Compute the 3D unit vector from top to bottom:
      `vec = (Xb-Xt, Yb-Yt, Zb-Zt) / dist`
   d. **If `interior <= refine_distance`** (short interval): place 2 points:
      - Near top: position = `min_distance / 20` along the vector from top
      - Near bottom: position = `interior + min_distance / 20` along the vector from top
   e. **If `interior > refine_distance`** (long interval): subdivide:
      - `N = floor(interior / refine_distance)` (number of interior+bottom points)
      - `spacing = interior / N`
      - Place 1 point near top: position = `min_distance / 20`
      - Place N evenly spaced points: position = `spacing * (j+1) + min_distance / 20`
        for j = 0 to N-1
      - Total: N + 1 points for this interval
   f. Each generated point's coordinates are: `(Xt + vec.x * position, Yt + vec.y *
      position, Zt + vec.z * position)`. This correctly handles non-vertical borings.
   g. Each generated point inherits the interval's material ID, boring name, and ground
      elevation.
6. Validate per `/evs-lpdv` rules.

**Data lost:** Interval extent. Points are placed within intervals but the original
interval boundaries are not preserved as discrete entities.

### LPDV to LSDV (Requires Assumptions)

**Fidelity:** Requires assumptions — interval boundaries must be manufactured.

**When to use:** Upgrading point data to interval representation, or when the data was
originally interval-based but exported as points.

**Prerequisite skills:** `/evs-lpdv`, `/evs-lsdv`

**Procedure:**

1. Parse the LPDV file.
2. Group points by boring name.
3. **Prompt the user** for the interval derivation method:
   - **Midpoint boundaries** (default) — each interval extends from halfway to the
     previous point to halfway to the next point (within the same boring). The first
     point's interval extends up to ground surface elevation (if available) or prompt
     the user. The last point's interval extends down by the same thickness as the
     previous interval.
   - **Fixed thickness** — user specifies a thickness value. Each interval is centered
     on the point: Z Top = Z + thickness/2, Z Bottom = Z - thickness/2 (Elevation mode).
   - **User-provided** — user supplies explicit top/bottom for each point.
4. Build LSDV header: same coordinate mode, same material definitions (LSDV allows
   non-sequential IDs, so source IDs can be preserved), same units.
5. For each point, generate one LSDV row with the derived Z Top/Z Bottom. For vertical
   borings, X Top = X Bottom = source X, Y Top = Y Bottom = source Y.
6. Set ground elevation from the LPDV ground elevation column if present. If not, prompt
   the user.
7. Validate per `/evs-lsdv` rules (check for overlapping intervals).

### PGF to LPDV (Composite)

**Fidelity:** Lossy (same as LSDV to LPDV).

**Procedure:** PGF can be refined directly using the same refine distance algorithm as
LSDV to LPDV. The PGF transition point model defines intervals as consecutive pairs of
points within the same boring — apply the same refine algorithm to each pair. Alternatively,
convert PGF to LSDV first (lossless), then refine LSDV to LPDV. Both approaches produce
the same result. Document the conversion in the generation metadata comments.

## Analytical Conversions

### APDV to AIDV (Requires Assumptions)

**Fidelity:** Requires assumptions — interval bounds must be manufactured. A max gap value
must be chosen.

**When to use:** When the target system needs interval data, or when merging with existing
AIDV datasets.

**Prerequisite skills:** `/evs-apdv`, `/evs-aidv`

**Procedure:**

1. Parse the APDV file.
2. AIDV does not support `2D` mode. If the source APDV is `2D`, abort and inform the user
   that this conversion is not possible.
3. **Prompt the user** for the max gap value. If unknown, compute a default per the AIDV
   skill's formula (see `/evs-aidv` Line 2 specification).
4. **Prompt the user** for the interval derivation method:
   - **Midpoint boundaries** (default) — each interval extends from halfway to the
     previous sample to halfway to the next sample (within the same boring). The first
     sample extends up to ground surface (if top elevation is available) or by the same
     half-thickness as the nearest interval. The last sample extends down similarly.
   - **Fixed thickness** — user specifies interval thickness. Each interval is centered
     on the point.
   - **User-provided** — user supplies explicit top/bottom for each sample.
5. Build AIDV header:
   - Line 1: same analyte names (copy the `@@` line)
   - Line 2: same coordinate mode, max gap value, same units
   - Line 3: same row count, same analyte count, same analyte units
6. For each APDV data row, produce one AIDV data row:
   - X, Y unchanged
   - Z becomes Z Top and Z Bottom per the chosen method
   - Analyte values unchanged (including all non-detect and missing flags)
   - Boring name and top elevation carried over if present
7. Validate per `/evs-aidv` rules (check for overlapping intervals within borings).

### AIDV to APDV (Lossy)

**Fidelity:** Lossy — interval extent is lost. Long intervals may produce multiple points
with duplicated analyte values.

**When to use:** When the target system needs point data, or for simplified visualization.
This is the same operation EVS performs internally when it imports AIDV data for kriging.

**Prerequisite skills:** `/evs-aidv`, `/evs-apdv`

**Procedure:**

1. Parse the AIDV file. Read the **max gap** value from the AIDV header (Line 2).
2. **Prompt the user** for the point placement method:
   - **EVS refine** (default) — uses the max gap value from the AIDV header to subdivide
     long intervals into multiple points, matching the EVS internal algorithm
     (RefineGWCData). Use this unless the user has a specific reason to choose otherwise.
   - **Simple midpoint** — each interval produces one point at Z = (Z Top + Z Bottom) / 2,
     regardless of interval length
   - **Simple top** — each interval produces one point at Z = Z Top
   - **Simple bottom** — each interval produces one point at Z = Z Bottom
3. Build APDV header:
   - Line 1: same analyte names (copy the `@@` line)
   - Line 2: same coordinate mode (drop max gap), same units. APDV supports `2D` but
     this conversion always produces `Elevation` or `Depth` (AIDV has no 2D mode).
   - Line 3: row count (recomputed — may differ from AIDV row count when using EVS refine),
     same analyte count, same analyte units

**EVS refine algorithm (default):**

4. For each AIDV data row, compute the number of output points:
   - `interval_length = |Z Top - Z Bottom|`
   - `N = max(1, ceil(interval_length / max_gap))`
   - This means intervals shorter than or equal to max gap produce 1 point (at the
     midpoint). Longer intervals produce multiple points.
5. Divide the interval into N equal sub-intervals and place one point at the **midpoint**
   of each sub-interval:
   - `interval = Z Top - Z Bottom` (signed: positive in Elevation mode, negative in Depth
     mode)
   - For each sub-interval `k` (k = 0 to N-1):
     - `Z = Z Top - interval / (2 * N) - k * interval / N`
   - This produces evenly spaced points, each centered within its sub-interval.
6. For each generated point:
   - X, Y are unchanged from the source row
   - All analyte values are **duplicated** identically to each point (including non-detect
     and missing flags)
   - Boring name and top elevation carried over if present
7. Count total output rows across all intervals (sum of all N values). Write this as the
   row count on APDV Line 3.
8. Validate per `/evs-apdv` rules.

**Simple methods (midpoint, top, bottom):**

4. For each AIDV data row, produce exactly one APDV data row:
   - X, Y unchanged
   - Z = midpoint, Z Top, or Z Bottom per the chosen method
   - Analyte values unchanged (including all flags)
   - Boring name and top elevation carried over if present
5. Row count is the same as the AIDV row count.
6. Validate per `/evs-apdv` rules.

**Data lost:** Interval extent (Z Top, Z Bottom replaced by single Z per point). When using
EVS refine, long intervals produce multiple points with identical analyte values — the
duplicated values are an artifact of the subdivision, not independent measurements.

### LPDV to APDV — Indicator Conversion (Conditional)

**Fidelity:** Conditional — only possible when the source has exactly 2 materials.

**When to use:** When lithology data needs to be treated as an analytical indicator for
merging with other analyte data in EVS. This converts a binary material classification into
a numeric 0/1 analyte value, allowing lithology to be processed with EVS's analytical
kriging and visualization tools.

**Prerequisite skills:** `/evs-lpdv`, `/evs-apdv`

**Prerequisites check:**

1. Parse the LPDV file and count the number of distinct materials in the header.
2. If exactly **2 materials**: proceed.
3. If **more than 2 materials**: abort and inform the user that indicator conversion
   requires exactly 2 materials. The user would need to reclassify their lithology data
   into a binary grouping before this conversion is possible.

**Procedure:**

4. **Prompt the user** to confirm which material maps to 0.0 and which maps to 1.0. Present
   both materials by name and let the user assign the values.
5. Build APDV header:
   - Line 1: `@@"Material"` (single analyte named "Material")
   - Line 2: same coordinate mode, same units
   - Line 3: same row count, `1` analyte, `ID` as the unit
6. For each LPDV data row, produce one APDV data row:
   - X, Y, Z unchanged
   - Analyte value = `0.0` or `1.0` based on the material mapping from step 4
   - Boring name and ground elevation carried over if present
7. Validate per `/evs-apdv` rules.

**Composite paths:** Any lithology format can reach APDV via intermediate conversion to
LPDV first: LSDV → LPDV → APDV, PGF → LPDV → APDV, or GEO → LSDV → LPDV → APDV. The
same 2-material restriction applies regardless of the source format. Document the full
conversion chain in the generation metadata comments.

**Data lost:** Material identity is replaced by numeric indicator values. The original
material names are recorded in the generation metadata comments.

## Stratigraphy Conversions

### Single-Layer Stratigraphy Model

A single-layer stratigraphy model is a minimal GEO or GMF file that defines only two
surfaces: the **ground surface** and the **model floor**. It contains no lithology
interpretation — just the top and bottom boundaries of the model domain.

**Purpose:** In EVS, a stratigraphy model (GEO or GMF) constrains grids and models to
realistic boundaries. Even without lithology data or a multi-layer geologic interpretation,
a single-layer model provides:
- A ground surface that follows the actual boring data
- A model floor that prevents artificial extrapolation below the data
- The ability to create bounded grids for analytical kriging, indicator kriging, or 3D
  visualization

**Recommendation:** When creating any boring data file (APDV, AIDV, LPDV, LSDV, or PGF)
and no GEO or GMF file exists for the site, **prompt the user** and ask if a single-layer
stratigraphy model should also be generated. This is especially valuable when the data will
be used for kriging or 3D modeling, since without a stratigraphy model EVS has no surface
constraints for the grid.

**Prerequisite skills:** Source boring format skill, `/evs-geo` or `/evs-gmf`

**Procedure:**

1. **Ground surface** — Use the top of every boring as the ground surface elevation at that
   location (same heuristic as the hierarchy algorithm Step 3).
2. **Model floor** — Use the bottom of each boring as the floor elevation at that location.
   Apply the same outlier detection as the hierarchy algorithm Step 4: if a small number of
   borings go much deeper than the rest, prompt the user about whether to truncate them. EVS
   interpolates the floor surface between boring locations, so outlier depths can create
   artificial spikes in the model bottom.
3. **Generate GEO output:**
   - 2 surfaces: "Ground" and "Floor" (or user-specified names)
   - One row per boring location
   - Ground column: the top elevation of that boring
   - Floor column: the bottom elevation of that boring (or `short` if the user chose to
     truncate it)
   - Boring name from the source data
   - Follow all GEO format rules per `/evs-geo`
4. **Alternatively, generate GMF output** by producing GEO first, then converting via the
   GEO to GMF procedure.

**Example:**

Given 3 borings from an APDV file spanning various depths:
```
# Single-layer stratigraphy model
X	Y	Ground	Floor	Bore
Elevation	Ground	Floor	m
3	2	1	1
100.0	200.0	10.0	-5.0	BH-01
110.0	210.0	12.0	-8.0	BH-02
120.0	220.0	8.0	-3.0	BH-03
```

### Lithology to Stratigraphy — Building a Geologic Hierarchy

This section describes the core algorithm for converting lithology data (PGF or LSDV) into
a stratigraphic model (GEO or GMF). Unlike the mechanical conversions above, this is an
**interpretive conversion** that requires building a geologic hierarchy — a consistent
top-to-bottom ordering of named layers across all borings. The algorithm involves heuristics,
judgment calls, and multiple points where user input is required.

**Fidelity:** Interpretive — the output is a geological interpretation of the input data, not
a direct reformatting. Different interpretations of the same data can produce valid but
different results.

**Prerequisite skills:** `/evs-data`, `/evs-lithology`, source format skill (`/evs-pgf` or
`/evs-lsdv`), target format skill (`/evs-geo` or `/evs-gmf`)

#### Step 1 — Suitability Assessment

Not all sites are appropriate for stratigraphic modeling. GEO and GMF formats assume a
**layered geology** where surfaces can be defined between units. Before attempting conversion,
assess whether the site's geology is fundamentally layered.

**Do not convert** sites with:
- Karst terrain
- Heavily faulted zones
- Chaotic depositional environments
- Other non-layered geologies where materials cannot be ordered into a consistent vertical
  hierarchy

For these sites, recommend using lithology data directly with EVS tools that do not require
stratigraphic interpretation, such as **Geologic Indicator Kriging (GIK)**.

See the suitability discussion in `/evs-lithology` (Simplification and Stratigraphic
Modeling). If the assessment is uncertain, present the concern to the user and let them
decide.

#### Step 2 — Preprocessing

Three preprocessing steps are required before hierarchy construction:

1. **Material simplification** — Simplification is a **required precursor** to
   lithology-to-stratigraphy conversion. The more distinct materials present, the harder the
   hierarchy interpretation becomes and the more complex the resulting model.

   **Triggers for simplification:**
   - **Material count:** If the source data has more than 8-10 distinct materials, flag this
     and recommend simplification.
   - **Insufficient coverage:** If a material appears in too few borings to form valid GEO
     surfaces (fewer than 3 borings with data at any surface position — the minimum for
     surface interpolation), it must be merged into a related material or the resulting
     surfaces will be invalid. Analyze per-material boring counts early in preprocessing to
     detect this before hierarchy construction.

   See the simplification guidance in `/evs-lithology`.

   **Any material merge is a data-altering decision that requires explicit user approval.**
   Present the specific merge proposal to the user: which materials are being merged, why
   (too many materials, insufficient coverage, geologic similarity), and what the resulting
   material set will be. Do not perform any merge without user agreement — even when the
   merge is clearly the right decision (e.g., two variants of the same rock type where one
   has insufficient coverage), the user must approve it.

2. **Merge consecutive identical materials** — See the merging rules in `/evs-pgf` and
   `/evs-lsdv`. This is standard data hygiene for both formats and reduces noise before
   hierarchy construction.

3. **Normalize to interval form** — The hierarchy algorithm operates on depth intervals. If
   the source is PGF, convert to intervals using the PGF to LSDV procedure (lossless). If
   the source is already LSDV, this step is complete. After normalization, each boring is a
   list of (top, bottom, material) intervals from ground surface downward.

#### Step 3 — Ground Surface

The ground surface is the first (topmost) surface in the output GEO or GMF file. It
represents the top of the model.

- Use the **top of every boring** as the ground surface elevation at that location.
  - PGF: the first point's Z value for each boring.
  - LSDV: the first interval's Z Top for each boring (or the ground elevation column).

#### Step 4 — Model Floor

The model floor is the deepest surface in the output. It defines the bottom boundary of the
model.

- Use the **bottom of the deepest borings** as the model floor.
- **Detect extreme outliers:** If a small number of borings go significantly deeper than the
  rest (e.g., 2-3 borings out of 100 that are much deeper), prompt the user:
  - Present the depth distribution (deepest boring, next deepest, median depth, etc.)
  - Ask whether the outlier borings should be **truncated** to the depth of the main cluster
  - Explain that honoring extreme outliers can create artificial spikes in the bottom of the
    model
- **Honor at least 10-20% of the deepest borings** so the model reflects deep data points
  that are part of the main depth population, not just the single deepest boring.
- If the user agrees to truncation, drop the intervals below the truncation depth and use the
  next deepest elevation as the floor.

#### Step 5 — Top Material Selection

When multiple distinct materials appear as the first (shallowest) material across different
borings, determine which material should be at the **top of the hierarchy** — the first layer
below ground surface.

Apply these heuristics in order:

1. **Check material names for obvious surficial materials.** Names like "Fill", "Topsoil",
   "Overburden", "Made Ground", or similar indicate anthropogenic or surficial materials that
   should always be at the very top of the hierarchy.
2. **Check frequency.** The most common surface material across all borings is a strong
   candidate for the top layer.
3. **Check what is immediately below.** If most borings show Material A at the surface with
   Material B immediately below, and other borings show Material B at the surface with
   Material C below, then A is likely the top material (B appears at the surface only where A
   has been eroded or was never deposited).
4. **If ambiguous, prompt the user.** Present the materials found at the surface, their
   frequencies, and the patterns observed. Let the user decide.

When in doubt, prompt and question the user. There is no penalty for asking.

#### Step 6 — Hierarchy Construction

The hierarchy is a **consistent top-to-bottom ordering of named layers** that applies across
all borings. Each layer occupies a fixed position in the hierarchy, and surfaces between
layers form the GEO/GMF output.

**Core rule — material reappearance always means a split:**

If the same material appears non-consecutively in any boring (i.e., separated by one or more
different materials), it must be split into distinct hierarchy layers. It is **never** the
same stratigraphic layer. For example, if a boring shows Sand → Clay → Sand, the two Sand
occurrences must become "Upper Sand" and "Lower Sand" (or "Sand 1" and "Sand 2") — two
separate layers in the hierarchy with Clay between them.

**Algorithm:**

1. For each boring, list its material sequence from top to bottom (after preprocessing).
2. Walk through all borings and build a **global hierarchy** — an ordered list of layer
   names from top to bottom that is consistent with every boring's material sequence.
3. When a material reappears in any boring:
   - Create split variants: "Upper [Material]" / "Lower [Material]", or "[Material] 1" /
     "[Material] 2" if more than two occurrences.
   - Each variant gets its own position in the hierarchy.
4. Materials that appear in only some borings (lenses) get their own hierarchical position.
   In borings where they are absent, they are marked as pinch-outs.
5. Present the proposed hierarchy to the user for confirmation before proceeding.

**Midpoint split heuristic:**

When splitting a material around an intervening lens or pinch-out, place the split surface
through the **depth midpoints** of the intervening material. For example, if a boring has:

```
Sand: 50.0 to 40.0
Clay: 40.0 to 35.0
Sand: 35.0 to 20.0
```

The Clay is a lens within Sand. Split Sand into Upper Sand and Lower Sand. The split surface
in borings where Clay is absent should pass through the midpoint of where Clay would be —
which keeps the Upper Sand and Lower Sand thicknesses relatively balanced. In borings that
do have Clay, Upper Sand sits above Clay and Lower Sand sits below.

This heuristic extends generally: whenever split surfaces need to be placed through borings
where the intervening material is absent, use the depth midpoints of the intervening
material from nearby borings where it is present. This produces the most even layer
thicknesses and the most natural-looking model.

**Layer thickness uniformity:**

Stratigraphic layers should vary **slowly** across the site, reflecting natural depositional
processes. Use this as a reasonableness check on the hierarchy: if a proposed hierarchy
produces layers that vary wildly in thickness from boring to boring, the interpretation may
need adjustment. Flag significant thickness anomalies to the user.

#### Step 7 — Generate Output

Once the hierarchy is finalized, generate the target format:

**For GEO output:**

- The hierarchy defines the surfaces. N layers require N+1 surfaces (ground + N layer
  boundaries).
- Surface names come from the hierarchy layer names (the lower surface names the layer,
  per GEO convention — see `/evs-geo`). **Use base material names without split suffixes.**
  When a material appears at multiple hierarchy positions (e.g., Dolomite above and below
  Sandstone), repeat the base name (e.g., `Dolomite Sandstone Dolomite Sandstone`). Do not
  add `-1`, `-2` or "Upper"/"Lower" suffixes to the GEO surface names — duplicate names are
  allowed and preferred (see `/evs-geo` Line 2 specification). The split variant names from
  Step 6 are for internal hierarchy tracking only.
- For each boring, write the elevation/depth of each surface.
- Where a layer is absent (pinch-out), use the `pinch` flag.
- Where a boring terminates above the model floor, use `short` for the remaining surfaces.
- Assign color numbers per the GEO format specification.
- Follow all GEO format rules per `/evs-geo`.
- **Generation metadata comments** — insert between the GEO header (Line 3) and the first
  data row. Document at minimum:
  - Source file path
  - Conversion type (e.g., "PGF to GEO, interpretive — hierarchy construction")
  - Date and time of generation
  - Material merges performed (which materials, why)
  - Floor truncation decisions (which borings truncated, at what depth)
  - Final hierarchy used (ordered list of layer names)
  - Any other key decisions or assumptions

**For GMF output:**

- Generate GEO first, then convert to GMF using the existing GEO to GMF procedure. This
  ensures consistent handling of pinch-outs and short borings.

**Merged source file offer:**

If any material merges were performed during preprocessing (Step 2), offer to produce a
version of the source file (PGF or LSDV) with the same merges applied. When the source
lithology file and the output stratigraphy file use different material sets, loading both in
EVS produces mismatched colors and legends that confuse the user. A merged source file
ensures consistency. If the user accepts, apply the same material remapping to the source
data (remap material IDs, merge consecutive identical intervals/points, update material
definitions and row count). Write the merged file alongside the original with a `-merged`
suffix (e.g., `site-data-merged.pgf`). Include **generation metadata comments** in the
merged file (between the header and data) documenting:
- Source file path (the original, unmerged file)
- Operation performed (e.g., "Material merge: limestone → dolomite")
- Date and time of generation
- Materials removed and their merge targets
- Number of points/intervals removed by consecutive-material merging
- Borings affected by the merge

If the user declines, note in the GEO/GMF generation metadata comments that the source
file uses the original (unmerged) material set and the output uses the simplified set.

#### User Confirmation Points

This conversion requires user input at multiple steps. Summary of all confirmation points:

1. **Suitability** — Is the site appropriate for stratigraphic modeling?
2. **Material merges** — Does the user approve any proposed material merges? Present the
   specific merge (which materials, why), not just a generic "simplify?" prompt.
3. **Floor truncation** — Should outlier deep borings be truncated?
4. **Top material** — Which material is at the top of the hierarchy (if ambiguous)?
5. **Proposed hierarchy** — Does the user approve the layer ordering?
6. **Material splits** — Does the user agree with how reappearing materials are split?
7. **Final output** — Does the user approve the generated file before writing?
8. **Merged source file** — If material merges were performed, should a version of the
   source file with matching merges be produced for EVS consistency?

Do not proceed past any of these points without explicit user agreement.

### PGF to GEO (Interpretive)

**Fidelity:** Interpretive — requires building a geologic hierarchy from lithology data.

**When to use:** Creating a stratigraphic model from PGF lithology data for use with EVS
stratigraphic tools.

**Prerequisite skills:** `/evs-pgf`, `/evs-geo`, `/evs-lithology`

**Procedure:**

1. Run the full **Lithology to Stratigraphy** algorithm above, using PGF as the source.
2. During preprocessing (Step 2), the PGF data is normalized to interval form via the
   PGF to LSDV conversion.
3. The algorithm produces a GEO file as output.

### LSDV to GEO (Interpretive)

**Fidelity:** Interpretive — requires building a geologic hierarchy from lithology data.

**When to use:** Creating a stratigraphic model from LSDV lithology data for use with EVS
stratigraphic tools.

**Prerequisite skills:** `/evs-lsdv`, `/evs-geo`, `/evs-lithology`

**Procedure:**

1. Run the full **Lithology to Stratigraphy** algorithm above, using LSDV as the source.
2. LSDV data is already in interval form, so the normalization substep is not needed.
3. The algorithm produces a GEO file as output.

### PGF to GMF (Composite)

**Fidelity:** Interpretive (same as PGF to GEO).

**Procedure:** Convert PGF to GEO using the PGF to GEO procedure above, then convert GEO to
GMF using the existing GEO to GMF procedure. Document the full conversion chain in the
generation metadata comments.

### LSDV to GMF (Composite)

**Fidelity:** Interpretive (same as LSDV to GEO).

**Procedure:** Convert LSDV to GEO using the LSDV to GEO procedure above, then convert GEO
to GMF using the existing GEO to GMF procedure. Document the full conversion chain in the
generation metadata comments.

### GEO to GMF (Lossless)

**Fidelity:** Lossless — all data is preserved, though representation changes.

**When to use:** When surfaces need independent point distributions, or when importing into
systems that prefer GMF.

**Prerequisite skills:** `/evs-geo`, `/evs-gmf`

**Procedure:**

1. Parse the GEO file: Line 1 (title), Line 2 (mode, surface names, units), Line 3
   (row count, surface count, colors), data rows.
2. GMF is elevation-only. If GEO uses `Depth` mode, convert all depth values to elevations:
   - Elevation = Ground Surface Value - Depth Value
   - Use each boring's top surface value (first surface column) as the ground surface
     reference
   - Warn the user about this conversion and confirm
3. Write the GMF `units` line using the GEO units.
4. For each surface index (1 to surface count):
   - Write `surface [color_number] [surface_name]` using the color number from GEO Line 3
     and the surface name from GEO Line 2
   - For each boring (GEO data row):
     - Read the surface value for this column
     - If **numeric**: write `X Y Z` (the elevation value)
     - If **pinch**: write `X Y Z` where Z = the **previous surface's numeric value** for
       this boring. This duplicates the elevation, which is how GMF represents pinch-outs.
     - If **short**, **missing**, **unknown**, **na**, or **terminated**: skip this boring
       for this surface. Per GEO rules, all deeper surfaces for this boring will also be
       Class A flags, so the boring will be skipped for all remaining surfaces too.
5. Write `end`. If GEO Line 1 starts with `$W` or `$G`, append `$W` to the end line.
6. Record the boring-name-to-X,Y mapping in the generation metadata comments (GMF does not
   store boring names).
7. Validate per `/evs-gmf` rules.

**Data preserved differently:** Boring names are not stored in GMF. The X,Y coordinates
serve as the implicit boring identifier. Document the name-to-coordinate mapping in the
generation metadata comments.

**Example:**

GEO input (3 borings, 3 surfaces):
```
X	Y	Top	Sand	Clay	Bore
Elevation	Top	Sand	Clay	m
3	3	1	1	2
100.0	200.0	10.0	5.0	-2.0	BH-01
110.0	210.0	12.0	pinch	-5.0	BH-02
120.0	220.0	8.0	short	short	BH-03
```

GMF output:
```
units m
# Source: [path to GEO file]
# Conversion: GEO to GMF (lossless)
# Boring mapping: BH-01=(100.0, 200.0), BH-02=(110.0, 210.0), BH-03=(120.0, 220.0)
surface 1 Top
100.0	200.0	10.0
110.0	210.0	12.0
120.0	220.0	8.0
surface 1 Sand
100.0	200.0	5.0
110.0	210.0	12.0
surface 2 Clay
100.0	200.0	-2.0
110.0	210.0	-5.0
end
```

Notes:
- BH-02's Sand value was `pinch`, so Sand gets the same Z as Top (12.0) for that location.
- BH-03 was `short` at Sand, so BH-03 is omitted from Sand and Clay surfaces.

### GMF to GEO (Conditional)

**Fidelity:** Conditional — works well when most X,Y locations appear in most surfaces.
Partial coverage is handled using GEO's `short` and `missing` flags.

**When to use:** Converting to the boring-correlated tabular format.

**Prerequisite skills:** `/evs-gmf`, `/evs-geo`

**Coverage analysis:**

1. Parse the GMF file: units, surface blocks (name, material number, points), end line.
2. Collect all unique X,Y locations across all surfaces. Each unique X,Y becomes a
   potential boring.
3. For each X,Y location, determine which surfaces have a point there. Compute a
   **coverage ratio** = (number of surfaces with data at this X,Y) / (total surfaces).
4. Classify each X,Y location:
   - **Full coverage** — present in all surfaces.
   - **Top-down coverage** — present in the first N surfaces contiguously from the top,
     absent in deeper surfaces. This maps naturally to `short` in GEO (boring didn't
     reach that deep).
   - **Partial coverage with gaps** — present in some surfaces but missing in others
     mid-sequence. These gaps become `missing` flags in GEO.
5. Assess overall data quality:
   - If **80% or more** of X,Y locations have a coverage ratio above 50% (i.e., most
     locations appear in most surfaces), proceed with the conversion. Locations with
     partial coverage use `short` and `missing` flags as appropriate.
   - If **more than 20%** of X,Y locations would produce borings with mostly missing
     values (coverage ratio below 50%), warn the user that the data may not be
     well-suited for GEO format and present options:
     - **Proceed anyway** — include all locations, using `short`/`missing` flags
     - **Abort** — keep GMF format
   - Do **not** interpolate or fabricate data for missing surfaces.
   - Do **not** offer to subset the data. If the user wants to exclude specific locations,
     they must request it explicitly.

**Procedure:**

6. Use all X,Y locations for the conversion.
7. GEO will use `Elevation` mode (GMF is elevation-only).
8. Build GEO Line 1: prompt the user for a title, or default to "Converted from GMF".
9. Build GEO Line 2: `Elevation`, surface names from GMF surface declarations, units.
10. Assign color numbers from GMF material numbers.
11. For each X,Y location (becomes one GEO row):
    - Write X, Y.
    - For each surface in order:
      - If the surface has a point at this X,Y: write the Z value.
      - Detect pinch-outs: if two consecutive surfaces have identical Z at this X,Y,
        write the numeric value for the first and `pinch` for the second.
      - If the surface has no point at this X,Y:
        - If **no deeper surface** has a point at this X,Y either: use `short`. Per GEO
          rules, all remaining deeper surfaces for this boring will also be `short`.
        - If a **deeper surface does** have a point at this X,Y: use `missing`. This
          indicates the data is absent at this surface but the boring continues.
    - Write boring name (see step 12 for naming).
12. **Assign boring names.** GMF does not store boring names, so prompt the user for a
    naming method:
    - **External mapping** — the user provides a data source (CSV, Excel, or another EVS
      file) that maps X,Y coordinates to boring names. Match GMF X,Y locations to the
      mapping source using Euclidean distance in full floating-point precision (do not round
      coordinates before matching). Apply a proximity threshold to avoid false matches:
      compute the **minimum inter-boring distance** (the smallest Euclidean distance between
      any two distinct X,Y locations in the mapping source) and reject any match whose
      distance exceeds **10% of the minimum inter-boring distance**. This ensures matches
      are only accepted when the coordinates are very close — much closer than the spacing
      between any two borings in the source data.
      Exact matches (distance = 0) are always accepted. Near-exact matches (within
      threshold) are accepted but documented in the generation metadata comments with the
      GMF X,Y, matched source X,Y, distance, and assigned name. Locations that exceed the
      threshold are reported to the user as unmatched and fall back to coordinate-based
      naming. This is especially useful when the GMF was originally converted from a GEO
      file whose metadata comments contain the original boring-name-to-coordinate mapping.
    - **Coordinate-based** — generate boring names from the X,Y coordinates (e.g.,
      `X100.0_Y200.0`). This produces unique, traceable names without requiring an external
      source.
    - **Sequential** — generate sequential names (e.g., BH-001, BH-002, ...). Simple but
      names have no geographic meaning.
    Document the chosen naming method and any mappings in the generation metadata comments.
13. Write GEO Line 3: row count, surface count, color numbers.
14. Validate per `/evs-geo` rules.
15. Check `$W` flag: if the GMF `end` line contained `$W`, prepend `$W` to GEO Line 1.

**Data representation:** Locations with partial coverage are represented using `short` and
`missing` flags. No data is excluded unless the user explicitly requests it.

### GEO to LSDV (Lossless)

**Fidelity:** Lossless — all data is preserved.

**When to use:** When lithology interval data is needed from stratigraphic surfaces, e.g.,
for kriging or combining with other lithology datasets.

**Prerequisite skills:** `/evs-geo`, `/evs-lsdv`

**Procedure:**

1. Parse the GEO file.
2. **Determine material mapping.** GEO defines N surfaces (boundaries); these create N-1
   layers between consecutive surfaces. In EVS, the **lower surface always names the
   layer** — the layer between surface[i] and surface[i+1] is the material named by
   surface[i+1]. Use the GEO color numbers as material IDs in the LSDV header, or remap
   per material ID ordering guidance if the user prefers.
3. Build LSDV header: same coordinate mode, material definitions from the mapping, same
   units.
4. For each boring (GEO data row), walk consecutive surface pairs to produce LSDV intervals:
   - For surfaces S[i] and S[i+1] where both are numeric:
     - Xt = Xb = boring X, Yt = Yb = boring Y (GEO borings are vertical)
     - Zt = S[i] value, Zb = S[i+1] value
     - Material = layer material per step 2 mapping
   - If S[i+1] is `pinch`: Zb = Zt (zero-thickness interval). Include in output to preserve
     the material boundary.
   - If S[i+1] is `short`, `missing`, or other Class A flag: boring terminates at S[i]. No
     more intervals for this boring (per GEO rules, all deeper surfaces will also be Class A
     flags).
   - Boring name from the GEO boring name column.
   - Ground elevation = first surface value for this boring (Elevation mode) or from the
     ground elevation column if present.
5. A boring with K valid surfaces (before termination) produces K-1 LSDV intervals.
6. Validate per `/evs-lsdv` rules.

**Data lost:** None. The GEO title (Line 1) is not structural data and is recorded in the
generation metadata comments.

### GEO to PGF (Lossless)

**Fidelity:** Lossless — all data is preserved. PGF requires continuous coverage from ground
surface to the deepest point, which GEO naturally provides (borings with Class A flags
simply end at the last valid surface).

**When to use:** When legacy PGF format is needed from stratigraphic data.

**Prerequisite skills:** `/evs-geo`, `/evs-pgf`

**Procedure:**

1. Parse the GEO file.
2. **Determine material mapping** — same as GEO to LSDV step 2.
3. Build PGF Line 1: prompt the user for a header message, or default to
   "Converted from GEO".
4. Build PGF Line 2: same coordinate mode, material definitions (ID|Name format), same
   units.
5. For each boring, generate PGF transition points from GEO surfaces:
   - **Point 1:** X, Y, Z = first surface (ground). Material = first layer's material.
   - **Point 2:** X, Y, Z = second surface. Material = first layer's material. (Satisfies
     PGF's first-two-match rule.)
   - **Point N (N >= 3):** X, Y, Z = surface[N-1]. Material = layer[N-2]'s material (the
     material of the interval from surface[N-2] to surface[N-1]).
   - If a surface is `pinch`: use the previous surface's Z value. The material changes but
     the position doesn't.
   - If a surface is `short`, `missing`, or other Class A flag: boring ends at the previous
     point. No more points for this boring.
   - A boring with K valid surfaces produces K PGF points.
6. Count total PGF data rows across all borings, write Line 3.
7. Validate per `/evs-pgf` rules.

**Data lost:** None. The GEO title (Line 1) is not structural data and is recorded in the
generation metadata comments.

### GEO to LPDV (Composite)

**Fidelity:** Lossy (GEO to LSDV is lossless, but LSDV to LPDV is lossy).

**When to use:** When point-based lithology data is needed from stratigraphic surfaces.

**Procedure:** Convert GEO to LSDV first (using the GEO to LSDV procedure above), then
refine LSDV to LPDV (using the LSDV to LPDV refine algorithm). Alternatively, apply the
refine algorithm directly to the GEO-derived intervals without writing an intermediate LSDV
file — the interval geometry is the same either way. Document the full conversion chain in
the generation metadata comments.

## Post-Conversion Validation

After any conversion, verify:

1. Run the **target format's full validation rules** (from the target format skill).
2. Compare **boring count** between source and target (should match unless borings were
   excluded due to flags).
3. Compare **data point count** accounting for expected changes (e.g., PGF with N points
   per boring produces N-1 LSDV intervals per boring).
4. Verify **coordinate ranges** are consistent (min/max X, Y, Z should match between
   source and target, accounting for any mode conversion).
5. Verify **material/analyte/surface names** are complete and match the source.
6. Confirm **generation metadata comments** document the conversion accurately.
7. Show the **complete output to the user** for confirmation before writing.
