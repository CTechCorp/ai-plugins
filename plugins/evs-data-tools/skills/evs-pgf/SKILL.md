---
name: evs-pgf
description: >-
  Detailed format specification for EVS PreGeology File (.pgf) files. Use when creating,
  reading, validating, transforming, or debugging .pgf files for Earth Volumetric Studio.
  These files contain point-based lithology transition data associated with borehole locations,
  where each boring is described as a sequence of transition points from the ground surface
  downward through successive material layers.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS PreGeology File (.pgf)

Prerequisites: Load `/evs-data` first for shared rules (delimiters, encoding, coordinate
system requirements, etc.) that apply to all EVS data files. Load `/evs-lithology` for
shared lithology concepts (material definitions, ID ordering, simplification guidance).

Official documentation:
https://www.ctech.com/studio_help/file-format-details/pre_geology_file/index.html

## Purpose

PGF files contain lithology (material classification) data for boreholes, represented as a
series of **transition points**. Each boring begins at the ground surface and progresses
downward, with each point marking where the material changes. The material at each point
describes the layer that extends from the previous point down to the current point.

PGF files are used for:
- Creating stratigraphic hierarchy
- Posting samples
- Gridding and horizons (extracting top/bottom surfaces for single layers)
- Lithologic modeling for Geologic Indicator Kriging (GIK)
- Adaptive indicator kriging

PGF is the older lithology format. **LSDV is the modern replacement** with additional
capabilities including support for missing intervals, borings that do not start at ground
surface, and explicit interval definitions with separate top/bottom coordinates per row.

**Use PGF when** the data represents lithology as a sequence of transition points from
ground surface downward, or when working with legacy PGF files.
**Use LSDV instead** when you need missing intervals, borings that don't start at ground
surface, or explicit top/bottom coordinate pairs per interval.

## File Format Overview

A PGF file is a UTF-8 text file with the `.pgf` extension. It consists of:

1. A **header message** line (informational only)
2. A **declaration line** defining coordinate mode, material mappings, and optional units
3. A **row count** line (integer count of data lines that follow)
4. **Data rows** — one row per transition point

Standard EVS delimiter rules apply (spaces, commas, tabs are interchangeable; consecutive
delimiters collapse).

## Header Specification

The header spans three lines. All three lines are required.

### Line 1 — Header Message

A free-text message line. This line is informational only and has no effect on parsing,
but it **cannot be blank and cannot be a comment** (must not start with `#`). Any non-empty
text is acceptable.

**Examples:**

```
Pregeology file
Site ABC Lithology Data - 2024
PGF export from database
```

### Line 2 — Declaration Line

A single line containing three parts in order:

#### Part 1 — Coordinate Mode

- `Elevation` — Z values are true elevations (e.g., feet above sea level)
- `Depth` — Z values are positive numbers representing depth below ground surface

The keyword is case-insensitive.

#### Part 2 — Material Definitions

One or more material definitions, each in the format `ID|Name`:

- The ID is an integer identifying the material
- The name is the material/lithology label
- The pipe character `|` separates the ID from the name with no spaces around it
- Sequential IDs starting from 0 are preferred but not required — any integer values are
  valid
- If a material name contains spaces, the entire `ID|Name` pair must be quoted
  (e.g., `"4|Silty Sand"`)
- All material IDs used in the data section must have a corresponding definition here
- Additional ID/Name pairs may be defined that do not appear in the data — these will
  appear in the legend
- The order of definitions determines the legend display order in EVS

#### Material ID Ordering

See [Material ID Ordering in `/evs-lithology`](/evs-lithology) for the 5 available
ordering strategies. Prompt the user when generating a new file.

#### Part 3 — Coordinate Units (optional)

- `m` or `ft`

**Examples:**

```
Elevation 1|Silt 2|Fill 3|Clay 4|Sand 5|Bedrock m
Depth 0|Topsoil 1|Clay 2|Sand 3|Gravel ft
Elevation 1|Sand "2|Silty Sand" 3|Clay "4|Sandy Gravel" m
```

### Line 3 — Row Count

A single integer specifying the exact number of data lines that follow.

**Example:**

```
17
```

## Column Specification

