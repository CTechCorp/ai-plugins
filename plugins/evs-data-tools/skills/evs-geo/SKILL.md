---
name: evs-geo
description: >-
  Detailed format specification for EVS Borehole Geologic Stratigraphy (.geo) files. Use when
  creating, reading, validating, transforming, or debugging .geo files for Earth Volumetric Studio.
  These files contain single-file geologic stratigraphy data describing geologic layers and
  formations encountered in boreholes.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Borehole Geologic Stratigraphy File (.geo)

Prerequisite: Load `/evs-data` first for shared rules (delimiters, encoding, coordinate
system requirements, etc.) that apply to all EVS data files.

Official documentation:
https://www.ctech.com/studio_help/file-format-details/geology_file_examples_figures/index.html

## Purpose

GEO files describe geologic stratigraphy as a tabular single-file format where each boring
provides depth/elevation values for every geologic surface. All borings share the same set
of surfaces, making GEO ideal for sites with relatively uniform layering.

EVS uses GEO data to construct 3D stratigraphic models by interpolating surface elevations
between boring locations. Each surface represents the bottom of a geologic layer (except the
first surface, which is the ground/top surface). The surface name defines the material of
the layer **above** that surface — the layer between surface[i] and surface[i+1] is the
material named by surface[i+1]. For example, if surfaces are "Ground", "Sand", "Clay", then
the layer from Ground to Sand is sand, and the layer from Sand to Clay is clay.

**Use GEO when** all borings share the same set of geologic surfaces and the stratigraphy
is relatively uniform across the site.
**Use GMF instead** when surfaces have independent point distributions (different borings
contribute to different surfaces), or when the data comes from non-borehole sources like
geophysical surveys.

## File Format Overview

A GEO file is a UTF-8 text file with the `.geo` extension. It consists of:

1. A **3-line header** defining column labels, surface names/mode/units, and row/surface counts
2. **Data rows** — one row per boring location

Standard EVS delimiter rules apply (spaces, commas, tabs are interchangeable; consecutive
delimiters collapse).

## Header Specification

### Line 1 — Column Labels / Title

A descriptive header line. This line is required and **cannot be blank**. It typically
contains column labels or a project title.

**Special prefix flags** — if the first two characters are `$W` or `$G`:
- This signals that the surfaces are **unrelated** (e.g., historical water table elevations
  at different times, or independent geophysical horizons)
- EVS disables corruption checking and kriging in thickness space for these files
- Use `$W` or `$G` when surfaces do not represent a stacked stratigraphic sequence
- **When surface names are dates/times** (e.g., water table elevations measured at different
  times), the `$W` flag should be used. This is the most common use case for `$W`.

**Examples:**
```
X Coord "Y Coord" Ground "Water Table" Bottom BoringID
X	Y	Top	Bot_0	Bot_1	Bot_2	Bot_3	Bore
$W Water Table Elevations Over Time
```

### Line 2 — Coordinate Mode, Surface Names, and Units

This line contains, in order:

1. **Coordinate mode** — `Elevation` or `Depth` (case-insensitive)
2. **Surface names** (optional but recommended) — one name per surface. The first name
   corresponds to the top surface (ground surface), and each subsequent name corresponds to
   the bottom of a geologic layer. Names with spaces must be quoted. Use only letters,
   numerals, hyphens, and underscores in unquoted names. Surface names can also be
   **dates/times** (e.g., for water table data over time) — date/time names with spaces
   must be quoted. When surface names are dates/times, use the `$W` flag on Line 1.
3. **Units** — the last token on the line: `m`, `ft`, `meters`, `feet`, etc.

The number of surface names must match the surface count specified on Line 3.

**Duplicate surface names are allowed.** When the same geologic material appears at
multiple positions in the hierarchy (e.g., alternating Dolomite / Sandstone layers),
use the same base name for each occurrence (e.g., `Dolomite Sandstone Dolomite Sandstone`).
Do not add uniqueness suffixes like `-1`, `-2`. In EVS, the first surface name associated
with a given color number is used as the display name for that material in legends and
visualizations. Suffixed names pollute the legend with entries like "Dolomite-3" instead of
the clean "Dolomite" that the user expects.

**Examples:**
```
Elevation	Ground	Water_Table	Bottom	m
Depth		TOP	Norite	Peridotite	Anorthosite	meters
Elevation		Fill	Mud	"Silty Sand"	Silt	m
Elevation	bathymetry	sediment-bottom	m
```

**Note:** An empty token (double delimiter) before the first surface name is common and
acceptable — it represents the unnamed Y column position.

### Line 3 — Data Counts and Colors

This line contains:

1. **Row count** — the number of data rows that follow
2. **Surface count** — the number of depth/elevation columns per boring (includes the top
   surface plus all layer bottoms)
3. **Color numbers** — one integer per surface, defining the display color for each layer.
   The first two color numbers should typically match (both represent the top surface).

Color numbers are integers that map to EVS's internal color palette.

**Examples:**
```
244	3	1	1	2
182	13	1	1	2	3	1	4	2	4	1	5	6	4	5
140	5	0	0	1	2	3
```

## Column Specification

Each data row contains the following columns in order:

| # | Column | Required | Description |
|---|--------|----------|-------------|
| 1 | X | Yes | X coordinate (easting) in projected coordinate system |
| 2 | Y | Yes | Y coordinate (northing) in projected coordinate system |
| 3 | Surface 1 (Top) | Yes | Ground surface elevation or depth = 0 |
| 4..N | Surface 2..N | Yes | Bottom of each geologic layer (elevation or depth) |
| N+1 | Boring name | Yes | Well/boring identifier |

