---
name: evs-lithology
description: >-
  Shared concepts, goals, and gotchas for EVS lithology data. Use when working with any
  lithology file type (.lpdv, .lsdv, .pgf) for Earth Volumetric Studio. Covers material
  definition format, material ID ordering strategies, material simplification guidance,
  and common lithology pitfalls. Load this skill alongside the specific file-type skill
  when creating, reading, validating, transforming, or debugging lithology files.
license: Proprietary. C Tech AI Integrations License Agreement, https://ctech.com/downloads/
---

# EVS Lithology Data

Prerequisite: Load `/evs-data` first for shared rules (delimiters, encoding, coordinate
system requirements, etc.) that apply to all EVS data files.

## Overview

Lithology data represents **categorical material classifications** in 3D space, typically
describing the soil or geologic materials encountered during drilling. Each measurement
has a distinct material type associated with it (e.g., Sand, Gravel, Clay, Silt).

Three EVS file types carry lithology data:

- **LPDV** — material classifications at discrete points (`/evs-lpdv`)
- **LSDV** — material classifications over depth intervals/screens (`/evs-lsdv`)
- **PGF** — material classifications as transition points from surface downward (`/evs-pgf`)

This skill documents concepts shared across all three formats. For format-specific rules,
invoke the corresponding skill listed above.

## Goals

### Material Simplification

Lithology data from drilling logs often contains a large number of distinct material types.
Too many materials degrades performance and interpretability in EVS and other 3D modeling
software — the model becomes noisy, color legends become unwieldy, and geostatistical
algorithms struggle to produce meaningful results with sparse per-material populations.

**Proactively assess the material count** when working with lithology data. If the source
data has many materials (roughly more than 8-10 distinct types), flag this to the user and
suggest considering simplification before generating an EVS file. Common simplification
strategies include:

1. **Merge by geologic similarity** — Combine materials that are geologically similar or
   behave similarly in the subsurface model. For example, "Fine Sand", "Medium Sand", and
   "Coarse Sand" could be merged into "Sand". "Silty Clay" and "Clay" could be merged into
   "Clay". Use geologic knowledge to propose groupings, but always confirm with the user.
2. **Merge rare materials** — Materials that appear in very few borings or intervals may not
   contribute meaningfully to the model. Propose merging them into the most geologically
   similar common material.
3. **User-defined grouping** — The user may have domain-specific knowledge about which
   materials should be grouped. Ask the user if they have a preferred grouping scheme.

When simplifying:
- Always present the proposed grouping to the user for approval before applying it
- Document the original material names and the simplification mapping in the generation
  metadata comments
- Renumber material IDs after simplification so they remain sequential starting from 0

Do **not** simplify without the user's explicit agreement. The goal is to raise awareness
and offer assistance, not to silently reduce data fidelity.

#### Simplification and Stratigraphic Modeling

Simplification is generally beneficial for all lithology work, but it becomes a **required
precursor** when the goal is to build a stratigraphic model (GEO or GMF files). Converting
lithology data (PGF/LSDV) to geologic stratigraphy requires interpreting a consistent
top-to-bottom geologic hierarchy across all borings. The more distinct materials present,
the harder this interpretation becomes — and the resulting hierarchy grows vastly more
complex, amplifying errors and ambiguities. Simplify materials *before* attempting any
lithology-to-stratigraphy conversion.

Note that not all sites are appropriate for stratigraphic modeling. Sites with complex,
non-layered geology — such as **karst terrain**, heavily faulted zones, or chaotic
depositional environments — should not be forced into a stratigraphic hierarchy. GEO and
GMF formats assume a layered geology where surfaces can be defined between units. When
the geology is fundamentally non-layered, lithology data should remain in its native
format (PGF, LSDV, or LPDV) and be used directly with EVS tools that do not require
stratigraphic interpretation, such as **Geologic Indicator Kriging (GIK)**.

## Material Definitions

All three lithology formats define materials as **integer-to-name mappings** using a
pipe-delimited format: the material ID (an integer), a pipe character (`|`), and the
material name.

- Material IDs are integers. Sequential IDs starting from 0 are preferred but not required
  — any integer values are valid.
- The pipe character (`|`) separates the ID from the name with no spaces around it.
- Material names are free-form strings. Names must not contain units (see naming
  conventions in `/evs-data`).

**Format-specific quoting rules:**

- **LPDV and LSDV:** Each material definition must be quoted: `"0|Sand"`, `"1|Clay"`
- **PGF:** Material definitions are unquoted unless the name contains spaces:
  `0|Sand`, `"1|Silty Sand"`

Do not confuse the pipe-delimited material definition format with the `@@` analyte name
format used in APDV and AIDV files. These are distinct header conventions for different
data categories.

## Material ID Ordering

When generating a new lithology file and assigning material IDs, **prompt the user** for
their preferred ordering strategy. The available options are:

1. **Geologic similarity (recommended)** — Group materials by geologic properties (grain
   size, consolidation, genesis), then order groups from surface-typical to deep-typical.
   Within each group, sub-order by average depth in the data. This produces IDs where
   similar soil types have adjacent numbers and shallow materials use lower IDs. Use general
   geologic knowledge to classify standard material names. For non-standard or ambiguous
   names, ask the user how to classify them.
2. **By average depth** — Compute the mean midpoint elevation (or depth) for each material
   across all intervals. Sort from shallowest (lowest ID) to deepest (highest ID). Simple
   and data-driven, but does not group geologically similar materials.
3. **Alphabetical** — Sort material names alphabetically. Deterministic and simple, but
   has no geologic meaning.
4. **By frequency** — Most frequently occurring material gets the lowest ID. Useful when
   the most common material should be visually prominent.
5. **By first appearance** — Assign IDs in the order materials are first encountered in
   the data.

If the user has no preference, default to **geologic similarity**.

## Shared Gotchas

- There are **no non-detect or missing flags** for lithology data — every data record
  (point, interval, or transition) must have a valid material integer. This contrasts with
  analytical data (APDV/AIDV) which supports non-detect and missing value flags.
- Material IDs are **integers**, not flags or string values. They must correspond to a
  definition in the file header.
- All three lithology formats support **non-vertical borings** — X and Y coordinates can
  vary along a boring as depth changes, representing the actual 3D borehole trajectory.
