---
name: evs-lpdv
description: >-
  Detailed format specification for EVS Lithology Point Data (.lpdv) files. Use when creating,
  reading, validating, transforming, or debugging .lpdv files for Earth Volumetric Studio.
  These files contain point-based lithology and soil classification data associated with
  borehole locations at specific depths.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Lithology Point Data File (.lpdv)

Prerequisites: Load `/evs-data` first for shared rules (delimiters, encoding, coordinate
system requirements, etc.) that apply to all EVS data files. Load `/evs-lithology` for
shared lithology concepts (material definitions, ID ordering, simplification guidance).

Official documentation:
https://www.ctech.com/studio_help/file-format-details/lpdv-lithology-point-data-value-file-format/index.html

## Purpose

LPDV files contain lithology (material classification) data at discrete 3D point locations.
Each point is assigned a single material type represented as an integer that maps to a
named material defined in the header. This is typically used for soil or rock type
classifications at specific depths along a boring, or for surface lithology measurements.

LPDV files are point-based — EVS uses the point data directly without further refinement.
This makes them suitable for surface lithology data or any situation where material
classifications are tied to specific points rather than intervals.

**Use LPDV when** each material classification is at a specific point in space.
**Use LSDV instead** when material classifications span depth intervals (screens).

LSDV files can be converted to LPDV for merging datasets, but LPDV files cannot be further
refined after creation.

## File Format Overview

An LPDV file is a UTF-8 text file with the `.lpdv` extension. It consists of:

1. A **single-line header** defining coordinate mode, material mappings, and units
2. **Data rows** — one row per measurement point (no row count specified)

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

### Part 3 — Coordinate Units

- `m` or `ft`

**Examples:**

```
Elevation "0|SAND" "1|SANDSTONE" "2|GRAVEL" m
Depth "0|Soil" "1|Silty-Sand" "2|Sand" "3|Sandstone" "4|Clays" m
Elevation "0|Plain" "1|Land" "2|Terra Rossa" "3|Quartzite" "4|Granite" "5|Breccia" "6|Andesite" m
```

**Determining Elevation vs Depth:** If not specified by the user, follow "Elevation vs
Depth" in `/evs-data`: **prompt the user**, offering a default inferred from the data. Compare
Z values only within the same boring, never across borings.

## Column Specification

Each data row contains the following columns in order:

| # | Column | Required | Description |
|---|--------|----------|-------------|
| 1 | X | Yes | X coordinate (easting) in projected coordinate system |
| 2 | Y | Yes | Y coordinate (northing) in projected coordinate system |
| 3 | Z | Yes | Elevation or depth value, depending on header mode |
| 4 | Material | Yes | Integer material code matching a header definition |
| 5 | Boring name | Optional | Well/boring identifier (no spaces unless quoted) |
| 6 | Ground elevation | Optional | Ground surface elevation at the boring location |

**Notes:**
- If boring name is present on any row, it must be present on all rows
- If ground elevation is present on any row, it must be present on all rows
- Ground elevation requires boring name to also be present
- The material integer must correspond to a number defined in the header

## Example File

A file with 2 borings and 3 material types, using Elevation mode in meters:

```
Elevation "0|SAND" "1|SANDSTONE" "2|GRAVEL" m
736133.27 1637594.56 1190.19 0 720B0001 1190.2
736133.27 1637594.56 1189.81 0 720B0001 1190.2
736133.27 1637594.56 1189.79 1 720B0001 1190.2
736133.27 1637594.56 1188.21 0 720B0001 1190.2
736133.27 1637594.56 1187.79 1 720B0001 1190.2
736133.27 1637594.56 1185.59 2 720B0001 1190.2
736286.27 1637647.56 1191.20 0 720B0002 1191.2
736286.27 1637647.56 1189.80 0 720B0002 1191.2
736286.27 1637647.56 1187.20 2 720B0002 1191.2
```

**Breakdown:**

- **Header:** Elevation mode, 3 materials (SAND=0, SANDSTONE=1, GRAVEL=2), meters.
- **Data rows:** X, Y, Z (elevation), material integer, boring name, ground elevation.
  Each point represents a single material classification at that depth.

## Validation Rules

In addition to the [common validation rules in `/evs-data`](/evs-data):

- Material integer in each data row must match a definition in the header
- Material IDs in the header should preferably be sequential integers starting from 0, but
  non-sequential IDs are allowed
- If boring name is present on any data row, it must be present on all rows
- If ground elevation is present on any data row, it must be present on all rows

## Edge Cases and Gotchas

- There is **no row count** in the header — data rows start immediately after the header
  line, and all remaining non-comment lines are treated as data
- See [Shared Gotchas in `/evs-lithology`](/evs-lithology) for material definition format
  notes, non-detect/missing flag rules, and other cross-format lithology gotchas
- LPDV files cannot be further refined after creation (unlike LSDV which can be refined
  and converted to LPDV)
- Material names containing spaces or special characters are protected by the required
  quoting of the `"number|name"` definition
- LPDV supports non-vertical borings — X and Y values can vary along a boring as depth
  changes, representing the actual 3D trajectory of the borehole
