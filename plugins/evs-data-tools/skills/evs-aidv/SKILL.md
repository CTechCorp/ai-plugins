---
name: evs-aidv
description: >-
  Detailed format specification for EVS Analyte Interval Data (.aidv) files. Use when creating,
  reading, validating, transforming, or debugging .aidv files for Earth Volumetric Studio.
  These files contain interval-based analyte/contaminant concentration data associated with
  borehole locations over depth ranges (top and bottom depths).
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Analyte Interval Data File (.aidv)

Prerequisite: Load `/evs-data` first for shared rules (delimiters, encoding, coordinate
system requirements, etc.) that apply to all EVS data files.

Official documentation:
https://www.ctech.com/studio_help/file-format-details/aidv-files/index.html

## Purpose

AIDV files contain analyte or property data measured over depth intervals (well screens).
Each measurement is associated with an X, Y coordinate and a Z Top / Z Bottom pair
defining the screened or sampled interval. This is typically chemical concentration data
(e.g., contaminant levels in groundwater across a well screen), but can also include any
measured quantity such as temperature, geophysical properties, or other analytical results.

When EVS imports an AIDV file, it uses the max gap parameter (line 2) to determine how to
place sample points within each interval for 3D kriging. If the interval length is less
than or equal to max gap, a single sample is centered in the interval. If the interval is
longer than max gap, multiple equally-spaced samples are placed along the interval.

**Use AIDV when** each measurement represents a depth interval (top and bottom depth).
**Use APDV instead** when each measurement is taken at a specific point (single depth).

## File Format Overview

An AIDV file is a UTF-8 text file with the `.aidv` extension. It consists of:

1. A **3-line header** defining analyte names, coordinate mode with max gap, and data
   dimensions
2. **Data rows** — one row per measurement interval

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
- It is **strongly recommended** to include analyte names in all AIDV files
- It is **recommended** to wrap each name in quotation marks

**Examples:**

```
X  Y  Ztop  Zbot  @@"Benzene" "Toluene" "Xylene"
@@"VOC" "Lead" "Arsenic"
@@"TCE"
@@"1,2-Dichloroethane" "Carbon tetrachloride"
```

**Naming rules:**
- Names containing spaces, commas, or tabs **must** be quoted
- Underscores in unquoted names are automatically converted to spaces on import
- Lines starting with `#` are comments and are skipped

### Line 2 — Coordinate Specification and Max Gap

Defines how Z values are interpreted, the maximum sample gap for kriging, and the
coordinate units.

- **First value:** one of the following (case-insensitive):
  - `Elevation` — Z Top and Z Bottom are true elevations (e.g., feet above sea level).
    Z Top > Z Bottom for each interval.
  - `Depth` — Z Top and Z Bottom are positive numbers representing depth below ground
    surface. Z Top < Z Bottom for each interval (top is shallower).
- **Second value:** max gap — a real number defining the maximum distance between sample
  points that EVS places within an interval during kriging. When an interval's length is
  less than or equal to max gap, a single centered sample is used. Longer intervals receive
  multiple equally-spaced samples.
- **Third value:** coordinate units (`m` or `ft`)

**Examples:**

```
Elevation 4 m
Depth 20.0 ft
Elevation 30 m
Depth 2 ft
```

**Determining Elevation vs Depth:** If not specified by the user, follow "Elevation vs
Depth" in `/evs-data`: **prompt the user**, offering a default inferred from the data. Compare
Z values only within the same boring, never across borings.

**Determining Max Gap:** Always ask the user to specify the max gap value. If the user
does not have a preferred value, offer to compute a default from the data. The default
max gap is the **minimum** of:

1. 90% of the 25th percentile of interval lengths (|Z Top - Z Bottom|)
2. 20% of the 5th percentile of XY distances between borings (Euclidean distance in X, Y
   between each pair of unique boring locations)

### Line 3 — Data Specifications

Defines the number of data rows, the number of analyte columns, and the units for each
analyte.

- **First value:** number of data rows (integer), or `All` to indicate all remaining lines
- **Second value:** number of analyte value columns per data row (integer)
- **Remaining values:** unit label for each analyte column (e.g., `mg/kg`, `mg/l`, `ug/L`,
  `ppm`, `%`)

**Examples:**

```
631 3 mg/l mg/l mg/l
All 1 mg/l
865 1 ug/kg
864 3 % % %
78 42 mg/l mg/l mg/l ...
```

## Column Specification

Each data row contains the following columns in order:

