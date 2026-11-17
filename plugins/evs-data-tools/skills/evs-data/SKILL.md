---
name: evs-data
description: >-
  Earth Volumetric Studio (EVS) data file format reference for C Tech Development Corporation.
  EVS is 3D subsurface modeling software — NOT electric vehicles. Use when working with EVS
  borehole data files, environmental site data, geologic modeling data, or any of these file
  types: .apdv, .aidv, .lpdv, .lsdv, .pgf, .geo, .gmf. Covers analyte point data, analyte
  interval data, lithology point data, lithology screen data, pre-geology lithology data,
  borehole geologic stratigraphy, and geologic stratigraphy multi-files. Also use when the
  user mentions EVS, Earth Volumetric Studio, borehole data formatting, or contamination data
  files.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Data Files

## Overview

EVS (Earth Volumetric Studio) is a 3D earth science modeling platform by C Tech Development
Corporation used for environmental visualization and subsurface data analysis. It works with
borehole logs, geophysical surveys, contamination data, geologic stratigraphy, and more.

EVS imports site data from 7 primary structured file types, each with a specific purpose.
They share the common conventions and concepts documented in this skill.

Official file format documentation: https://www.ctech.com/studio_help/file-format-details/index.html

### Sample Files

The EVS Sample Projects contain correctly formatted examples of all file types. If
installed, their path is the `Samples` value under the Windows registry key
`Computer\HKEY_CURRENT_USER\Software\C Tech Development Corporation`.

## File Type Directory

When you need to work with a specific EVS file type, **invoke the corresponding skill** for
detailed format specifications. Do NOT attempt to generate or validate a specific file type
using only the information in this skill.

| Extension | Full Name                          | Skill         | When to Use                                    |
|-----------|------------------------------------|---------------|------------------------------------------------|
| `.apdv`   | Analyte Point Data File            | `/evs-apdv`   | Point-based analyte/contaminant data at specific depths |
| `.aidv`   | Analyte Interval Data File         | `/evs-aidv`   | Interval-based analyte/contaminant data (top/bottom depths) |
| `.lpdv`   | Lithology Point Data File          | `/evs-lpdv`   | Point-based lithology/soil classification at specific depths |
| `.lsdv`   | Lithology Screen Data File         | `/evs-lsdv`   | Screen interval-based lithology data           |
| `.pgf`    | PreGeology File                    | `/evs-pgf`    | Point-based lithology transition data          |
| `.geo`    | Borehole Geologic Stratigraphy     | `/evs-geo`    | Single-file geologic stratigraphy per borehole |
| `.gmf`    | Geologic Stratigraphy Multi-File   | `/evs-gmf`    | Multi-file geologic stratigraphy format        |

## Global Rules

### Precision and Assumptions

Details are critical. When generating, editing, or validating any EVS file, **never assume
values or details that are not explicitly provided**. If any required information is missing
or ambiguous, always stop and ask the user to specify. This includes but is not limited to: coordinate values, depth values, units, analyte names,
borehole IDs, geologic layer names, and any format-specific parameters.

When in doubt, prompt the user. It is always better to ask than to guess.

### Spatial Data and Numeric Precision

**Never round coordinates, depths, elevations, or any numeric data when performing
computations.** Carry full precision through every calculation. Rounding introduces
positional errors that compound in 3D modeling and geostatistical analysis; rounding
coordinates (especially to whole numbers) can move borings or merge distinct borings.
Values are only shortened when written, following "Numeric Precision" below.

### Coordinate System Consistency

All positional data within a file must use one coordinate system and unit of measure.
Coordinates must be in a **projected coordinate system** (e.g., State Plane, UTM).
Latitude/longitude (geographic coordinates) are not valid.

X, Y, and any elevation (Z) or depth values must all use the same unit — e.g., if X and Y
are in meters, Z/elevation/depth must also be meters, not feet. No mixing units in a file.

All EVS files for the same project must use the **same coordinate system and units**;
verify consistency across all output files.

### Delimiters

Except for certain header lines in some file types, all EVS data files are delimited by
spaces, commas, or tabs.

**Multiple consecutive delimiters with no value between them are treated as a single
delimiter.** This means empty fields cannot be represented by adjacent delimiters — they
will be collapsed. For example, all of the following are equivalent and produce three
values (X, Y, Z):

```
X  Y  Z
X,	Y	 Z
X	Y	Z
X Y Z
```

Delimiters can be mixed within a line; any sequence of spaces, commas, and/or tabs between
values is one delimiter.

### Quoted Strings

String values can optionally be wrapped in quotation marks, which are not part of the
value. Quoting lets values contain delimiter characters (spaces, commas, tabs). For example:

