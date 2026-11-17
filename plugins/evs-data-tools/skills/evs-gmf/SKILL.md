---
name: evs-gmf
description: >-
  Detailed format specification for EVS Geologic Stratigraphy Multi-File (.gmf) files. Use when
  creating, reading, validating, transforming, or debugging .gmf files for Earth Volumetric Studio.
  These files contain geologic stratigraphy data where each surface is defined by its own
  independent set of X, Y, Z points, all within a single file.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Geologic Stratigraphy Multi-File (.gmf)

Prerequisite: Load `/evs-data` first for shared rules (delimiters, encoding, coordinate
system requirements, etc.) that apply to all EVS data files.

Official documentation:
https://www.ctech.com/studio_help/file-format-details/gmf_file/index.html

## Purpose

GMF files describe geologic stratigraphy where each surface has its own independent set of
X, Y, Z points. Unlike GEO files, surfaces do not need to share the same boring locations
or even have the same number of points. This makes GMF ideal for data from mixed sources
(boreholes, geophysical surveys, topographic data) or sites where different surfaces have
been sampled at different locations.

Despite the name "Multi-File", the GMF format is a **single file** containing multiple
surface definitions. The name refers to the fact that each surface is defined independently
(as if it were its own file), rather than being tied to a common set of boring locations.

As with GEO, each surface represents the bottom of a geologic layer (except the first
surface, which is the ground/top surface). The surface name defines the material of the
layer **above** that surface — the layer between surface[i] and surface[i+1] is the material
named by surface[i+1].

**Use GMF when** surfaces have independent point distributions, when data comes from
non-borehole sources, or when different surfaces are sampled at different locations.
**Use GEO instead** when all borings share the same set of geologic surfaces and every
boring reports values for every surface.

Surface names in GMF files can represent dates or times, making the format suitable for
temporal data such as changing groundwater elevations over time.

## File Format Overview

A GMF file is a UTF-8 text file with the `.gmf` extension. It consists of:

1. A **units declaration** line (required, exactly once)
2. One or more **surface blocks**, each containing a surface declaration followed by
   X, Y, Z data points
3. An **`end`** terminator line
4. Lines starting with `#` are comments and are ignored

Standard EVS delimiter rules apply (spaces, commas, tabs are interchangeable; consecutive
delimiters collapse).

## Units Declaration

The units line is **required** and must appear exactly once, before any surface declarations:

```
units m
units ft
```

The keyword `units` is followed by the unit identifier (`m`, `ft`, `meters`, `feet`, etc.).

## Surface Declaration

Each surface block begins with a surface declaration line:

```
surface [material_number] [surface_name]
```

- **`surface`** — the literal keyword (case-insensitive)
- **material_number** — an integer identifying the surface's color/material assignment in
  EVS. Multiple surfaces can share the same material number.
- **surface_name** — the name of the surface. Names containing spaces must be enclosed in
  quotation marks.

**Examples:**
```
surface 0 ground
surface 1 clay
surface 2 "silty sand"
surface 0 "2019-03-15 12:00"
```

**Notes:**
- Material numbers do **not** need to be sequential or unique — multiple surfaces can share
  the same material number (they will share the same display color)
- Surface names can represent **dates/times** for temporal data (e.g., water table
  elevations at different measurement times). Date/time entries with spaces must be quoted.
  When surface names are dates/times, use the `$W` flag on the `end` line (see
  [File Termination](#file-termination)).
- The first surface declared is treated as the topmost surface

## Data Points

After each surface declaration, one or more X, Y, Z data points follow, one per line:

```
X_coordinate	Y_coordinate	Z_coordinate
```

- Each surface can have a different number of data points
- Each surface can have different X, Y locations
- Z values are always elevations (GMF files do not have a Depth mode)
- Delimiters can be spaces, tabs, or commas

## File Termination

The file must end with a line starting with the keyword `end`:

```
end
end $W
```

This signals the end of all surface data. The `end` keyword is required.

**`$W` flag:** When surfaces represent unrelated data such as water table elevations at
different times (surface names are dates/times), append `$W` after `end`. This tells EVS
to disable hierarchy checks and kriging in thickness space, treating the surfaces as
independent measurements rather than a stacked stratigraphic sequence.

## Comments

Lines beginning with `#` are comments and are ignored by EVS. Comments can appear anywhere
in the file.

```
# This is a comment
```

## Example File

A file with 3 surfaces, using meters, where each surface has a different number of points:

```
units m
surface 0 ground
151016.67	137446.04	1130.66
151024.85	137447.77	1130.60
151031.60	137443.04	1130.42
151038.07	137434.66	1130.95
151042.44	137446.67	1130.55
surface 0 sand
151016.67	137446.04	1123.16
151024.85	137447.77	1123.10
151031.60	137443.04	1122.92
surface 1 clay
151016.67	137446.04	1116.66
151024.85	137447.77	1116.60
151031.60	137443.04	1116.42
151038.07	137434.66	1116.95
end
```

**Breakdown:**

- **Line 1:** Units are meters.
- **Surface "ground":** 5 points defining the ground surface, material number 0.
- **Surface "sand":** 3 points defining the bottom of the sand layer, material number 0
  (same color as ground — these share a visual boundary).
- **Surface "clay":** 4 points defining the bottom of the clay layer, material number 1.
- Each surface has a different number of data points and can have different locations.
- The file ends with `end`.

## Pinch-Outs in GMF

GMF files do **not** use special flags like `pinch` or `short` (those are GEO-only). Instead,
a pinch-out is represented by **duplicating the Z elevation** between two consecutive surfaces
at the same X, Y location. When two surfaces have identical X, Y, Z values, the layer
between them has zero thickness at that point.

## Validation Rules

In addition to the [common validation rules in `/evs-data`](/evs-data):

- The `end` keyword must be present at the end of the file
- Every surface declaration must be followed by at least one data point
- Each data point must have exactly 3 values (X, Y, Z)
- The `units` line is required and must appear exactly once, before any surface declarations

## Edge Cases and Gotchas

- Despite the name "Multi-File", GMF is a **single file** format — the "multi" refers to
  each surface being independently defined
- GMF files use **elevation only** — there is no Depth mode. Z values are always true
  elevations.
- There are **no special flags** (`short`, `pinch`, `missing`, etc.) in GMF files — these
  are GEO-only features. Pinch-outs are handled by duplicating elevations on consecutive
  surfaces.
- Material numbers are **not required to be sequential** and can repeat across surfaces —
  they only control display color, not surface ordering
- The first surface is treated as the topmost — surface ordering in the file defines the
  stratigraphic sequence
- Each surface is fully independent — different surfaces can have entirely different point
  counts and X, Y locations
- Surface names can be dates/times (useful for temporal water table data). Date/time names
  with spaces must be quoted.
- When converting from GEO to GMF:
  - Each boring contributes one point per surface (where it has numeric values)
  - `pinch` flags become duplicated Z values on consecutive surfaces
  - `short` / `missing` flags mean the boring is simply omitted from that surface
- EVS may append a SHA checksum hash after the `end` line — this is for internal integrity
  checking and should be preserved if present, but is not required when creating new files