| # | Column | Required | Description |
|---|--------|----------|-------------|
| 1 | X | Yes | X coordinate (easting) in projected coordinate system |
| 2 | Y | Yes | Y coordinate (northing) in projected coordinate system |
| 3 | Z Top | Yes | Top of the interval (elevation or depth, per line 2 mode) |
| 4 | Z Bottom | Yes | Bottom of the interval (elevation or depth, per line 2 mode) |
| 5+ | Analyte values | Yes | One or more values; count must match line 3 |
| | Boring name | Recommended | Well/boring identifier (no spaces unless quoted) |
| | Top elevation | Recommended | Ground surface elevation at the boring location |

**Notes:**
- If boring name is present on any row, it must be present on all rows
- If top elevation is present on any row, it must be present on all rows
- In `Elevation` mode, Z Top > Z Bottom (top of interval is at a higher elevation)
- In `Depth` mode, Z Top < Z Bottom (top of interval is shallower / closer to surface)

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

A file with 3 borings, 2 analytes (Benzene and Xylene), using Elevation mode in meters
with a max gap of 5:

```
X  Y  Ztop  Zbot  @@"Benzene" "Xylene"  Boring  Top
Elevation 5 m
9 2 mg/l mg/l
3708938.71 2338190.14 3324.86 3314.86 <0.004 <0.003 NMW_23S 3345.01
3709033.59 2338785.61 3324.60 3290.80 <0.004 <0.003 GWMW_40 3348.00
3709033.59 2338785.61 3290.80 3280.80 <0.004 0.15 GWMW_40 3348.00
3709227.67 2338189.02 3325.59 3315.59 0.005 nm NMW_29s 3345.74
3709236.89 2338104.68 3324.59 3314.59 <0.004 <0.003 GWMW_39 3346.69
3709239.79 2338101.79 3324.37 3315.27 <0.004 <0.003 GWMW_97 3346.47
3709239.79 2338101.79 3315.27 3311.37 0.012 <0.003 GWMW_97 3346.47
3709239.79 2338101.79 3311.37 3307.47 <0.004 0.08 GWMW_97 3346.47
3709239.79 2338101.79 3307.47 3302.92 <0.004 <0.003 GWMW_97 3346.47
```

**Line-by-line breakdown:**

- **Line 1:** Text before `@@` is ignored. `@@` starts analyte names: "Benzene" and
  "Xylene". "Boring" and "Top" after the analyte names are also ignored (only the count
  from line 3 determines how many analyte names are read).
- **Line 2:** Z values are elevations, max gap is 5 units, units are meters.
- **Line 3:** 9 data rows, 2 analyte values per row, both in mg/l.
- **Data rows:** X, Y, Z Top (elevation), Z Bottom (elevation), Benzene value, Xylene
  value, boring name, top elevation. Note Z Top > Z Bottom in Elevation mode. Note `nm`
  (not measured) and `<0.004` / `<0.003` (non-detect) flags in use. Boring GWMW_97 has
  multiple intervals at the same X, Y location.

## Validation Rules

In addition to the [common validation rules in `/evs-data`](/evs-data):

- Number of analyte values per data row must match the count on header line 3
- Number of analyte names on line 1 (after `@@`) must match the analyte count on line 3
- If boring name is present on any data row, it must be present on all rows
- If top elevation is present on any data row, it must be present on all rows
- In `Elevation` mode, Z Top must be greater than Z Bottom for each row
- In `Depth` mode, Z Top must be less than Z Bottom for each row (top is shallower)
- Intervals within the same boring must not overlap — if overlapping intervals are
  detected, show the user the specific overlapping rows and ask how they want to resolve
  the issue (do not auto-correct)
- Max gap on line 2 must be a positive number

## Edge Cases and Gotchas

- `@@` on line 1 starts the analyte name list; all text before `@@` is ignored
- Analyte names on line 1 are **space-delimited only** — this is an exception to the
  standard EVS delimiter rules where tabs and commas also work
- Underscores in unquoted analyte names are converted to spaces on import
- The `<` non-detect prefix can appear with or without a leading zero (e.g., `<0.004` or
  `<.0002` are both valid)
- `0.0` is a valid analyte value and is **not** equivalent to non-detect. Use `<`, `nd`,
  or `nondetect` flags for non-detect results
- The row count on line 3 can be the literal word `All` instead of an integer
- Boring names containing spaces must be quoted (e.g., `"Boring 4D"`)
- Max gap affects how EVS distributes sample points within intervals for kriging — it does
  not affect the data itself, only the interpolation behavior
- A single boring can have multiple intervals (rows) at the same X, Y location with
  different Z Top / Z Bottom values — but intervals within the same boring **must not
  overlap**. For example, in Elevation mode, intervals `0 to -2` and `-1 to -3` overlap
  and are invalid; they should be `0 to -2` and `-2 to -3` (or non-overlapping ranges)
- AIDV does **not** have a `2D` mode — Z Top and Z Bottom are always required