- `"1,2-Dimethoxyethane"` — analyte name containing a comma
- `"Boring 4D"` — borehole name containing a space

When writing EVS data files, quote any string value that contains spaces, commas, or tabs.

### Encoding

All EVS data files must be written in UTF-8.

### Missing and Empty Values

There are no empty values: consecutive delimiters collapse into one (see
[Delimiters](#delimiters)), so blank fields cannot exist. Represent any empty or missing
value with the **special flag values** defined in the individual file-type skill.

### Comments

Lines beginning with `#` are comments and are skipped. The `#` must be the first character
of the line.

### Date and Time Formatting

Dates and times must use the **Windows invariant culture** format. Preferred formats:

- **Short date:** `MM/dd/yyyy` (e.g., `03/15/2024`)
- **Short date + time:** `MM/dd/yyyy HH:mm` (e.g., `03/15/2024 14:30`)

Values with a time component contain a space, so they must be quoted
(e.g., `"03/15/2024 14:30"`).

EVS is timezone-agnostic — no timezone information is stored or expected.

### Line Endings

All EVS data files must use Windows/DOS style line endings (CRLF, `\r\n`).

### Case Sensitivity

Values are **case sensitive**. Boring IDs, analyte names, material names, and other strings
must use consistent casing throughout a file — `MW-01` and `mw-01` are two different borings.

If inconsistent casing is detected (e.g., `A-922` and `a-922` in the same file), **stop and
ask the user** whether these are intended to be the same identifier, and if so,
which casing to use.

### Coordinate Unit Abbreviations

Coordinate units must be abbreviated: `m` (meters) or `ft` (feet). Do not use longer forms
such as "meters", "foot", "US_Feet", etc.

### Units Prompting

Unit strings are free-form. EVS recognizes common units automatically; unrecognized units
must be configured by the user in the application.

If units cannot be determined from the source data, **always prompt the user** to specify
them. Do not guess or assume units. Common situations requiring a prompt:

- Coordinate units (meters vs feet) when not stated in the source
- Analyte concentration units (mg/L, ug/L, ppm, etc.)
- Elevation vs depth values

### Elevation vs Depth

**Always prompt the user** to confirm whether Z is elevation or depth below ground
surface, offering a default inferred from the data with the reason (e.g. "`SampleDepth`
looks like depth. Use `Depth`?"). Evidence:

- **Within one boring only:** Z increasing down a boring's samples suggests `Depth`;
  decreasing suggests `Elevation`. Never compare Z across borings: values from different
  locations are unrelated.
- **Column names**, e.g. `Depth`, `SampleDepth`, `Elevation`.
- Z near and below a ground-surface or top elevation, or negative Z, suggests `Elevation`.

### Zero Values

In an EVS file, `0.0` is a measured value, not a non-detect. Whether a zero in source
data is a non-detect depends on its units:

- **Log-processed units** (e.g. mg/kg, mg/L, ug/L, ppm): a zero cannot be log processed,
  so it is likely a non-detect. **Prompt the user**: write it as a non-detect (`<` plus
  detection limit, else `nd`), the default, or keep `0.0`?
- **Other units** (e.g. pH, deg C, mV, ft): zero and negative values are normal. Keep them.
- **Units missing or unclear:** **prompt the user**.

When validating a file, raise a `0.0` in log-processed units as a question, not an error.

### Naming Conventions

Names in EVS files (analyte, surface/layer, material, and boring names) are free-form, with
no required naming lists or CAS numbers. Names **must not contain units**; units belong in
the file header.

**Incorrect:**
- `Benzene_mg/L` — unit embedded in analyte name
- `Sand_m` — unit embedded in surface name

**Correct:**
- `Benzene` (with `mg/L` specified on the units line)
- `Sand` (with `m` specified in the header)

If source names include units, strip them and put the unit in the appropriate header field.

### Depth Conventions

EVS uses **true vertical depth** measured from the **ground surface**. For file types with
measurements along a boring (AIDV, LSDV, PGF, GEO), measurements within each boring must
proceed in a **single consistent direction** — typically top-down from the ground surface.
Do not mix ascending and descending order within a boring.

### Borehole Identification

Boring IDs (also called well or borehole IDs) are free-form strings with no character
restrictions, but IDs containing delimiters must be quoted (see
[Quoted Strings](#quoted-strings)). Keep IDs under 20–30 characters for readability.

In file types where multiple data records share the same boring ID (APDV, AIDV, LSDV,
PGF, GEO), all records for a given boring must be **grouped together** — do not
interleave records from different borings. Within each boring's group, records must be
ordered by depth.

### Numeric Precision

How to write values:

- **Source values** (copied unchanged): write exactly as in the source. Do not add or
  remove decimal places.
- **Computed values:** calculate at full precision, then write them at least as precisely
  as the source values they came from, but no more:
  - Same units (interpolated or derived depths, elevations, averages): use the decimal
    places of the most precise input (ground `5280.4` minus depth `12.25` is `5268.15`).
  - Unit conversions: keep one more significant figure than the source (`3903.24` ft
    becomes `1189.708` m, not `1189.71` or `1189.707552`).
- **Floating-point artifacts:** clean them (`-3.8000000000000007` becomes `-3.8`). If
  that leaves a whole number, keep one trailing zero (`3.0000000000001` becomes `3.0`).
  A source value that was already whole (`4`) is written as-is, without `.0`.
- **Never** write any value (coordinates, depths, elevations, analyte or surface values)
  less precisely than the source. In the same units, that means no fewer decimal places;
  never round to whole numbers when the source has decimals. Units can be ft, m, mm,
  mg/kg or ug/L, so judge precision by the source, not by what seems realistic.

### Output Verification

When generating or modifying any EVS data file, **always show the complete file content
to the user and ask them to confirm it is correct** before writing to disk. Never assume
correctness without explicit user confirmation.

### Generation Metadata Comments

When generating any EVS data file, insert concise `#` comment lines between the header and
the first data row documenting the generation process:

1. **Source** — the original data source (file path, sheet name, database, etc.)
2. **Generated** — date and time of generation
3. **Assumptions and decisions** — any choices made during generation, including:
   - User-specified parameters (units, coordinate mode, etc.)
   - Material ID ordering strategy and groupings (for lithology formats)
   - How non-detect or missing values were handled
   - Any data transformations applied (coordinate reprojection, unit conversion, etc.)
   - Any data excluded or filtered

Format example:
```
# Source: C:\data\site_samples.xlsx (sheet "lith")
# Generated: 2024-03-15 14:30
# Coordinate units: meters (user-specified)
# Material ID ordering: geologic similarity
#   0=SAND, 1=GRAVEL (unconsolidated coarse)
#   2=SILT, 3=CLAY (unconsolidated fine)
#   4=SANDSTONE, 5=MUDSTONE (consolidated)
```

## Data Categories

The 7 file types represent 3 distinct categories of data:

### Analytical Data (APDV, AIDV)

Analytical data represents measured values in 3D space — typically chemical concentrations
(e.g., contaminant levels in soil or groundwater), but can be any measured quantity such as
temperature, geophysical properties, or other analytical results.

The two analytical file types are nearly identical but differ in one critical way:

- **APDV (Analyte Point Data)** — Each measurement is at a **discrete point** (single X, Y,
  Z). Use when samples are collected at specific depth points.
- **AIDV (Analyte Interval Data)** — Each measurement covers a **depth interval** (X, Y with
  Z Top and Z Bottom). Use when samples represent a range of depths — e.g., groundwater
  concentrations along a well screen, where one value represents the whole screen.

**When to choose APDV vs AIDV:** If the data has a single depth per measurement, use APDV.
If the data has a top and bottom depth per measurement, use AIDV.

### Lithology Data (LPDV, LSDV, PGF)

Lithology data represents **categorical material classifications** in 3D space, typically
the soil or geologic materials encountered during drilling. Each measurement has a material
type (e.g., Sand, Gravel, Clay, Silt) — what is present, not a numeric measured value.

For shared lithology concepts — material definitions, ID ordering strategies, material
simplification guidance, and common gotchas — invoke `/evs-lithology`.

The three lithology file types offer different ways to represent material classifications:

- **LPDV (Lithology Point Data)** — Materials at **discrete points** (single X, Y, Z).
- **LSDV (Lithology Screen Data)** — Materials over **depth intervals** (screens): X, Y with
  top and bottom depth. The most feature-rich format, supporting non-vertical borings,
  missing intervals, and borings that do not start at ground surface.
- **PGF (PreGeology File)** — Materials as a sequence of **transition points** from ground
  surface downward; each boring starts at the surface and records where materials change.
  PGF is the older format; LSDV is its modern replacement.

**When to choose:** LSDV is the typical choice for borehole lithology logs due to its
flexibility. PGF is used for legacy data or when the transition-point model is preferred.
LPDV is more commonly used for surface measurements of lithologic data.

### Geologic Stratigraphy Data (GEO, GMF)

Geologic stratigraphy data defines **stratigraphic horizons** — the surfaces between layers
in a layered geology model, typically the ground surface followed by the bottom of each
successive layer downward.

For example, a site with sand at the surface, silt beneath it, and gravel at the bottom
would be defined by 4 horizons:
1. Ground surface (top)
2. Bottom of sand
3. Bottom of silt
4. Bottom of gravel

Both GEO and GMF define these horizon surfaces, but they differ in structure and flexibility:

- **GEO (Borehole Geologic Stratigraphy)** — Boring data as a table: one row per boring,
  one column per horizon. Flag values handle missing data or short borings that did not
  penetrate all layers. Typically used when all measurements come from borings starting at
  the ground surface. Preferred for boring-based data, as it keeps the correlation between
  borings and supports richer visualization in EVS.
- **GMF (Geologic Stratigraphy Multi-File)** — Each horizon is defined independently, with
  its own X, Y locations; no shared locations or boring associations are required. More
  flexible but often less desirable: measurements cannot be correlated to borings and
  visualization options are more limited.

**When to choose GEO vs GMF:** Use GEO when stratigraphy data comes from borings and all
measurements start at the ground surface. Use GMF when horizon data comes from multiple
independent sources with no consistent boring locations across surfaces.

**Single-layer stratigraphy recommendation:** When creating any boring data file (APDV,
AIDV, LPDV, LSDV, or PGF) and no GEO or GMF file exists for the site, **prompt the user**
and ask if a single-layer stratigraphy model should also be generated. It defines just the
ground surface and a model floor from the boring data — no lithology interpretation required
— giving EVS the surface constraints for gridding that are essential for kriging and 3D
visualization of analytical or indicator data. See the Single-Layer
Stratigraphy Model procedure in `/evs-convert` for details.

## Validation Rules

The following validation rules apply to **all** EVS data file types. Individual file-type
skills define additional format-specific rules.

- Coordinate values must be in a projected coordinate system (no lat/lon)
- All coordinate axes must use the same units
- All data rows within a file must have the same number of columns
- Boring names, analyte names, surface names, and material names are case sensitive —
  inconsistent casing should be flagged and confirmed with the user
- Duplicate data (same boring/location with identical values) typically indicates a
  problem — show the user the duplicates. If the values are identical, suggest removing
  one. Otherwise, ask the user how to resolve the conflict.

### File Type Detection

The file extension is the most reliable way to identify an EVS file type. Each format's
unique internal structure can also identify it, but always prefer the extension when available.

### Cross-File Validation

When generating or validating multiple EVS files for the same project, verify that all
files use the **same coordinate system and units** (see
[Coordinate System Consistency](#coordinate-system-consistency)).

If the same boring ID appears in multiple files with different coordinates, **stop and
prompt the user** for clarification. This usually indicates a coordinate system mismatch, which a transformation tool
such as Proj4 or GDAL may correct. **Always confirm with the user** before reprojecting —
clearly state the source and target coordinate systems and get explicit agreement first.

### Common Errors

- **Geographic coordinates used instead of projected** — Lat/lon is not valid. Proj4 or
  GDAL can reproject (e.g., to State Plane, UTM); always confirm the target system with the
  user first.
- **Coordinate system mismatch across files** — All project files must share one coordinate
  system and units. Proj4 or GDAL can transform to a common system; always confirm the source
  and target systems with the user first.
- **Units embedded in names** — Analyte names, surface names, and other identifiers must
  not contain units (see [Naming Conventions](#naming-conventions)).
- **Inconsistent boring ID casing** — Boring IDs are case sensitive. `MW-01` and `mw-01`
  are treated as different borings (see [Case Sensitivity](#case-sensitivity)).

## Sub-Skill Reference

For detailed format specifications, invoke the corresponding skill:

- `.apdv` files: invoke `/evs-apdv` — [SKILL.md](../evs-apdv/SKILL.md)
- `.aidv` files: invoke `/evs-aidv` — [SKILL.md](../evs-aidv/SKILL.md)
- `.lpdv` files: invoke `/evs-lpdv` — [SKILL.md](../evs-lpdv/SKILL.md)
- `.lsdv` files: invoke `/evs-lsdv` — [SKILL.md](../evs-lsdv/SKILL.md)
- `.pgf` files: invoke `/evs-pgf` — [SKILL.md](../evs-pgf/SKILL.md)
- Shared lithology concepts: invoke `/evs-lithology` — [SKILL.md](../evs-lithology/SKILL.md)
- `.geo` files: invoke `/evs-geo` — [SKILL.md](../evs-geo/SKILL.md)
- `.gmf` files: invoke `/evs-gmf` — [SKILL.md](../evs-gmf/SKILL.md)
- Converting between formats: invoke `/evs-convert` — [SKILL.md](../evs-convert/SKILL.md)

For global EVS data rules that apply across all file types, refer to the sections above
in this skill.