Each data row contains **5 or 6 columns** in order:

| # | Column | Required | Description |
|---|--------|----------|-------------|
| 1 | X | Yes | X coordinate of the transition point |
| 2 | Y | Yes | Y coordinate of the transition point |
| 3 | Z | Yes | Elevation or depth of the transition point |
| 4 | Lithology ID | Yes | Integer material code matching a header definition |
| 5 | Boring name | Yes | Well/boring identifier (must be quoted if it contains spaces) |
| 6 | Ground elevation | Conditional | Ground surface elevation at the boring location |

**Notes:**
- Column 6 (ground elevation) is **optional in Elevation mode** and **required in Depth
  mode**. In Depth mode, if ground elevation is omitted it defaults to 0.0 with a warning.
- For vertical borings, X and Y are the same for every point in the boring
- For non-vertical borings (slant holes), X and Y can vary between points, allowing the
  boring trajectory to be represented
- Each boring name must correspond to a unique X, Y starting location — overlapping borings
  at the same X, Y with overlapping Z ranges are not permitted
- In `Elevation` mode, Z values decrease from the first point (ground surface) downward
- In `Depth` mode, Z values increase from the first point (ground surface = 0 or near 0)
  downward

## Transition Point Model

PGF uses a **transition point model** rather than explicit intervals. Each boring is
described as a series of points from the ground surface downward:

1. **Point 1** — The ground surface location and the material present at the surface
2. **Point 2** — The bottom of the first material layer (must have the **same material ID**
   as Point 1)
3. **Point 3** — The bottom of the second material layer (may have a different material ID)
4. **Point N** — The bottom of the (N-1)th material layer

**Critical rule:** The first two points of every boring must have the **same Lithology ID**.
Point 1 establishes "what material is at the surface" and Point 2 establishes "how deep
that first layer extends." Together they define the first complete layer.

**How layers are derived:**

| Layer | Top (from) | Bottom (to) | Material |
|-------|------------|-------------|----------|
| 1 | Point 1 Z | Point 2 Z | Material of Points 1 and 2 (same) |
| 2 | Point 2 Z | Point 3 Z | Material of Point 3 |
| 3 | Point 3 Z | Point 4 Z | Material of Point 4 |
| ... | ... | ... | ... |

**Example for a single boring:**

```
100.0 200.0  50.0  1  BH-01    # Ground surface: Silt starts here
100.0 200.0  45.0  1  BH-01    # Bottom of Silt layer (same material as surface)
100.0 200.0  30.0  2  BH-01    # Bottom of Fill layer
100.0 200.0  20.0  3  BH-01    # Bottom of Clay layer
```

This defines:
- Silt from elevation 50.0 to 45.0
- Fill from elevation 45.0 to 30.0
- Clay from elevation 30.0 to 20.0

## Example File

A file with 3 borings and 5 material types, using Elevation mode in meters:

```
Pregeology file
Elevation 1|Silt 2|Fill 3|Clay 4|Sand 5|Bedrock m
17
11086.5 12830.7 4.5 1 B-49
11086.5 12830.7 -3.8 1 B-49
11086.5 12830.7 -21 2 B-49
11086.5 12830.7 -26 3 B-49
11086.5 12830.7 -42 4 B-49
11086.5 12830.7 -55 5 B-49
11199 12600 4 1 B-51
11199 12600 -5 1 B-51
11199 12600 -20 2 B-51
11199 12600 -25 3 B-51
11199 12600 -39 4 B-51
11199 12600 -53 5 B-51
11259.7 12819.3 2 1 B-46
11259.7 12819.3 -7.5 1 B-46
11259.7 12819.3 -27 3 B-46
11259.7 12819.3 -40 4 B-46
11259.7 12819.3 -53 5 B-46
```

**Breakdown:**

- **Line 1:** Header message — informational text, no parsing effect.
- **Line 2:** Elevation mode, 5 materials (Silt=1, Fill=2, Clay=3, Sand=4, Bedrock=5),
  meters.
- **Line 3:** 17 data rows follow.
- **B-49 (6 points):** Starts at ground surface elevation 4.5. First two points are both
  material 1 (Silt), defining the first layer from 4.5 to -3.8. Then Fill to -21, Clay to
  -26, Sand to -42, Bedrock to -55.
