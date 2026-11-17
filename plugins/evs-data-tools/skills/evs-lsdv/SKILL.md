---
name: evs-lsdv
description: >-
  Detailed format specification for EVS Lithology Screen Data (.lsdv) files. Use when creating,
  reading, validating, transforming, or debugging .lsdv files for Earth Volumetric Studio.
  These files contain screen interval-based lithology data associated with borehole locations,
  describing soil and geologic characteristics over depth ranges.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Lithology Screen Data File (.lsdv)

Prerequisites: Load `/evs-data` first for shared rules (delimiters, encoding, coordinate
system requirements, etc.) that apply to all EVS data files. Load `/evs-lithology` for
shared lithology concepts (material definitions, ID ordering, simplification guidance).

Official documentation:
https://www.ctech.com/studio_help/file-format-details/lsdv-lithology-screen-data-value-file-format/index.html

## Purpose

LSDV files contain lithology (material classification) data over depth intervals (screens).
Each interval is assigned a single material type represented as an integer that maps to a
named material defined in the header. This is the preferred format for borehole lithology
logs where soil or rock types are recorded over depth ranges.

LSDV is a feature-rich replacement for the older PGF lithology format, with support for
non-vertical borings, missing intervals, and lithology data that does not start at ground
surface.

**Use LSDV when** material classifications span depth intervals (screens).
**Use LPDV instead** when each material classification is at a specific point in space
(e.g., surface measurements).

LSDV files can be refined and converted to LPDV for merging with other point-based
datasets.

## File Format Overview

An LSDV file is a UTF-8 text file with the `.lsdv` extension. It consists of:

1. A **single-line header** defining coordinate mode, material mappings, and units
2. **Data rows** — one row per screen interval (no row count specified)

Standard EVS delimiter rules apply (spaces, commas, tabs are interchangeable; consecutive
delimiters collapse).

## Header Specification

The header is a single line (the first non-comment line) containing three parts in order:

### Part 1 — Coordinate Mode

- `Elevation` — Z values are true elevations (e.g., feet above sea level)
- `Depth` — Z values are positive numbers representing depth below ground surface

### Part 2 — Material Definitions

One or more material definitions, each as a quoted string in the format `"number|name"`:

- The number is an integer identifying the material
- The name is the material/lithology label
- Sequential IDs starting from 0 are preferred but not required — any integer values are
  valid
- Each definition must be quoted

### Material ID Ordering

See [Material ID Ordering in `/evs-lithology`](/evs-lithology) for the 5 available
ordering strategies. Prompt the user when generating a new file.

### Part 3 — Coordinate Units

- `m` or `ft`

**Examples:**

```
Elevation "0|SAND" "1|SANDSTONE" "2|GRAVEL" "3|MUDSTONE" "4|LIMESTONE" m
Depth "0|Soil" "1|Silty-Sand" "2|Sand" "3|Sandstone" "4|Clays" m
Elevation "0|Plain" "1|Land" "2|Terra Rossa" "3|Quartzite" "4|Granite" "5|Breccia" "6|Andesite" m
```

**Determining Elevation vs Depth:** If not specified by the user, follow "Elevation vs
Depth" in `/evs-data`: **prompt the user**, offering a default inferred from the data. Compare
Z values only within the same boring, never across borings.

## Column Specification

Each data row contains exactly **9 columns** in order:

| # | Column | Required | Description |
|---|--------|----------|-------------|
| 1 | X Top | Yes | X coordinate at the top of the interval |
| 2 | Y Top | Yes | Y coordinate at the top of the interval |
| 3 | Z Top | Yes | Elevation or depth at the top of the interval |
| 4 | X Bottom | Yes | X coordinate at the bottom of the interval |
| 5 | Y Bottom | Yes | Y coordinate at the bottom of the interval |
| 6 | Z Bottom | Yes | Elevation or depth at the bottom of the interval |
| 7 | Material | Yes | Integer material code matching a header definition |
| 8 | Boring name | Yes | Well/boring identifier (no spaces unless quoted) |
| 9 | Ground elevation | Yes | Ground surface elevation at the boring location |

