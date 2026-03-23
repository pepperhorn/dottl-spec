# Dottl Song Format Specification

**Version:** 1.0 (based on internal schema v3)
**File extension:** `.dottl`
**MIME type:** `application/json` (JSON with `.dottl` extension)
**Encoding:** UTF-8

---

## Overview

A `.dottl` file is a JSON document that represents a musical composition created in the Dottl ecosystem. It uses a grid-based model where notes are placed at column/row coordinates, with time flowing left-to-right (columns) and pitch arranged vertically (rows mapped to chromatic note names). The format supports multiple instrument layers, sustain/legato connections between notes, and playback metadata.

`.dottl` files are designed to be portable across any app in the Dottl ecosystem — grid editors, play-along games, visualizers, educational tools, and more.

---

## Top-Level Structure

```json
{
  "version": 3,
  "projectName": "My Song",
  "bpm": 120,
  "divisor": 2,
  "transposition": 0,
  "difficulty": "intermediate",
  "layers": [...]
}
```

| Field           | Type     | Required | Description |
|-----------------|----------|----------|-------------|
| `version`       | `3`      | Yes      | Schema version. Always `3` for this spec. |
| `projectName`   | string   | Yes      | Human-readable song name. May be empty. |
| `bpm`           | number   | Yes      | Tempo in beats per minute. Range: `20–300`. |
| `divisor`       | `1 \| 2 \| 3 \| 4` | Yes | Grid subdivision per beat. See [Timing Model](#timing-model). |
| `transposition` | number   | Yes      | Semitone offset applied at playback. Range: `-12` to `+12`. `0` = no transposition. |
| `difficulty`    | `easy \| intermediate \| advanced` | Yes | Difficulty rating for the song. |
| `layers`        | Layer[]  | Yes      | One or more instrument layers. Must contain at least one layer. |

---

## Timing Model

The grid uses a column-based timing system. Each column represents a subdivision of a beat:

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

There is no fixed number of columns — the grid extends as far right as notes are placed. The total duration of a song is determined by the rightmost note (including its sustain).

---

## Layer

Each layer represents an independent instrument track.

```json
{
  "id": "a1b2c3d4",
  "name": "Piano",
  "color": "#88a7f8",
  "instrumentType": "keys",
  "instrumentName": "Splendid Grand Piano",
  "volume": 70,
  "reverb": 20,
  "notes": [...],
  "lines": [...]
}
```

| Field            | Type           | Required | Description |
|------------------|----------------|----------|-------------|
| `id`             | string         | Yes      | Unique identifier (UUID recommended). |
| `name`           | string         | Yes      | Display name for the layer. |
| `color`          | string         | Yes      | CSS color string for visual tinting. Empty string = default/no tint. |
| `instrumentType` | InstrumentType | Yes      | Instrument category. See [Instrument Types](#instrument-types). |
| `instrumentName` | string         | Yes      | Specific instrument within the category. |
| `volume`         | number         | Yes      | Playback volume. Range: `0–100`. |
| `reverb`         | number         | Yes      | Reverb effect amount. Range: `0–100`. |
| `notes`          | Note[]         | Yes      | Array of placed notes. May be empty. |
| `lines`          | SustainLine[]  | Yes      | Array of sustain/legato connections. May be empty. |

### Instrument Types

Canonical instrument types:

| `instrumentType` | Description |
|-------------------|-------------|
| `keys`            | Piano/keyboard instruments |
| `mallet`          | Vibraphone, marimba, etc. |
| `mellotron`       | Mellotron tape-replay instruments |
| `smolken`         | Bass instruments |
| `drum-machine`    | Drum machines (TR-808, etc.) |
| `soundfont`       | General MIDI soundfonts |

**Implementation note:** Apps that don't support a given instrument type should fall back gracefully — play the notes with any available instrument rather than failing.

**Legacy types:** Files may contain `splendid-grand-piano` or `electric-piano` as instrument types. These should be normalized to `keys` on load.

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
  "sustainCells": 2
}
```

| Field          | Type     | Required | Description |
|----------------|----------|----------|-------------|
| `id`           | string   | Yes      | Unique identifier within the layer. |
| `name`         | NoteName | Yes      | Chromatic pitch name. See [Note Names](#note-names). |
| `col`          | number   | Yes      | Horizontal grid position (0-indexed). Represents time. |
| `row`          | number   | Yes      | Vertical grid position (0-indexed). Represents the row on the visual grid. |
| `octave`       | number   | Yes      | Musical octave. Range: `0–8`. Octave 4 = middle C octave. |
| `isRoot`       | boolean  | Yes      | Whether this note is marked as a root note (visual/harmonic hint). |
| `sustainCells` | number   | Yes      | Duration extension in grid cells beyond the note's initial cell. `0` = single-cell note. |

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

## Versioning & Migration

### Current Version: 3

This spec describes version 3. Apps should write version 3 files exclusively.

### Reading Older Versions

Apps should support migrating older formats on load:

**v2 → v3:**
- Wrap all notes and lines into a single default layer
- Set `projectName: ""`, `transposition: 0`
- Use default instrument (`keys` / `Splendid Grand Piano`)

**v1 → v3:**
- v1 notes had `connectedToId` instead of a separate `lines` array
- Convert each note with `connectedToId` + `sustainCells > 0` into a SustainLine
- Then apply v2 → v3 migration

### Future Versions

The `version` field allows forward evolution. Apps encountering an unknown version should:
1. Warn the user that the file was created with a newer format
2. Attempt a best-effort load (ignore unknown fields)
3. Never silently overwrite a file with a version they don't fully understand

---

## Validation Rules

A valid `.dottl` file must satisfy:

1. `version` must be `3` (or a recognized legacy version)
2. `bpm` must be a number in `[20, 300]`
3. `divisor` must be one of `1, 2, 3, 4`
4. `transposition` must be an integer in `[-12, 12]`
5. `difficulty` must be one of `easy`, `intermediate`, `advanced`
6. `layers` must be a non-empty array
7. All `id` fields must be unique within their scope (notes within a layer, lines within a layer, layers within the song)
8. `fromNoteId` in a SustainLine must reference a valid note `id` in the same layer
9. `toNoteId` must reference a valid note `id` in the same layer, or be `null`
10. `name` must be one of the 12 valid NoteName values
11. `octave` must be an integer in `[0, 8]`
12. `volume` and `reverb` must be numbers in `[0, 100]`

---

## Minimal Example

A single C major chord (C4, E4, G4) at 120 BPM:

```json
{
  "version": 3,
  "projectName": "C Major Chord",
  "bpm": 120,
  "divisor": 2,
  "transposition": 0,
  "difficulty": "easy",
  "layers": [
    {
      "id": "layer-1",
      "name": "Piano",
      "color": "",
      "instrumentType": "keys",
      "instrumentName": "Splendid Grand Piano",
      "volume": 70,
      "reverb": 20,
      "notes": [
        { "id": "n1", "name": "C", "col": 0, "row": 0, "octave": 4, "isRoot": true, "sustainCells": 0 },
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
- Normalize legacy instrument types (`splendid-grand-piano` → `keys`, `electric-piano` → `keys`)
- Support loading all schema versions (v1, v2, v3) via migration
- If `instrumentName` is not recognized, use a sensible default for the `instrumentType`

### For Producers (writers)

- Always write `version: 3`
- Always include all required fields — do not omit fields with default values
- Use stable UUIDs for `id` fields (avoid sequential integers that could collide across apps)
- Sanitize `projectName` for use as a filename: lowercase, replace non-alphanumeric with hyphens, strip leading/trailing hyphens, fall back to `"untitled"`

### File Naming Convention

```
{sanitized-project-name}.dottl
```

Example: `"My Cool Song!"` → `my-cool-song.dottl`

---

## Extension Points

The format is intentionally minimal. Future versions or app-specific extensions may add:

- **Sections/markers** — Named regions (intro, verse, chorus) with column ranges
- **Time signature** — Currently implicit (4/4); could be made explicit
- **Key signature** — Currently inferred from root notes
- **Dynamics** — Per-note velocity
- **Metadata** — Author, creation date, tags, license
- **Grid dimensions** — Explicit row-to-pitch mapping for custom tunings

Apps may store additional data in a top-level `extensions` object. Other apps should ignore unrecognized extensions:

```json
{
  "version": 3,
  "extensions": {
    "com.example.myapp": { ... }
  },
  ...
}
```