**Notes:**
- The number of surface columns must match the surface count in Line 3
- In `Elevation` mode, values decrease downward (top surface has highest elevation)
- In `Depth` mode, values increase downward (top surface is typically 0 or a small value,
  with deeper layers having larger depth values)
- Surface values can be replaced with special flags (see below)

## Special Flags

GEO files support special text flags in place of numeric surface values. All flags are
**case-insensitive**.

### Class A — Missing / Incomplete Data

These flags indicate that the boring did not reach a layer, or that data is missing:

| Flag | Description |
|------|-------------|
| `short` | Boring terminated before reaching this layer |
| `terminated` or `term` | Same as `short` |
| `missing` | Data for this layer is missing or unknown |
| `unknown` or `unk` | Same as `missing` |
| `na` | Not applicable / not available |

When a Class A flag is used, all subsequent (deeper) surfaces for that boring must also
use Class A flags. A boring that is `short` at one layer cannot have numeric values for
deeper layers.

### Class B — Pinch-Out

These flags indicate that a geologic layer is absent at a boring location (the layer
has pinched out):

| Flag | Description |
|------|-------------|
| `pinch` | Layer has zero thickness at this location |
| `pinched` | Same as `pinch` |
| `pinch-out` | Same as `pinch` |

A pinch flag is numerically equivalent to the value from the previous column — the layer
has zero thickness. Unlike Class A flags, subsequent layers can still have numeric values
after a pinch.

**Constraint:** A `pinch` flag cannot follow a `short` (or other Class A) flag. If a boring
is short, deeper layers must also be short — they cannot pinch.

## Example File

A file with 5 borings, 5 surfaces (top + 4 layer bottoms), using Elevation mode in meters:

```
X	Y	Top	Bot_0	Bot_1	Bot_2	Bot_3	Bore
Elevation		Fill	Mud	"Silty Sand"	Silt	m
5	5	0	0	1	2	3
51830.49	29848.29	5.260	3.060	pinch	-4.240	-41.000	BOR2-B026
51825.84	29887.89	5.390	3.990	2.290	-0.110	-41.000	BOR3-CP-100
51841.94	29911.92	7.060	3.960	-0.640	-6.840	-41.000	BOR3-CP-101
51853.16	29923.93	8.810	2.210	-1.890	-8.690	-41.000	BOR3-CP-102
51863.70	29939.00	9.350	1.950	-2.150	-10.550	-41.000	BOR3-CP-103
```

**Breakdown:**

- **Line 1:** Column labels (descriptive header).
- **Line 2:** Elevation mode, 5 surface names (unnamed top, Fill, Mud, Silty Sand, Silt),
  meters.
- **Line 3:** 5 data rows, 5 surfaces, color numbers 0, 0, 1, 2, 3.
- **Data rows:** X, Y, top elevation, 4 layer bottom elevations, boring name.
  - BOR2-B026 has a `pinch` for the Mud layer (zero thickness — Mud is absent here).
  - Values decrease downward as expected in Elevation mode.

## Validation Rules

In addition to the [common validation rules in `/evs-data`](/evs-data):

- Row count on Line 3 must match the actual number of data rows
- Surface count on Line 3 must match the number of surface columns in each data row
- Number of color values on Line 3 must equal the surface count
- Number of surface names on Line 2 must match the surface count (if names are provided)
- In `Elevation` mode, numeric surface values should generally decrease from left to right
  (top to bottom) — but this is not strictly enforced when pinch flags are present
- In `Depth` mode, numeric surface values should generally increase from left to right
- Class A flags (`short`, `missing`, etc.) must propagate — once used, all subsequent
  surfaces in that row must also be Class A flags
- Class B flags (`pinch`) cannot follow Class A flags in the same row
- The final (deepest) surface must have at least 3 borings with numeric values to define
  a plane — if fewer than 3, the model cannot be constructed
- Line 1 cannot be blank

## Edge Cases and Gotchas

- Line 1 is a **title/label line**, not parsed for data — but it **cannot be blank**
- The `$W` / `$G` prefix on Line 1 changes how EVS treats the surfaces (as independent
  rather than stacked stratigraphy)
- Surface names on Line 2 may include an empty token for the Y column position — this is
  normal and not a parsing error
- The first two color numbers on Line 3 should typically match — the first colors the top
  surface and the second colors the first layer, which visually share the same boundary
- In `Depth` mode, the top surface value is typically the depth to ground surface from the
  reference point (often 0 or a small positive number)
- All borings must report values for **all** surfaces (using flags where needed) — unlike
  GMF where each surface has independent point sets
- GEO files are inherently **vertical boring** format — X and Y are constant for each
  boring (one row per boring)
- When converting from GEO to GMF, pinch flags become duplicated elevation points on
  consecutive surfaces, and short flags simply omit the boring from deeper surfaces
- Large GEO files with many surfaces can be wide (many columns per row) — this is normal
  but can be difficult to read in text editors without word wrap disabled
- When generating or validating GEO data, look for consecutive columns with **duplicated
  numeric values** — this typically indicates a pinch-out. Use the `pinch` flag instead of
  repeating the elevation/depth value. For example, if layer N bottom = 10.0 and layer N+1
  bottom = 10.0, the second value should be `pinch` instead of `10.0`