**Notes:**
- All 9 columns are required for every data row
- For vertical borings, X Top = X Bottom and Y Top = Y Bottom
- For non-vertical borings (slant holes), the X, Y coordinates can differ between top and
  bottom, allowing the boring trajectory to be represented
- In `Elevation` mode, Z Top > Z Bottom (top is at a higher elevation)
- In `Depth` mode, Z Top < Z Bottom (top is shallower / closer to surface)
- Each boring name corresponds to a unique starting X, Y location

## Example File

A file with 3 borings and 3 material types, using Elevation mode in meters:

```
Elevation "0|SAND" "1|SANDSTONE" "2|GRAVEL" m
736133.27 1637594.56 1190.20 736133.27 1637594.56 1189.80 0 "720B0001" 1190.2
736133.27 1637594.56 1189.80 736133.27 1637594.56 1188.20 1 "720B0001" 1190.2
736133.27 1637594.56 1188.20 736133.27 1637594.56 1187.80 0 "720B0001" 1190.2
736133.27 1637594.56 1187.80 736133.27 1637594.56 1185.60 1 "720B0001" 1190.2
736286.27 1637647.56 1191.20 736286.27 1637647.56 1190.20 0 "720B0002" 1191.2
736286.27 1637647.56 1190.20 736286.27 1637647.56 1187.20 1 "720B0002" 1191.2
736286.27 1637647.56 1187.20 736286.27 1637647.56 1186.20 2 "720B0002" 1191.2
736918.27 1637179.55 1194.20 736918.27 1637179.55 1190.80 0 "720B0004" 1194.2
736918.27 1637179.55 1190.80 736918.27 1637179.55 1185.20 1 "720B0004" 1194.2
```

**Breakdown:**

- **Header:** Elevation mode, 3 materials (SAND=0, SANDSTONE=1, GRAVEL=2), meters.
- **Data rows:** X/Y/Z top, X/Y/Z bottom, material integer, boring name, ground
  elevation. For these vertical borings, X and Y are the same at top and bottom.
  Z Top > Z Bottom in Elevation mode. Intervals are contiguous within each boring.

## Validation Rules

In addition to the [common validation rules in `/evs-data`](/evs-data):

- Material integer in each data row must match a definition in the header
- Material IDs in the header should preferably be sequential integers starting from 0, but
  non-sequential IDs are allowed
- Every data row must have exactly 9 columns
- Intervals within the same boring must not overlap — if overlapping intervals are
  detected, show the user the specific overlapping rows and ask how they want to resolve
  the issue (do not auto-correct)
- In `Elevation` mode, Z Top must be greater than Z Bottom for each row
- In `Depth` mode, Z Top must be less than Z Bottom for each row (top is shallower)
- Each boring name must correspond to a unique starting X, Y location
- Data must be in proper order — EVS cannot sort the data because borings may be
  non-vertical or upward-trending

## Merging Consecutive Identical Materials

When creating, cleaning, or preprocessing LSDV data, **merge adjacent intervals that have
the same material** within each boring. Replace two consecutive intervals with the same
material with a single interval spanning from the first interval's top to the second
interval's bottom. Two consecutive Sand intervals are really one Sand layer — the boundary
between them is not geologically meaningful. This applies to any workflow that produces or
modifies LSDV data, not just conversions. Always merge before further processing (e.g.,
hierarchy construction, export, or visualization).

## Edge Cases and Gotchas

- There is **no row count** in the header — data rows start immediately after the header
  line, and all remaining non-comment lines are treated as data
- See [Shared Gotchas in `/evs-lithology`](/evs-lithology) for material definition format
  notes, non-detect/missing flag rules, and other cross-format lithology gotchas
- LSDV supports **non-vertical borings** — the X, Y coordinates at the top and bottom of
  an interval can differ, which is why X, Y, Z are specified twice per row
- Intervals within a boring **do not need to be contiguous** — gaps between intervals are
  valid and represent missing data spans where lithology was not recorded
- Intervals **do not need to start at ground surface** — borings can begin at any depth
- Material names containing spaces or special characters are protected by the required
  quoting of the `"number|name"` definition
- LSDV files can be refined and converted to LPDV format for merging with point-based
  datasets
