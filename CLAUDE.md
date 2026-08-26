# CLAUDE.md

Instructions for Claude Code when working with this repository.

## Project Overview

Braids module for Move Anything - a macro oscillator with 47 synthesis algorithms based on Mutable Instruments Braids.

## Architecture

```
src/
  dsp/
    braids_plugin.cpp   # Main plugin wrapper (V2 API)
    param_helper.h      # Parameter definition helpers (shared)
    braids/             # Braids DSP engine (MIT, Emilie Gillet)
      macro_oscillator  # Entry point - routes to analog/digital
      analog_oscillator # Classic waveforms
      digital_oscillator # FM, physical modeling, noise, etc.
      envelope.h        # AR envelope (unused - replaced by SimpleADSR in plugin)
      svf.h             # State variable filter
      resources         # Lookup tables
    stmlib/             # Mutable Instruments support library
  module.json           # Module metadata
  chain_patches/        # Signal Chain presets
```

## Key Implementation Details

### Plugin API

Implements Move Anything plugin_api_v2 (multi-instance):
- `create_instance`: Initializes 4 voices, each with MacroOscillator + ADSR envelopes + SVF
- `destroy_instance`: Cleanup
- `on_midi`: Note on/off with voice allocation, pitch bend, mod wheel (FM)
- `set_param`: engine, timbre, color, attack, decay, sustain, release, fm, cutoff, resonance, filt_env, f_attack, f_decay, f_sustain, f_release, volume, octave_transpose
- `get_param`: ui_hierarchy, chain_params, state serialization, engine_name
- `render_block`: Renders 24-sample Braids blocks into 128-sample Move blocks

### Parameters

- `engine` (int 0-46): Synthesis algorithm (CSAW, MORPH, FM, PLUK, BELL, etc.)
- `timbre` (float 0-1): Primary tone parameter
- `color` (float 0-1): Secondary tone parameter
- `attack` (float 0-1): Amp envelope attack time
- `decay` (float 0-1): Amp envelope decay time
- `sustain` (float 0-1): Amp envelope sustain level
- `release` (float 0-1): Amp envelope release time
- `fm` (float 0-1): FM amount (also controlled by mod wheel)
- `cutoff` (float 0-1): SVF filter cutoff
- `resonance` (float 0-1): SVF filter resonance
- `filt_env` (float 0-1): Filter envelope modulation amount
- `f_attack` (float 0-1): Filter envelope attack time
- `f_decay` (float 0-1): Filter envelope decay time
- `f_sustain` (float 0-1): Filter envelope sustain level
- `f_release` (float 0-1): Filter envelope release time
- `volume` (float 0-1): Output gain
- `octave_transpose` (int -3 to +3): Octave shift

### Parameter Visualisations (`viz`)

`viz_json_for()` in `braids_plugin.cpp` declares three graphic groups on
`chain_params`, so the knob pages draw real pictures instead of letting the
host's detectors guess (see `schwung/docs/MODULES.md`, "Parameter
visualisations"):

- `amp` — attack/decay/sustain/release, drawn as the amp envelope
- `filter_env` — f_attack/f_decay/f_sustain/f_release, drawn as the filter envelope
- `filter` — cutoff/resonance, drawn as the filter response curve
- `volume` — a single `{"kind":"fader"}`

**The `root` and `filter` knob orders are both load-bearing.** A viz group is
only drawn when its roles sit contiguously on one row (a page is two rows of
four). Reordering either `knobs`/`params` array in `ui_hierarchy` will silently
kill the graphics — the params still work, they just stop being a picture.

- `root` puts `cutoff` at slot 3 to close row 0, so `attack`/`decay`/`sustain`/
  `release` occupy slots 4–7 — all of row 1 — and the amp envelope draws as a
  curve. Splitting those four across the row boundary turns it back into four
  plain dials. (`root` is at the 8-knob cap, so `filt_env` is not on it; it is
  reachable on `filter` at slot 6.)
- `filter` lists `f_attack..f_release` first so those four land in row 0, with
  `cutoff`/`resonance` at slots 4–5 and `filt_env` alone at slot 6.

`root`'s `knobs` array is declared twice — in `src/module.json` and in the
runtime `ui_hierarchy` in `braids_plugin.cpp`. The runtime one is what the
shadow UI reads. **Keep them identical**; they had drifted apart once, and the
runtime copy had dropped `release` entirely (SCH-42).

Deliberately left undeclared, per the migration guide's "if unsure, leave it
undeclared" rule:

- `filt_env` is a modulation *depth*, not a stage of either envelope and not a
  member of the filter pair. There is no honest role for it.
- `engine` is 47 cryptic algorithm abbreviations (CSAW, MORPH, PLUK…). The only
  candidate kind is `waveform`, and no silhouette can truthfully represent a
  macro-oscillator algorithm.

Both fall through to plain knob dials, which is the correct outcome. `timbre`
and `color` likewise match no detector.

Note that `node tools/param-pages/validate.mjs braids` (run from the `schwung`
repo) reads a checked-in fleet capture, not the live device — it will report
these groups as `viz-inferred` until that fixture is re-captured.

### Voice Management

4-voice polyphonic with voice stealing (oldest voice). Each voice has independent MacroOscillator, amplitude ADSR, filter ADSR, and SVF filter with per-sample envelope modulation.

### Sample Rate

Braids lookup tables are calibrated for 96kHz. A pitch correction offset of +1724 (128ths of semitone) compensates for Move's 44.1kHz operation.

## Build

```bash
./scripts/build.sh           # Cross-compile via Docker
./scripts/install.sh         # Deploy to Move
```

## License

MIT (inherited from Mutable Instruments Braids)
