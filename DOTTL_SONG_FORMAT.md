# Dottl Song Format Specification

**Version:** 3.1 (based on internal schema v5)
**File extension:** `.dottl` (app-neutral) or `.drumlet` (drumlet-flavored — see [App-Specific File Extensions](#app-specific-file-extensions))
**MIME type:** `application/dottl+json` (formerly `application/json`; both are accepted)
**Encoding:** UTF-8

---

## Overview

A `.dottl` file is a JSON document that represents a musical composition created in the Dottl ecosystem. It uses a grid-based model where notes are placed at column/row coordinates, with time flowing left-to-right (columns) and pitch arranged vertically (rows mapped to chromatic note names). The format supports multiple instrument layers, sustain/legato connections between notes, chord annotations, and playback metadata.

`.dottl` files are designed to be portable across any app in the Dottl ecosystem — grid editors, play-along games, visualizers, educational tools, and more.

---

## Top-Level Structure

```json
{
  "version": 5,
  "projectName": "My Song",
  "bpm": 120,
  "divisor": 2,
  "transposition": 0,
  "difficulty": "intermediate",
  "timeSignature": { "numerator": 4, "denominator": "1/4" },
  "chords": [...],
  "markers": [...],
  "sections": [...],
  "layers": [...]
}
```

| Field           | Type     | Required | Description |
|-----------------|----------|----------|-------------|
| `version`       | `5`      | Yes      | Schema version. Always `5` for this spec. |
| `projectName`   | string   | Yes      | Human-readable song name. May be empty. |
| `bpm`           | number   | Yes      | Tempo in beats per minute. Range: `20–300`. |
| `divisor`       | `1 \| 2 \| 3 \| 4` | Yes | Grid subdivision per beat. See [Timing Model](#timing-model). |
| `transposition` | number   | Yes      | Semitone offset applied at playback. Range: `-12` to `+12`. `0` = no transposition. |
| `difficulty`    | `easy \| intermediate \| advanced \| null` | Yes | Difficulty rating for the song. `null` = unrated. |
| `timeSignature` | `{ numerator: number, denominator: string }` | No | Time signature. See [Time Signature](#time-signature). Defaults to `{ "numerator": 4, "denominator": "1/4" }` if omitted. |
| `chords`        | Chord[]  | No       | Chord annotations for display at specific grid columns. See [Chords](#chords). May be empty or omitted. |
| `markers`       | Marker[] | No       | Repeat and loop markers. See [Markers](#markers). May be empty or omitted. |
| `sections`      | Section[] | No      | Named song sections. See [Sections](#sections). May be empty or omitted. |
| `layers`        | Layer[]  | Yes      | One or more instrument layers. Must contain at least one layer. |

---

## Timing Model

The grid uses a column-based timing system. Each column represents a subdivision of a beat. **Column 1 is beat 1** — the start of the song. Column 0 is a regular grid line before the first bar, available for pickup/anacrusis notes.

| `divisor` | Columns per beat | Musical value    |
|-----------|-----------------|------------------|
| `1`       | 1               | Quarter notes    |
| `2`       | 2               | Eighth notes     |
| `3`       | 3               | Eighth triplets  |
| `4`       | 4               | Sixteenth notes  |

**Column-to-time conversion:**

```
time_seconds = column / divisor * (60 / bpm)
duration_seconds = sustainCells / divisor * (60 / bpm)
```

**Bar structure:** Beats per bar comes from the `timeSignature.numerator` (defaults to `4`). Columns per bar = `numerator * divisor`. Bar lines (beat 1) appear at columns `1, 1 + numerator*divisor, 1 + 2*numerator*divisor, ...`

---

## Time Signature

Optional top-level field describing the song's time signature. When omitted, consumers should default to `{ "numerator": 4, "denominator": "1/4" }` (4/4).

```json
{
  "numerator": 7,
  "denominator": "1/8"
}
```

| Field         | Type   | Required | Description |
|---------------|--------|----------|-------------|
| `numerator`   | number | Yes      | Beats per bar. Common values: `2, 3, 4, 5, 6, 7, 9, 12`. |
| `denominator` | string | Yes      | Note value that gets the beat, expressed as a fraction string. Common values: `"1/2"`, `"1/4"`, `"1/8"`, `"1/16"`. |

The time signature is informational for display and bar-line computation. It does not change column-to-time conversion (which is governed by `bpm` and `divisor`).

There is no fixed number of columns — the grid extends as far right as notes are placed. The total duration of a song is determined by the rightmost note (including its sustain).

---

## Layer

Each layer represents an independent instrument track.

```json
{
  "id": "a1b2c3d4",
  "name": "Piano",
  "color": "#88a7f8",
  "instrumentCategory": "Keys",
  "smplrLibrary": "SplendidGrandPiano",
  "smplrPatch": "Splendid Grand Piano",
  "volume": 70,
  "reverb": 20,
  "notes": [...],
  "lines": [...]
}
```

| Field                | Type               | Required | Description |
|----------------------|--------------------|----------|-------------|
| `id`                 | string             | Yes      | Unique identifier (UUID recommended). |
| `name`               | string             | Yes      | Display name for the layer. |
| `color`              | string             | Yes      | CSS color string for visual tinting. Empty string = default/no tint. |
| `instrumentCategory` | InstrumentCategory | Yes      | General instrument category. See [Instrument Categories](#instrument-categories). |
| `smplrLibrary`       | SmplrLibrary       | No       | The [smplr](https://github.com/danigb/smplr) constructor/library used. See [smplr Fields](#smplr-fields). |
| `smplrPatch`         | string             | Yes      | Specific instrument patch name. For smplr-based apps, this is the smplr instrument name. For other apps, treat as a hint. |
| `volume`             | number             | Yes      | Playback volume. Range: `0–100`. |
| `reverb`             | number             | Yes      | Reverb effect amount. Range: `0–100`. |
| `notes`              | Note[]             | Yes      | Array of placed notes. May be empty. |
| `lines`              | SustainLine[]      | Yes      | Array of sustain/legato connections. May be empty. |

### Instrument Categories

Human-readable categories that describe the general type of instrument. Apps should use these to select an appropriate sound from whatever audio library they use.

| `instrumentCategory` | Description |
|----------------------|-------------|
| `Keys`               | Piano, keyboard, and organ instruments (acoustic piano, electric piano, harpsichord, accordion, etc.) |
| `Guitars`            | Guitars, bass guitars, and plucked string instruments (acoustic guitar, electric guitar, bass, banjo, sitar, harp, etc.) |
| `Strings`            | Bowed string instruments, string ensembles, and choirs (violin, cello, string ensemble, mellotron strings, choir, etc.) |
| `Drums`              | Drum machines, percussion kits, and pitched percussion (TR-808, taiko, timpani, steel drums, etc.) |
| `Horns`              | Woodwind and brass instruments, including synth versions (trumpet, sax, flute, clarinet, trombone, mellotron brass, etc.) |
| `Other`              | Everything else — mallets (vibraphone, xylophone, marimba), FX, pads, leads, sound effects, and any instrument not covered above |

**Implementation note:** Apps that don't support a given category should fall back gracefully — play the notes with any available instrument rather than failing.

### smplr Fields

The `smplrLibrary` and `smplrPatch` fields provide specific sound-reproduction hints for apps that use the [smplr](https://github.com/danigb/smplr) audio library. Apps that don't use smplr should ignore `smplrLibrary` and use `instrumentCategory` to pick an appropriate sound. `smplrPatch` can still serve as a human-readable hint for the specific sound (e.g., "CP80" tells you it's a Yamaha CP80 electric piano).

| `smplrLibrary`       | smplr Constructor    | Typical Category |
|----------------------|---------------------|------------------|
| `SplendidGrandPiano` | `SplendidGrandPiano` | Keys            |
| `ElectricPiano`      | `ElectricPiano`      | Keys            |
| `Mallet`             | `Mallet`             | Other           |
| `Mellotron`          | `Mellotron`          | Strings / Horns |
| `Smolken`            | `Smolken`            | Guitars         |
| `DrumMachine`        | `DrumMachine`        | Drums           |
| `Soundfont`          | `Soundfont`          | Any             |
| `Sampler`            | `Sampler`            | Any             |

Note: Mellotron patches span multiple categories (strings, brass, woodwinds). The `instrumentCategory` determines the UI grouping; `smplrLibrary` determines the audio engine.

---

## Note

A note placed on the grid.

```json
{
  "id": "n1",
  "name": "C#/Db",
  "col": 4,
  "row": 7,
  "octave": 4,
  "isRoot": true,
  "isStartNote": false,
  "sustainCells": 2,
  "velocity": 96,
  "subCol": 1,
  "subCount": 3
}
```

| Field          | Type     | Required | Description |
|----------------|----------|----------|-------------|
| `id`           | string   | Yes      | Unique identifier within the layer. |
| `name`         | NoteName | Yes      | Chromatic pitch name. See [Note Names](#note-names). |
| `col`          | number   | Yes      | Horizontal grid position. Represents time. **Column 1 = beat 1.** Column 0 is reserved for pickup/anacrusis notes. |
| `row`          | number   | Yes      | Vertical grid position (0-indexed). Represents the row on the visual grid. |
| `octave`       | number   | Yes      | Musical octave. Range: `0–8`. Octave 4 = middle C octave. |
| `isRoot`       | boolean  | Yes      | Whether this note is a chord root — a harmonic hint for analysis and display. See [Root Notes vs Start Notes](#root-notes-vs-start-notes). |
| `isStartNote`  | boolean  | No       | Whether this note is the octave anchor for the layer. See [Root Notes vs Start Notes](#root-notes-vs-start-notes). Defaults to `false` if omitted. |
| `sustainCells` | number   | Yes      | Duration extension in grid cells beyond the note's initial cell. `0` = single-cell note. |
| `velocity`     | number   | No       | Per-note dynamics. Range: `0–127` (MIDI convention). When omitted, consumers should use a sensible default (e.g. `96` ≈ mezzo-forte). See [Velocity](#velocity). |
| `subCol`       | number   | No       | 0-indexed sub-position within the cell, when the cell contains multiple evenly-spaced hits (stutter, flam, ratamacue, drag). Requires `subCount`. See [Sub-Cell Hits](#sub-cell-hits). |
| `subCount`     | number   | No       | Total number of evenly-spaced sub-hits in the cell. Required when `subCol` is present. Range: `2–8`. |

### Velocity

`velocity` is an optional per-note dynamics value following MIDI convention:

- `0` — silent (rest equivalent)
- `1–31` — pianissimo to piano (pp–p)
- `32–63` — mezzo-piano (mp)
- `64–95` — mezzo-forte (mf)
- `96–127` — forte to fortissimo (f–fff)

Apps that don't support per-note velocity should ignore the field and use a uniform default. Apps that support it but read a file without `velocity` should treat all notes as a single default velocity (recommended: `96`).

### Sub-Cell Hits

`subCol` and `subCount` represent multiple evenly-spaced hits **within** a single grid cell — the building block for stutter beats, flams, drags, ratamacues, and similar rudiments that fall between the main grid divisions.

The two fields always travel together: a note with `subCount: N` is the *k*-th of *N* evenly-spaced hits in its cell, where `k = subCol`.

**Sub-cell timing:**

```
sub_time_seconds = (col - 1 + subCol / subCount) / divisor * (60 / bpm)
```

A note **without** `subCol`/`subCount` is a single hit landing exactly on the cell boundary (`subCol = 0`, `subCount = 1` implicitly).

**Examples (in a `divisor: 4` / sixteenth grid):**

| Pattern        | Notes in cell                                  |
|----------------|------------------------------------------------|
| Single hit     | One note, no `subCol`/`subCount`               |
| Flam (2-hit)   | Two notes at `subCol: 0, subCount: 2` and `subCol: 1, subCount: 2` |
| Triplet stutter| Three notes at `subCol: 0\|1\|2, subCount: 3`  |
| Drag (4-hit)   | Four notes at `subCol: 0\|1\|2\|3, subCount: 4`|

A cell may contain at most one `subCount` value across its sub-hits — mixing different subdivisions of the same cell is undefined behavior.

Apps that don't support sub-cell hits should fall back gracefully: render each sub-hit as a single hit at the parent cell, or pick the first (`subCol: 0`) and ignore the rest.

### Root Notes vs Start Notes

These two boolean flags serve different purposes and should not be confused:

**`isRoot`** (harmonic) — Marks a note as the root of a chord. Multiple notes can be roots (one per chord). Used by apps for:
- Visual distinction (highlighting chord roots on the grid)
- Harmonic analysis (inferring chord names, key signatures)
- Educational display (showing which note is the tonal center)

**`isStartNote`** (structural) — Marks the octave anchor note for a layer. At most one note per layer should have `isStartNote: true`. Used by apps for:
- Computing octaves of newly placed notes relative to this anchor
- Establishing the pitch reference point when composing interactively
- Identifying the "home base" note of the layer

In many songs, the first note placed is both the start note and a chord root — but they are independent concepts. An imported MIDI file may have chord roots but no start note (octaves are already computed). A composition tool may set a start note that isn't a chord root.

**When `isStartNote` is absent or all false:** Apps should compute octaves from neighboring notes rather than prompting for a starting note. This is the expected state for imported files.

### Note Names

The 12 chromatic pitch classes, using combined sharp/flat notation:

```
C, C#/Db, D, D#/Eb, E, F, F#/Gb, G, G#/Ab, A, Bb, B
```

This is an exhaustive enum — no other values are valid.

### MIDI Pitch Conversion

To convert a note to a MIDI pitch number:

```
midi_pitch = (octave + 1) * 12 + semitone_offset + transposition
```

Where `semitone_offset` is:

| NoteName | Offset |
|----------|--------|
| C        | 0      |
| C#/Db    | 1      |
| D        | 2      |
| D#/Eb    | 3      |
| E        | 4      |
| F        | 5      |
| F#/Gb    | 6      |
| G        | 7      |
| G#/Ab    | 8      |
| A        | 9      |
| Bb       | 10     |
| B        | 11     |

Example: C4 = `(4+1)*12 + 0` = MIDI 60 (middle C).

---

## Sustain Line

Connects notes for sustain or legato phrasing.

```json
{
  "id": "l1",
  "fromNoteId": "n1",
  "toNoteId": "n2",
  "cells": 3,
  "style": "solid"
}
```

| Field        | Type              | Required | Description |
|--------------|-------------------|----------|-------------|
| `id`         | string            | Yes      | Unique identifier within the layer. |
| `fromNoteId` | string            | Yes      | ID of the source note. Must reference a note in the same layer. |
| `toNoteId`   | string \| null    | Yes      | ID of the destination note, or `null` for a pure horizontal sustain (no target). |
| `cells`      | number            | Yes      | Horizontal span in grid cells. |
| `style`      | `solid \| dashed` | Yes      | Visual style. `solid` = legato/sustain. `dashed` = stylistic variation (e.g., staccato hint). |

---

## Chords

Chord annotations specify chord names at specific grid columns for display purposes (e.g., showing "Am" or "Cmaj7" above the grid). Chords are purely informational — they don't affect playback.

```json
{
  "col": 0,
  "name": "Am",
  "display": "Am"
}
```

| Field     | Type   | Required | Description |
|-----------|--------|----------|-------------|
| `col`     | number | Yes      | Grid column where the chord begins (0-indexed). |
| `name`    | string | Yes      | Chord name in standard notation (e.g., `"Am"`, `"Cmaj7"`, `"F#dim"`, `"Bb/D"`). |
| `display` | string | No       | Optional display override. If omitted, apps should display `name`. Useful for simplified notation (e.g., `name: "Cmaj7"`, `display: "C"`). |

**Chord names** should follow standard notation conventions:
- Root note: `C`, `C#`, `Db`, `D`, etc.
- Quality: `m` (minor), `maj` (major, often omitted), `dim`, `aug`, `sus2`, `sus4`
- Extensions: `7`, `maj7`, `9`, `11`, `13`, `add9`, `6`
- Bass note (slash chords): `/E`, `/Bb`

Examples: `"C"`, `"Am"`, `"F#m7"`, `"Bbmaj7"`, `"Gsus4"`, `"D/F#"`, `"Edim"`

Apps that don't support chord display should ignore this field.

---

## Markers

Markers define repeat and loop boundaries for playback control. They are placed on beat-1 columns (bar lines).

```json
{
  "id": "m1",
  "type": "repeat-start",
  "col": 1,
  "repeatCount": 2
}
```

| Field         | Type       | Required | Description |
|---------------|------------|----------|-------------|
| `id`          | string     | Yes      | Unique identifier. |
| `type`        | MarkerType | Yes      | One of: `repeat-start`, `repeat-end`, `loop-start`, `loop-end`. |
| `col`         | number     | Yes      | Grid column where the marker is placed. Must be a beat-1 column. |
| `repeatCount` | number     | No       | Only for `repeat-end`. Number of additional times to replay the section (1 = play once extra, 2 = twice extra, 3 = three times extra). Range: `1–3`. Defaults to `1` if omitted. |

### Marker Types

| Type           | Visual                     | Description |
|----------------|----------------------------|-------------|
| `repeat-start` | Bright red line, → arrow   | Start of a repeat section. |
| `repeat-end`   | Bright red line, ← arrow   | End of a repeat section. Paired with the nearest `repeat-start` to its left. |
| `loop-start`   | Purple dashed line, → arrow | Start of a loop section. |
| `loop-end`     | Purple dashed line, ← arrow | End of a loop section. |

### Repeat Semantics

A `repeat-start` at column A and `repeat-end` at column B with `repeatCount: N` means: play columns A through B once (normal pass), then replay the same section N additional times (total plays = N + 1).

### Loop Semantics

Loop markers define a section that replays indefinitely until the user stops playback. At most one loop pair per song. Loop start must be at a lower column than loop end.

### Validation Rules for Markers

1. Each `repeat-start` must have a matching `repeat-end` to its right (no nesting)
2. Each `repeat-end` must have a preceding `repeat-start`
3. At most one `loop-start` and one `loop-end` per song
4. `loop-start.col` must be less than `loop-end.col`
5. All marker `col` values must be beat-1 columns

---

## Sections

Named song sections for display and arrangement purposes. Sections mark the start of a named region (e.g., "Intro", "Verse", "Chorus").

```json
{
  "id": "s1",
  "col": 1,
  "name": "Verse"
}
```

| Field  | Type   | Required | Description |
|--------|--------|----------|-------------|
| `id`   | string | Yes      | Unique identifier. |
| `col`  | number | Yes      | Grid column where the section starts. Must be a beat-1 column. |
| `name` | string | Yes      | Section name. Common values: `Intro`, `Verse`, `Pre-Chorus`, `Chorus`, `Bridge`, `Outro`, `A`, `B`, `C`. Custom names are allowed. |

A section's range extends from its `col` to the column before the next section's `col` (or to the end of the song if it is the last section).

### Arrangement Export

Apps may support an arrangement feature that reorders and repeats sections. When exporting an arrangement, the output is a standard `.dottl` file with all notes "baked in" — sections are expanded into sequential columns with repeats flattened into actual note data. The exported file has no markers or sections (it is a standalone linear composition).

---

## Versioning & Migration

### Current Schema Version: 5 (Spec Doc: 3.1)

This spec describes schema version 5. Apps should write `version: 5` files exclusively.

The spec doc itself is versioned independently — schema v5 has gone through doc revisions 3.0 → 3.1 as additive optional fields are clarified or added. New optional fields do not require a schema bump; existing readers ignore them safely.

### Changes in spec doc 3.1 (additive, schema stays at v5)

- **Added optional `velocity` on Note** — per-note dynamics, MIDI range `0–127`. Promoted from the Extension Points list into the Note table.
- **Added optional `subCol` + `subCount` on Note** — sub-cell hits for stutter beats, flams, drags, and similar rudiments. See [Sub-Cell Hits](#sub-cell-hits).
- **Added optional `timeSignature` at top level** — explicit time signature. Promoted from Extension Points. Defaults to 4/4 when omitted.
- **Added `Sampler` to the `smplrLibrary` enum** — for layers driven by user-loaded samples rather than a built-in patch.
- **Clarified `col 1 = beat 1`** in the Note `col` description (was previously only stated in [Timing Model](#timing-model)).

All 3.1 additions are backward-compatible: schema v5 readers built against doc 3.0 will simply ignore the new optional fields.

### Changes in v5 (from v4)

- **Updated `instrumentCategory` values** to match the app's current categories: `Keys`, `Guitars`, `Strings`, `Drums`, `Horns`, `Other` (replacing `Piano`, `Mallet`, `Strings`, `Bass`, `Drums`, `Soundfont`)
- **Added `isStartNote` field on notes** — optional boolean that marks the octave anchor note for a layer, separating structural purpose from `isRoot` (harmonic)
- **Added `chords` array** at the top level for chord name annotations at specific grid columns
- **Clarified `col 1` = beat 1** — column 0 is a regular grid line before the first bar
- **Added `markers` array** at the top level for repeat and loop markers
- **Added `sections` array** at the top level for named song sections

### Changes in v4 (from v3)

- **Renamed `instrumentType` → `instrumentCategory`** with human-readable values
- **Renamed `instrumentName` → `smplrPatch`**
- **Added optional `smplrLibrary` field**

### Reading Older Versions

Apps should support migrating older formats on load:

**v4 → v5:**
- Map old v4 category names to v5:

  | v4 `instrumentCategory` | v5 `instrumentCategory` |
  |---|---|
  | `Piano` | `Keys` |
  | `Mallet` | `Other` |
  | `Strings` | `Strings` |
  | `Bass` | `Guitars` |
  | `Drums` | `Drums` |
  | `Soundfont` | `Other` |

- `isStartNote` defaults to `false` for all notes (v4 files don't have it)
- `chords` defaults to `[]` (v4 files don't have it)

**v3 → v4:**
- Map `instrumentType` to `instrumentCategory`
- Rename `instrumentName` to `smplrPatch`
- Derive `smplrLibrary` from v3 fields

**v2 → v3:**
- Wrap all notes and lines into a single default layer
- Set `projectName: ""`, `transposition: 0`
- Use default instrument

**v1 → v3:**
- Convert `connectedToId` on notes to separate `lines` array
- Then apply v2 → v3 migration

Then apply v3 → v4 → v5 migration chain.

### Future Versions

The `version` field allows forward evolution. Apps encountering an unknown version should:
1. Warn the user that the file was created with a newer format
2. Attempt a best-effort load (ignore unknown fields)
3. Never silently overwrite a file with a version they don't fully understand

---

## Validation Rules

A valid `.dottl` file must satisfy:

1. `version` must be `5` (or a recognized legacy version)
2. `bpm` must be a number in `[20, 300]`
3. `divisor` must be one of `1, 2, 3, 4`
4. `transposition` must be an integer in `[-12, 12]`
5. `difficulty` must be one of `easy`, `intermediate`, `advanced`, or `null`
6. `layers` must be a non-empty array
7. All `id` fields must be unique within their scope (notes within a layer, lines within a layer, layers within the song)
8. `fromNoteId` in a SustainLine must reference a valid note `id` in the same layer
9. `toNoteId` must reference a valid note `id` in the same layer, or be `null`
10. `name` must be one of the 12 valid NoteName values
11. `octave` must be an integer in `[0, 8]`
12. `volume` and `reverb` must be numbers in `[0, 100]`
13. `instrumentCategory` must be one of: `Keys`, `Guitars`, `Strings`, `Drums`, `Horns`, `Other`
14. `smplrLibrary`, if present, must be one of: `SplendidGrandPiano`, `ElectricPiano`, `Mallet`, `Mellotron`, `Smolken`, `DrumMachine`, `Soundfont`, `Sampler`
15. At most one note per layer may have `isStartNote: true`
16. Chord `col` values should be non-negative integers
17. `col` values in notes must be non-negative integers
18. Marker `col` values must be beat-1 columns (col = 1 + n × colsPerBar)
19. Marker `type` must be one of: `repeat-start`, `repeat-end`, `loop-start`, `loop-end`
20. At most one `loop-start` and one `loop-end` per file
21. Section `col` values must be beat-1 columns
22. Section `name` must be a non-empty string
23. `velocity`, if present, must be an integer in `[0, 127]`
24. `subCol` and `subCount` must both be present or both absent on a note
25. When present, `subCount` must be an integer in `[2, 8]` and `subCol` must be an integer in `[0, subCount - 1]`
26. All sub-hits within the same cell of the same layer must share the same `subCount`
27. `timeSignature.numerator`, if present, must be a positive integer; `timeSignature.denominator` must be a fraction string of the form `"1/N"` where `N` is a power of 2 in `[1, 64]`

---

## Minimal Example

A single C major chord (C4, E4, G4) at 120 BPM with a chord annotation:

```json
{
  "version": 5,
  "projectName": "C Major Chord",
  "bpm": 120,
  "divisor": 2,
  "transposition": 0,
  "difficulty": "easy",
  "chords": [
    { "col": 0, "name": "C" }
  ],
  "layers": [
    {
      "id": "layer-1",
      "name": "Piano",
      "color": "",
      "instrumentCategory": "Keys",
      "smplrLibrary": "SplendidGrandPiano",
      "smplrPatch": "Splendid Grand Piano",
      "volume": 70,
      "reverb": 20,
      "notes": [
        { "id": "n1", "name": "C", "col": 0, "row": 0, "octave": 4, "isRoot": true, "isStartNote": true, "sustainCells": 0 },
        { "id": "n2", "name": "E", "col": 0, "row": 4, "octave": 4, "isRoot": false, "sustainCells": 0 },
        { "id": "n3", "name": "G", "col": 0, "row": 7, "octave": 4, "isRoot": false, "sustainCells": 0 }
      ],
      "lines": []
    }
  ]
}
```

---

## Interoperability Guidelines

### For Consumers (readers)

- Be lenient: ignore unknown top-level or nested fields
- Support loading all schema versions (v1, v2, v3, v4, v5) via migration
- If `smplrPatch` is not recognized, use a sensible default for the `instrumentCategory`
- If `smplrLibrary` is missing, use `instrumentCategory` to pick an appropriate sound
- If `isStartNote` is missing on all notes, compute octaves from neighboring notes
- If `chords` is missing, treat as empty
- If `markers` is missing, treat as empty (no repeats/loops)
- If `sections` is missing, treat as empty (no named sections)
- If `timeSignature` is missing, default to 4/4
- If `velocity` is missing on a note, use a uniform default (recommended: `96`)
- If `subCol`/`subCount` are missing, treat the note as a single hit at the cell boundary; if present but unsupported, render the sub-hit as a single hit at the parent cell or take only `subCol: 0`
- Legacy files will have `instrumentType`/`instrumentName` or old category names — migrate them

### For Producers (writers)

- Always write `version: 5`
- Always include all required fields — do not omit fields with default values
- Include `smplrLibrary` when the sound was produced with smplr (recommended but optional)
- Set `isStartNote: true` on the octave anchor note when the app uses one; omit or set `false` otherwise
- Include `chords` when chord annotations are available; omit or set `[]` otherwise
- Include `markers` when repeat/loop markers are present; omit or set `[]` otherwise
- Include `sections` when named sections are present; omit or set `[]` otherwise
- Include `timeSignature` when the song is in any meter other than 4/4; omit for 4/4 to keep files compact
- Include `velocity` on notes when the app supports per-note dynamics; omit for uniform-velocity songs
- Include `subCol` + `subCount` together when writing sub-cell hits (stutter, flam, drag, etc.); never write one without the other
- Use stable UUIDs for `id` fields (avoid sequential integers that could collide across apps)
- Sanitize `projectName` for use as a filename: lowercase, replace non-alphanumeric with hyphens, strip leading/trailing hyphens, fall back to `"untitled"`

### File Naming Convention

```
{sanitized-project-name}.dottl     # app-neutral
{sanitized-project-name}.drumlet   # drumlet-flavored (see below)
```

Example: `"My Cool Song!"` → `my-cool-song.dottl`

### App-Specific File Extensions

Apps that write a dottl-spec payload **plus** their own round-trip metadata in the [extensions](#extension-points) block MAY use a custom extension to advertise that fact at the filesystem level. The file is still a valid `.dottl` document — readers can rename `.drumlet` → `.dottl` and load it normally; the extra `extensions.{appname}` block will be ignored by apps that don't recognize it.

| Extension  | Contents | Producer |
|------------|----------|----------|
| `.dottl`   | Pure dottl-spec, no `extensions` block (or only foreign extensions) | App-neutral, transcribers, foreign apps |
| `.drumlet` | dottl-spec + `extensions.drumlet` (round-trippable Drumlet UI state) | [Drumlet](https://drumlet.app/) |

Apps SHOULD accept any of `.dottl`, `.drumlet`, or other `.{app}` variants on import — the file content is the source of truth, not the extension. Apps SHOULD prefer the `.dottl` extension when their export contains no app-specific extensions block.

---

## Extension Points

The format is intentionally minimal. Future versions or app-specific extensions may add:

- **Key signature** — Currently inferred from root notes
- **Metadata** — Author, creation date, tags, license
- **Grid dimensions** — Explicit row-to-pitch mapping for custom tunings
- **Microtiming** — Per-note timing offsets independent of the grid (swing/humanize/groove)

Apps may store additional data in a top-level `extensions` object. Other apps should ignore unrecognized extensions:

```json
{
  "version": 5,
  "extensions": {
    "com.example.myapp": { ... }
  },
  ...
}
```