- **B-51 (6 points):** Starts at ground surface elevation 4. Same pattern — Silt first
  layer, then Fill, Clay, Sand, Bedrock.
- **B-46 (5 points):** Starts at ground surface elevation 2. First layer is Silt to -7.5.
  Note that B-46 skips material 2 (Fill) — not every boring encounters every material.
  Then Clay to -27, Sand to -40, Bedrock to -53.
- **No ground elevation column** is present because Elevation mode is used (column 6 is
  optional in Elevation mode).
- All borings are vertical (X, Y are constant within each boring).

## Validation Rules

In addition to the [common validation rules in `/evs-data`](/evs-data):

- Every Lithology ID in the data must match a definition on Line 2
- The row count on Line 3 must match the actual number of data rows
- The first two points of each boring must have the **same Lithology ID**
- Line 1 (header message) must not be blank and must not be a comment
- Data within each boring must be sorted from beginning to end (normally top to bottom) —
  EVS cannot sort the data because borings may be non-vertical
- Direction variance exceeding 90 degrees between consecutive points in a boring triggers
  an error — ensure points proceed in a consistent direction
- Each boring name must correspond to a unique starting X, Y location — do not define
  overlapping borings at the same X, Y
- In `Elevation` mode, Z values within a boring must decrease from surface downward
- In `Depth` mode, Z values within a boring must increase from surface downward
- In `Depth` mode, ground surface elevation (column 6) is required — if omitted, EVS
  defaults to 0.0 with a warning
- Every data row must have at least 5 columns (6 in Depth mode)

## Merging Consecutive Identical Materials

When creating, cleaning, or preprocessing PGF data, **merge consecutive transition points
that share the same material**. If Point[i], Point[i+1], and Point[i+2] all have the same
Lithology ID, remove the intermediate Point[i+1] — the boundary it represents is not
geologically meaningful. The result is a cleaner boring with fewer redundant points and the
same layer geometry. This applies to any workflow that produces or modifies PGF data, not
just conversions. Always merge before further processing (e.g., hierarchy construction,
export, or visualization).

## Edge Cases and Gotchas

- PGF has a **mandatory row count** on Line 3 — unlike LSDV which has no row count and
  treats all remaining non-comment lines as data
- PGF has a **mandatory header message** on Line 1 — this is unique among EVS lithology
  formats. It cannot be blank or a comment, but its content is ignored by the parser
- Material IDs do **not** need to be sequential — any integer values are valid, though
  sequential starting from 0 is preferred. This allows a unified set of material IDs
  across multiple PGF files for a large site
- Not every boring needs to encounter every defined material — some materials may only
  appear in certain borings (see B-46 in the example, which skips Fill)
- Additional material definitions can appear on Line 2 that are never used in the data —
  these appear in the EVS legend but do not correspond to any data
- The order of material definitions on Line 2 determines the **legend display order** in
  EVS, not the numeric order of the IDs
- Ground surface elevation (column 6) is **optional in Elevation mode** because the first
  point's Z value already represents the ground surface elevation. In **Depth mode it is
  required** because Z=0 is relative and the actual elevation must be specified
- PGF supports **non-vertical borings** — X and Y can vary between points within the same
  boring, representing a deviated wellbore trajectory
- Unlike LSDV, PGF does **not support missing intervals** — every boring must have
  continuous coverage from the ground surface to the bottom of the deepest recorded layer,
  with no gaps
- Unlike LSDV, PGF borings **always start at the ground surface** — there is no way to
  represent a boring that begins at depth
- See [Shared Gotchas in `/evs-lithology`](/evs-lithology) for material definition format
  notes, non-detect/missing flag rules, and other cross-format lithology gotchas
- Comment lines (starting with `#`) can appear anywhere in the file and are skipped — but
  they do **not** count toward the row count on Line 3
- Material names containing spaces require the entire `ID|Name` pair to be quoted
  (e.g., `"4|Silty Sand"`)
- PGF files can be converted to LSDV format for additional flexibility — each pair of
  consecutive points in a PGF boring maps to one LSDV interval row
