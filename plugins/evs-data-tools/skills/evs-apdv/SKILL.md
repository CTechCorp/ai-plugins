---
name: evs-apdv
description: >-
  Detailed format specification for EVS Analyte Point Data (.apdv) files. Use when creating,
  reading, validating, transforming, or debugging .apdv files for Earth Volumetric Studio.
  These files contain point-based analyte/contaminant concentration data associated with
  borehole locations at specific depths.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Analyte Point Data File (.apdv)

Prerequisite: Load `/evs-data` first for shared rules (delimiters, encoding, coordinate
system requirements, etc.) that apply to all EVS data files.

Official documentation:
https://www.ctech.com/studio_help/file-format-details/apdv-files/index.html

## Purpose

APDV files contain analyte or property data measured at discrete 3D point locations. Each
measurement is associated with an X, Y, Z coordinate. This is typically chemical
concentration data (e.g., contaminant levels in soil or groundwater), but can also include
any measured quantity such as temperature, geophysical properties, hydraulic head, or other
analytical results.

**Use APDV when** each measurement is taken at a specific point in space (single depth).
**Use AIDV instead** when measurements represent an interval (top and bottom depth).

## File Format Overview

An APDV file is a UTF-8 text file with the `.apdv` extension. It consists of:

1. A **3-line header** defining analyte names, coordinate mode, and data dimensions
2. **Data rows** — one row per measurement point

Standard EVS delimiter rules apply to data rows (spaces, commas, tabs are interchangeable;
consecutive delimiters collapse). Line 1 of the header has a special delimiter rule (see
below).

## Header Specification

### Line 1 — Analyte Names

Defines the names of the analyte columns in the data rows.

- Any text before `@@` is ignored and can be used as a descriptive header
- `@@` marks the start of the analyte name list
- One name per analyte must follow, **space-delimited** (this line uses space-only
  delimiting, not the standard EVS delimiter rules)
- The number of names must match the analyte count specified on line 3
- It is **strongly recommended** to include analyte names in all APDV files
- It is **recommended** to wrap each name in quotation marks

**Examples:**

```
X  Y  Z  @@"Benzene" "Toluene" "Xylene"
@@"VOC" "Lead" "Arsenic"
Head Data  @@"head"
@@"1,2-Dichloroethane" "Carbon tetrachloride"
```

**Naming rules:**
- Names containing spaces, commas, or tabs **must** be quoted
- Underscores in unquoted names are automatically converted to spaces on import
- Lines starting with `#` are comments and are skipped

### Line 2 — Coordinate Specification

Defines how Z values are interpreted and the coordinate units.

- **First value:** one of the following (case-insensitive):
  - `Elevation` — Z values are true elevations (e.g., feet above sea level)
  - `Depth` — Z values are positive numbers representing depth below ground surface
  - `2D` — No Z coordinate; Z column is omitted from data rows entirely
- **Second value:** coordinate units (`m` or `ft`)

**Examples:**

```
Elevation m
Depth ft
2D m
```

**Determining Elevation vs Depth:** If not specified by the user, follow "Elevation vs
Depth" in `/evs-data`: **prompt the user**, offering a default inferred from the data. Compare
Z values only within the same boring, never across borings.

### Line 3 — Data Specifications

Defines the number of data rows, the number of analyte columns, and the units for each
analyte.

- **First value:** number of data rows (integer), or `All` to indicate all remaining lines
- **Second value:** number of analyte value columns per data row (integer)
- **Remaining values:** unit label for each analyte column (e.g., `mg/kg`, `mg/l`, `ppm`)

**Examples:**

```
168 4 mg/kg mg/kg mg/kg mg/kg
All 1 mg/l
104 1 mg/l
664 8 mg/kg mg/kg mg/kg mg/kg mg/kg mg/kg mg/kg mg/kg
```

## Column Specification

Each data row contains the following columns in order:

| # | Column | Required | Description |
|---|--------|----------|-------------|
| 1 | X | Yes | X coordinate (easting) in projected coordinate system |
| 2 | Y | Yes | Y coordinate (northing) in projected coordinate system |
| 3 | Z | Yes (unless `2D`) | Elevation or depth value, depending on line 2 mode |
| 4+ | Analyte values | Yes | One or more values; count must match line 3 |
| | Boring name | Recommended | Well/boring identifier (no spaces unless quoted) |
| | Top elevation | Recommended | Ground surface elevation at the boring location |

