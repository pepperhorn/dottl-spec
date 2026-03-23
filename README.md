# Dottl Song Format Specification

The `.dottl` file format is a JSON-based format for representing musical compositions in the [Dottl](https://dottl.com) ecosystem. It uses a grid-based model where notes are placed at column/row coordinates — time flows left-to-right and pitch is arranged vertically.

## Quick Facts

| | |
|---|---|
| **File extension** | `.dottl` |
| **MIME type** | `application/json` |
| **Encoding** | UTF-8 |
| **Current version** | 3 |

## Format at a Glance

A `.dottl` file contains:

- **Song metadata** — name, BPM (20–300), grid subdivision (1–4), transposition (-12 to +12)
- **Layers** — independent instrument tracks, each with its own instrument, volume, reverb, and color
- **Notes** — placed on a grid with column (time), row (visual position), octave, pitch name, and optional sustain
- **Sustain lines** — connections between notes for legato/sustain phrasing

## Specification

**[Read the full specification →](DOTTL_SONG_FORMAT.md)**

## Examples

The [`examples/`](examples/) directory contains valid `.dottl` files demonstrating various features:

| File | Description |
|------|-------------|
| [`c-major-chord.dottl`](examples/c-major-chord.dottl) | Minimal example — a single C major chord |
| [`twinkle-twinkle.dottl`](examples/twinkle-twinkle.dottl) | Simple melody with sustain, single layer |
| [`multi-layer-beat.dottl`](examples/multi-layer-beat.dottl) | Multiple layers — drums, bass, and keys |
| [`legato-phrases.dottl`](examples/legato-phrases.dottl) | Sustain lines connecting notes for legato phrasing |

## Instrument Types

| Type | Description |
|------|-------------|
| `keys` | Piano/keyboard instruments |
| `mallet` | Vibraphone, marimba, etc. |
| `mellotron` | Mellotron tape-replay instruments |
| `smolken` | Bass instruments |
| `drum-machine` | Drum machines (TR-808, etc.) |
| `soundfont` | General MIDI soundfonts |

## Timing Model

Columns represent subdivisions of a beat, controlled by the `divisor` field:

| Divisor | Columns per beat | Musical value |
|---------|-----------------|---------------|
| 1 | 1 | Quarter notes |
| 2 | 2 | Eighth notes |
| 3 | 3 | Eighth triplets |
| 4 | 4 | Sixteenth notes |

**Time conversion:** `time_seconds = column / divisor * (60 / bpm)`

## Interoperability

**Consumers** should be lenient — ignore unknown fields, normalize legacy instrument types, and support loading v1/v2/v3 files via migration.

**Producers** should always write v3, include all required fields, and use stable UUIDs for IDs.

See the [full spec](DOTTL_SONG_FORMAT.md) for migration paths and validation rules.

## License

This specification is released under the [MIT License](LICENSE).
