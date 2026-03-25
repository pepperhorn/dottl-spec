# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a **specification-only repository** for the `.dottl` song file format used across the Dottl ecosystem (grid editors, play-along games, visualizers, educational tools). There is no source code, build system, or tests — just the format specification in `DOTTL_SONG_FORMAT.md`.

## The Dottl Format

- `.dottl` files are JSON documents (UTF-8) representing musical compositions on a grid
- Current schema version: **5** (apps must always write v5, but support reading v1/v2/v3/v4 via migration)
- Grid model: columns = time (left-to-right, col 0 = beat 1), rows = pitch (chromatic note names)
- Supports multiple instrument layers, sustain/legato lines, chord annotations, and playback metadata (BPM, divisor, transposition)

## Key Spec Details to Remember

- **Timing**: `time_seconds = column / divisor * (60 / bpm)` — divisor is 1–4 (quarter to sixteenth note subdivision). Col 0 = beat 1.
- **Note names**: Exactly 12 valid values: `C, C#/Db, D, D#/Eb, E, F, F#/Gb, G, G#/Ab, A, Bb, B`
- **MIDI conversion**: `midi_pitch = (octave + 1) * 12 + semitone_offset + transposition`
- **Instrument categories**: `Keys`, `Guitars`, `Strings`, `Drums`, `Horns`, `Other`
- **smplr fields**: `smplrLibrary` (optional) specifies the smplr constructor; `smplrPatch` specifies the patch name
- **Note flags**: `isRoot` = harmonic chord root (multiple per layer); `isStartNote` = octave anchor (at most one per layer, optional)
- **Chords**: Top-level `chords` array with `{ col, name, display? }` for chord annotations
- **IDs**: Must be unique within scope; UUIDs recommended over sequential integers
- **Sustain lines**: Connect notes via `fromNoteId`/`toNoteId` (toNoteId can be null for pure horizontal sustain)