**Notes:**
- If boring name is present on any row, it must be present on all rows
- If top elevation is present on any row, it must be present on all rows
- In `2D` mode, the Z column is omitted entirely

## Special Flag Values

### Non-Detect Flags

Used when an analyte was tested for but not detected. Flags are **not case-sensitive**.

| Flag | Description |
|------|-------------|
| `<value` | Non-detect with detection limit (preferred). Example: `<0.01` |
| `nd` | Non-detect abbreviation (preferred when no detection limit is known) |
| `nondetect` | Non-detect (no detection limit specified) |
| `non-detect` | Same as `nondetect` |

**Important:** `0.0` is a valid measured value and is **not** equivalent to non-detect.
Always use the flags above for non-detect results.
When source data has zeros, follow "Zero Values" in `/evs-data`: whether a zero is a
non-detect depends on its units. In log-processed units (e.g. mg/kg, ug/L) **prompt the
user**, offering non-detect as the default; in other units (e.g. pH, deg C) keep it.

### Missing/Unmeasured Flags

Used when an analyte was not measured at all for a given sample. Flags are **not
case-sensitive**.

| Flag | Description |
|------|-------------|
| `nm` | Not measured (preferred) |
| `missing` | Missing data |
| `unmeasured` | Unmeasured |
| `not-measured` | Same as `unmeasured` |
| `unknown` or `unk` | Unknown value |
| `na` | Not applicable / not available |

## Example File

A file with 3 borings, 2 analytes (Benzene and Xylene), using Elevation mode in meters:

```
X  Y  Z  @@"Benzene" "Xylene"  Boring  Top
Elevation m
9 2 mg/kg mg/kg
196344.90 205644.65 1059.93 0.025 0.05 MW-01 1061.43
196344.90 205644.65 1058.93 0.14 nm MW-01 1061.43
196344.90 205644.65 1056.93 <0.01 0.05 MW-01 1061.43
196361.50 205650.72 1060.22 0.025 0.05 MW-02 1061.72
196361.50 205650.72 1058.72 0.025 0.05 MW-02 1061.72
196361.50 205650.72 1056.22 0.025 0.05 MW-02 1061.72
196284.95 205650.01 1060.18 0.025 76.7 MW-03 1061.68
196284.95 205650.01 1057.68 0.025 2.4 MW-03 1061.68
196284.95 205650.01 1055.68 0.14 0.05 MW-03 1061.68
```

**Line-by-line breakdown:**

- **Line 1:** Text before `@@` is ignored. `@@` starts analyte names: "Benzene" and
  "Xylene". "Boring" and "Top" after the analyte names are also ignored (only the count
  from line 3 determines how many analyte names are read).
- **Line 2:** Z values are elevations, units are meters.
- **Line 3:** 9 data rows, 2 analyte values per row, both in mg/kg.
- **Data rows:** X, Y, Z (elevation), Benzene value, Xylene value, boring name, top
  elevation. Note `nm` (not measured) and `<0.01` (non-detect) flags in use.

## Validation Rules

In addition to the [common validation rules in `/evs-data`](/evs-data):

- Number of analyte values per data row must match the count on header line 3
- Number of analyte names on line 1 (after `@@`) must match the analyte count on line 3
- If boring name is present on any data row, it must be present on all rows
- If top elevation is present on any data row, it must be present on all rows
- Z values must be consistent with the coordinate mode specified on line 2

## Edge Cases and Gotchas

- `@@` on line 1 starts the analyte name list; all text before `@@` is ignored
- Analyte names on line 1 are **space-delimited only** — this is an exception to the
  standard EVS delimiter rules where tabs and commas also work
- Underscores in unquoted analyte names are converted to spaces on import
- In `2D` mode, the Z column is omitted from data rows entirely. In `Elevation` or `Depth`
  mode, the Z column is required but values can be 0.0
- The `<` non-detect prefix can appear with or without a leading zero (e.g., `<0.01` or
  `<.0002` are both valid)
- `0.0` is a valid analyte value and is **not** equivalent to non-detect. Use `<`, `nd`,
  or `nondetect` flags for non-detect results
- The row count on line 3 can be the literal word `All` instead of an integer
- Boring names containing spaces must be quoted (e.g., `"Boring 4D"`)
