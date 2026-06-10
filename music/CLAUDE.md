# Music apps

## Mission

Build a growing collection of locally-run, single-file HTML music apps that listen to live drum performance — rhythm, dynamics, and timbre — alongside MIDI controller input, and algorithmically synthesize accompaniment. Each app is its own piece: a different song, a different angle on synthesis, a different way of letting drumming drive sound.

The user plays drums and is good at it, but has limited time to play with other musicians. These apps are the band: an evolving set of experiments in letting the drummer's own playing generate the rest of the music.

## Design principles

- **One file per app.** Each app is a single self-contained `.html` file with inline CSS/JS — same pattern as the timers and stretch coach in the parent directory. No build step, no bundler, no node_modules. Open the file in a browser and play.
- **Local only.** No servers, no network calls, no external APIs at runtime. Everything runs in the browser on the user's machine.
- **Live performance first.** Latency matters. Prefer `AudioWorklet` / `ScriptProcessor` only when needed; use the Web Audio graph (oscillators, filters, gain, delay, convolver) wherever possible. Avoid blocking the audio thread.
- **Drumming drives everything.** The drum input is the conductor — tempo, dynamics, density, and timbral character of the playing should be what shapes the generated accompaniment. Don't reduce the drummer to a metronome click.
- **Each app explores one idea.** Don't build a swiss-army synth. A new song / new sonic concept = a new file. Re-use is fine via copy-paste, not via a shared library.

## Inputs

Apps draw from some combination of:

- **MIDI drum input** (Web MIDI API): for electronic kits. Parse note-on velocity for dynamics, intervals between hits for rhythm.
- **Acoustic drum audio input** (`getUserMedia` → `MediaStreamAudioSourceNode`): for mic'd acoustic drums via built-in mic OR an external audio interface (Scarlett, etc.). FFT spectrum gives you dB-per-frequency in real time — the basis for onset detection, band energy ("kick / snare / cymbal" regions), and rough pitch estimation. See **Audio analysis** below for the full toolkit.
- **MIDI controller input** (Web MIDI API): knobs, pads, keys — for live control of generative parameters while playing.

Each app's header comment should state which inputs it uses.

## Synthesis & generation

- Lean on Web Audio's native nodes: `OscillatorNode`, `BiquadFilterNode`, `DelayNode`, `ConvolverNode`, `WaveShaperNode`, `DynamicsCompressorNode`, custom `PeriodicWave`.
- Algorithmic accompaniment ideas to explore across apps: Markov chains over pitch/rhythm, cellular automata, Euclidean rhythms, granular synthesis driven by drum onsets, FM/AM synthesis modulated by playing dynamics, generative harmony from a key/mode locked by a MIDI controller, drone layers that respond to playing density.
- Treat synthesis as a first-class theme — synthesizers and the elements of music (rhythm, harmony, timbre, dynamics, form) are the subject matter, not just the implementation.

## Music theory

Use this section as the single source of truth for scales, frequencies, and pitch arithmetic. Don't re-derive these in each app — copy the JS helpers below.

### Modes & scales (semitone offsets from tonic)

The diatonic modes are 7-note scales built from a sequence of whole steps (W = 2 semitones) and half steps (H = 1 semitone). The semitone column is what apps use directly: add each offset to the tonic's MIDI number to get the notes in the scale. Append `+12` for the octave when iterating up.

| Scale            | Steps (W=2, H=1) | Semitones from tonic       | Notes |
|------------------|------------------|----------------------------|-------|
| Ionian (major)   | W W H W W W H    | `[0, 2, 4, 5, 7, 9, 11]`   | the major scale |
| Dorian           | W H W W W H W    | `[0, 2, 3, 5, 7, 9, 10]`   | minor with raised 6 |
| Phrygian         | H W W W H W W    | `[0, 1, 3, 5, 7, 8, 10]`   | minor with flat 2 (Spanish flavor) |
| Lydian           | W W W H W W H    | `[0, 2, 4, 6, 7, 9, 11]`   | major with sharp 4 (dreamy) |
| Mixolydian       | W W H W W H W    | `[0, 2, 4, 5, 7, 9, 10]`   | major with flat 7 (rock/blues) |
| Aeolian (minor)  | W H W W H W W    | `[0, 2, 3, 5, 7, 8, 10]`   | natural minor |
| Locrian          | H W W H W W W    | `[0, 1, 3, 5, 6, 8, 10]`   | diminished, rarely a tonic |
| Pentatonic major | —                | `[0, 2, 4, 7, 9]`          | 5-note major (folk/pop) |
| Pentatonic minor | —                | `[0, 3, 5, 7, 10]`         | 5-note minor (blues/rock) |
| Blues            | —                | `[0, 3, 5, 6, 7, 10]`      | minor pentatonic + the "blue note" (♭5) |

### Frequency reference (12-TET, A4 = 440 Hz)

Formula: `freq = 440 * 2^((midi - 69) / 12)`

MIDI numbering: `midi = (octave + 1) * 12 + pitchClassIndex`, where `C=0, C#=1, D=2, …, B=11`. So `C4 = 60`, `A4 = 69` (concert pitch), and middle C is `C4`.

Frequencies (Hz, rounded to 2 decimals):

| Octave | MIDI    | C       | C#      | D       | D#      | E       | F       | F#      | G       | G#      | A       | A#      | B       |
|--------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|
| -1     | 0–11    | 8.18    | 8.66    | 9.18    | 9.72    | 10.30   | 10.91   | 11.56   | 12.25   | 12.98   | 13.75   | 14.57   | 15.43   |
| 0      | 12–23   | 16.35   | 17.32   | 18.35   | 19.45   | 20.60   | 21.83   | 23.12   | 24.50   | 25.96   | 27.50   | 29.14   | 30.87   |
| 1      | 24–35   | 32.70   | 34.65   | 36.71   | 38.89   | 41.20   | 43.65   | 46.25   | 49.00   | 51.91   | 55.00   | 58.27   | 61.74   |
| 2      | 36–47   | 65.41   | 69.30   | 73.42   | 77.78   | 82.41   | 87.31   | 92.50   | 98.00   | 103.83  | 110.00  | 116.54  | 123.47  |
| 3      | 48–59   | 130.81  | 138.59  | 146.83  | 155.56  | 164.81  | 174.61  | 185.00  | 196.00  | 207.65  | 220.00  | 233.08  | 246.94  |
| 4      | 60–71   | 261.63  | 277.18  | 293.66  | 311.13  | 329.63  | 349.23  | 369.99  | 392.00  | 415.30  | 440.00  | 466.16  | 493.88  |
| 5      | 72–83   | 523.25  | 554.37  | 587.33  | 622.25  | 659.26  | 698.46  | 739.99  | 783.99  | 830.61  | 880.00  | 932.33  | 987.77  |
| 6      | 84–95   | 1046.50 | 1108.73 | 1174.66 | 1244.51 | 1318.51 | 1396.91 | 1479.98 | 1567.98 | 1661.22 | 1760.00 | 1864.66 | 1975.53 |
| 7      | 96–107  | 2093.00 | 2217.46 | 2349.32 | 2489.02 | 2637.02 | 2793.83 | 2959.96 | 3135.96 | 3322.44 | 3520.00 | 3729.31 | 3951.07 |
| 8      | 108–119 | 4186.01 | 4434.92 | 4698.64 | 4978.03 | 5274.04 | 5587.65 | 5919.91 | 6271.93 | 6644.88 | 7040.00 | 7458.62 | 7902.13 |
| 9      | 120–127 | 8372.02 | 8869.84 | 9397.27 | 9956.06 | 10548.08| 11175.30| 11839.82| 12543.85| —       | —       | —       | —       |

Anchor points to spot-check: **A4 = 440.00 Hz exactly**, **C4 = 261.63 Hz (middle C)**, **A0 = 27.50 Hz (lowest piano A)**, **C8 = 4186.01 Hz (highest piano C)**.

The user's gear in pitch terms:
- **Akai pads** ch1 notes 36–43 → C2–G2 → 65.41–98.00 Hz (sub-bass; usually used as *triggers* for pitched accompaniment, not as pitches themselves)
- **Alesis keys** ch1 notes 36–60 → C2–C4 → 65.41–261.63 Hz (full 2-octave keyboard, C2 lowest to C4 middle C)

### JS helpers (copy into apps)

```js
// 7 diatonic modes + 3 common scales — semitone offsets from the tonic.
const SCALES = {
  ionian:          [0, 2, 4, 5, 7, 9, 11],   // major scale
  dorian:          [0, 2, 3, 5, 7, 9, 10],
  phrygian:        [0, 1, 3, 5, 7, 8, 10],
  lydian:          [0, 2, 4, 6, 7, 9, 11],
  mixolydian:      [0, 2, 4, 5, 7, 9, 10],
  aeolian:         [0, 2, 3, 5, 7, 8, 10],   // natural minor
  locrian:         [0, 1, 3, 5, 6, 8, 10],
  pentatonicMajor: [0, 2, 4, 7, 9],
  pentatonicMinor: [0, 3, 5, 7, 10],
  blues:           [0, 3, 5, 6, 7, 10]
};

// MIDI → Hz. 12-TET, A4 = 440.
const midiToFreq = midi => 440 * Math.pow(2, (midi - 69) / 12);

// Note-name ↔ MIDI conversion. Accepts "C4", "F#3", "Bb-1", etc.
const NOTE_TO_PC = { C:0, 'C#':1, Db:1, D:2, 'D#':3, Eb:3, E:4, F:5, 'F#':6, Gb:6, G:7, 'G#':8, Ab:8, A:9, 'A#':10, Bb:10, B:11 };
const PC_TO_NAME = ['C','C#','D','D#','E','F','F#','G','G#','A','A#','B'];
function noteNameToMidi(name) {
  const m = String(name).match(/^([A-G][b#]?)(-?\d+)$/);
  if (!m) return null;
  const pc = NOTE_TO_PC[m[1]];
  return pc == null ? null : (parseInt(m[2], 10) + 1) * 12 + pc;
}
const midiToNoteName = midi => PC_TO_NAME[((midi % 12) + 12) % 12] + (Math.floor(midi / 12) - 1);

// Step within a scale. `degree` is 0-indexed from the tonic; can be negative or > scale length
// (octave shifts are handled automatically: degree 7 in Ionian = octave above the tonic).
function noteInScale(tonicMidi, scale, degree) {
  const intervals = SCALES[scale];
  if (!intervals) throw new Error('Unknown scale: ' + scale);
  const len = intervals.length;
  const oct = Math.floor(degree / len);
  const idx = ((degree % len) + len) % len;
  return tonicMidi + intervals[idx] + 12 * oct;
}
const freqInScale = (tonicMidi, scale, degree) => midiToFreq(noteInScale(tonicMidi, scale, degree));

// Reverse: given a note (MIDI) that's in the scale, what's its scale-degree index?
// Returns null if the note isn't in the scale.
function degreeOfNote(midi, tonicMidi, scale) {
  const intervals = SCALES[scale];
  if (!intervals) return null;
  const offset = midi - tonicMidi;
  const octaves = Math.floor(offset / 12);
  const semi = ((offset % 12) + 12) % 12;
  const idx = intervals.indexOf(semi);
  return idx < 0 ? null : idx + octaves * intervals.length;
}

// Move N scale degrees from a current note. If the current note isn't in the scale,
// falls back to chromatic movement.
function step(midi, tonicMidi, scale, deltaDegrees) {
  const d = degreeOfNote(midi, tonicMidi, scale);
  return d == null ? midi + deltaDegrees : noteInScale(tonicMidi, scale, d + deltaDegrees);
}

// Example — "D Dorian, tonic D3, give me the 4th degree":
//   noteInScale(noteNameToMidi('D3'), 'dorian', 3)   →  55   (= G3)
//   freqInScale(noteNameToMidi('D3'), 'dorian', 3)   →  195.998 Hz
//   midiToNoteName(55)                                →  'G3'
```

## Audio analysis

For apps where the input isn't (only) MIDI — drums mic'd through the built-in mac mic or an external audio interface like a Scarlett, line input from a guitar/synth, whatever — analysis happens via the Web Audio API's `AnalyserNode`. The output is a real-time spectrum: **dB level per frequency bin**, updated every audio frame. From that one signal you can derive everything generative logic typically needs.

Derived signals (all available from a single AnalyserNode):

- **Band energy** — dB averaged across a frequency range. Kick = 50–120 Hz, snare = 200–500 Hz, cymbals = 5–10 kHz, etc.
- **Note energy** — dB at the fundamental frequency of a specific MIDI note (uses `midiToFreq` from above).
- **Peak frequency** — the loudest bin's center frequency. A rough pitch estimate for monophonic, clean input.
- **Onset detection** — energy spikes vs. running average. Triggers when a drum hits or a note is struck.

### Choosing the input source

`navigator.mediaDevices.getUserMedia({ audio: ... })` returns a `MediaStream`. The browser picks the default audio input device unless you constrain to a specific `deviceId`. For the user's Scarlett, after permission is granted, `enumerateDevices()` lists it alongside the built-in mic — the app can show a picker.

```js
// Must call getUserMedia at least once with permission granted before labels are readable.
const devices = await navigator.mediaDevices.enumerateDevices();
const inputs = devices.filter(d => d.kind === 'audioinput');
// Each: { deviceId, label, kind, groupId }. label is e.g. "Scarlett 2i2 USB" once permission granted.
```

### FFT trade-offs

FFT size = the time/frequency resolution trade-off. Pick per app:

| FFT size | Freq resolution (48 kHz) | Time window | Good for                                 |
|----------|---------------------------|-------------|------------------------------------------|
| 256      | 187.5 Hz / bin            | ~5.3 ms     | onset / transient detection (low latency)|
| 512      | 93.8 Hz / bin             | ~10.7 ms    | fast trigger detection                   |
| 1024     | 46.9 Hz / bin             | ~21.3 ms    | balanced — sane default                  |
| 2048     | 23.4 Hz / bin             | ~42.7 ms    | pitch detection in the mid range         |
| 4096     | 11.7 Hz / bin             | ~85.3 ms    | pitch detection in the bass              |
| 8192     | 5.9 Hz / bin              | ~170.7 ms   | precise pitch, sluggish                  |

For apps that need both — onset detection AND pitch — run **two `AnalyserNode`s in parallel** off the same source: one with `fftSize=256` for transient timing, one with `fftSize=4096` for pitch content. The source connects to both.

`smoothingTimeConstant` (0–1) controls temporal smoothing of the dB output: `~0.8` looks smooth on a spectrogram visualization, `~0.2` is responsive enough to drive logic, `0` is raw frame-by-frame.

### dB scale

`getFloatFrequencyData(buf)` fills `buf` (length = `fftSize / 2`) with **dBFS** values — dB relative to digital full-scale. `0 dBFS` is the loudest digital signal, negative values are quieter. Music typically sits around `-30` to `-60 dBFS`. Silence reads as `-Infinity` (clamped to `minDecibels`).

`AnalyserNode.minDecibels` (default `-100`) and `maxDecibels` (default `-30`) clip the reported range. For broader sensitivity set `minDecibels = -120`, `maxDecibels = 0`.

### Mapping bins ↔ frequency

```
freq(binIdx)   = binIdx * sampleRate / fftSize
binIdx(freq)   = round(freq * fftSize / sampleRate)
```

So the dB at a specific MIDI note's fundamental is `spectrum[round(midiToFreq(note) * fftSize / sampleRate)]`. Combine that with the music-theory helpers and you can answer "how loud is C3 in the input right now?" with one line — see `noteEnergy` below.

### JS helpers (copy into apps)

```js
// Open an audio input, return { ctx, analyser, source, stream }.
// Keep all four alive for the app's lifetime — drop them and the analysis stops.
async function openAudioInput(opts = {}) {
  const ctx = new (window.AudioContext || window.webkitAudioContext)();
  // Defeat browser voice-chat DSP — it ruins music analysis.
  const audioConstraints = { echoCancellation: false, noiseSuppression: false, autoGainControl: false };
  if (opts.deviceId) audioConstraints.deviceId = { exact: opts.deviceId };
  const stream = await navigator.mediaDevices.getUserMedia({ audio: audioConstraints });
  const source = ctx.createMediaStreamSource(stream);
  const analyser = ctx.createAnalyser();
  analyser.fftSize = opts.fftSize ?? 2048;
  analyser.smoothingTimeConstant = opts.smoothing ?? 0.2;
  analyser.minDecibels = opts.minDecibels ?? -120;
  analyser.maxDecibels = opts.maxDecibels ?? 0;
  source.connect(analyser);
  // IMPORTANT: do NOT connect analyser → ctx.destination for mic input — you'll feedback.
  // The analyser receives data whether or not it's connected downstream.
  return { ctx, analyser, source, stream };
}

// List audio input devices. Labels are empty strings until getUserMedia permission is granted.
async function listAudioInputs() {
  const devices = await navigator.mediaDevices.enumerateDevices();
  return devices.filter(d => d.kind === 'audioinput');
}

// Current spectrum as Float32Array of dB values. Length = analyser.frequencyBinCount.
// Reuse the `buf` argument across calls to avoid allocation churn.
function getSpectrum(analyser, buf) {
  buf = buf || new Float32Array(analyser.frequencyBinCount);
  analyser.getFloatFrequencyData(buf);
  return buf;
}

// Bin ↔ frequency conversions. Pass the AudioContext + AnalyserNode.
const binToFreq = (binIdx, ctx, analyser) => binIdx * ctx.sampleRate / analyser.fftSize;
const freqToBin = (freq, ctx, analyser) => Math.round(freq * analyser.fftSize / ctx.sampleRate);

// Average dB across a frequency band. (Mean in dB-space is fine for control purposes;
// if you need true power averaging, convert to linear with 10^(db/20), average, convert back.)
function bandEnergy(spectrum, lowHz, highHz, ctx, analyser) {
  const lo = Math.max(0, freqToBin(lowHz, ctx, analyser));
  const hi = Math.min(spectrum.length - 1, freqToBin(highHz, ctx, analyser));
  let sum = 0, count = 0;
  for (let i = lo; i <= hi; i++) {
    if (spectrum[i] > -Infinity) { sum += spectrum[i]; count++; }
  }
  return count ? sum / count : -Infinity;
}

// dB at the fundamental of a specific MIDI note. Pairs with the music-theory section.
function noteEnergy(spectrum, midi, ctx, analyser) {
  const b = freqToBin(midiToFreq(midi), ctx, analyser);
  return (b >= 0 && b < spectrum.length) ? spectrum[b] : -Infinity;
}

// Loudest bin's center frequency + its dB. Skips DC (idx 0). Rough pitch estimate
// only — for accurate pitch on noisy / polyphonic signals, use autocorrelation (out of scope here).
function peakFrequency(spectrum, ctx, analyser) {
  let maxIdx = 1, maxVal = spectrum[1];
  for (let i = 2; i < spectrum.length; i++) {
    if (spectrum[i] > maxVal) { maxVal = spectrum[i]; maxIdx = i; }
  }
  return { freq: binToFreq(maxIdx, ctx, analyser), db: maxVal };
}

// Energy-based onset detector. Call every frame. Persist `state` across calls:
//   const onset = { ema: -120 };
//   if (detectOnset(spectrum, onset)) { /* trigger */ }
// Tune `threshold` (dB above running average) and `alpha` (EMA factor) per source.
// Defaults work for typical drum mics.
function detectOnset(spectrum, state, opts = {}) {
  const threshold = opts.threshold ?? 6;
  const alpha     = opts.alpha     ?? 0.1;
  let sum = 0, n = 0;
  for (let i = 1; i < spectrum.length; i++) {
    if (spectrum[i] > -Infinity) { sum += spectrum[i]; n++; }
  }
  const energy = n ? sum / n : -120;
  const isOnset = (energy - state.ema) > threshold;
  state.ema = state.ema * (1 - alpha) + energy * alpha;
  return isOnset;
}

// Example wiring — all-in-one drum analyzer pattern:
//   const { ctx, analyser } = await openAudioInput({ fftSize: 1024 });
//   await ctx.resume(); // requires user gesture
//   const spectrum = new Float32Array(analyser.frequencyBinCount);
//   const onset = { ema: -120 };
//   function tick() {
//     getSpectrum(analyser, spectrum);
//     const kick   = bandEnergy(spectrum, 50, 120,    ctx, analyser);
//     const snare  = bandEnergy(spectrum, 200, 500,   ctx, analyser);
//     const cymbal = bandEnergy(spectrum, 5000, 12000, ctx, analyser);
//     if (detectOnset(spectrum, onset)) { /* a drum was hit */ }
//     requestAnimationFrame(tick);
//   }
//   tick();
```

### Gotchas

- **User gesture required.** `AudioContext` starts in `suspended` state on most browsers. Call `ctx.resume()` from a click handler. The Start button pattern from `midi-monitor.html` works here too.
- **Don't connect the analyser to `ctx.destination`** when the input is a microphone — the speakers will pick up the monitored audio and create a feedback loop. The analyser still receives data without being connected downstream.
- **Browser DSP off.** `echoCancellation`, `noiseSuppression`, and `autoGainControl` are great for voice chat and ruinous for music analysis. The helpers above set them to `false` — keep it that way unless you have a reason.
- **Sample rate** is typically 48000 Hz on macOS Chrome. Always read `ctx.sampleRate` rather than hardcoding.
- **Device labels empty until permission granted.** First call `getUserMedia` with default audio (to trigger the permission prompt), then call `enumerateDevices()` for a labeled list — letting the user re-select via constraints.
- **Latency** ≈ `fftSize / sampleRate` (the FFT window) + ~10–30 ms (browser audio buffer). For drum-trigger response, keep `fftSize` ≤ 512.
- **Stop properly.** When done, `stream.getTracks().forEach(t => t.stop())` to release the mic, then `ctx.close()`. Otherwise the mic indicator stays on.

## App conventions

- Filename: `<song-or-concept-name>.html`, kebab-case.
- Top of file: an HTML comment block with — title, one-line concept, required inputs (MIDI device? audio input?), and how to play it.
- Fully standalone: full `<!doctype html>` document with self-defined colors. The timer apps in the parent dir are Claude artifact fragments that rely on host CSS variables — music apps must not depend on those, since they run from `file://` locally.
- UI: minimal. A start button (Web Audio + Web MIDI both require a user gesture), input device pickers, and whatever live controls the app needs. Match the visual restraint of the existing timer apps (DM Mono + Playfair Display, off-white background `#faf8f3`, near-black text `#2a2724`, accent green `#1d9e75`, hairline `#ece7e0` borders).
- No autoplay. Wait for the user to click start.

## Shared data — device mapping profiles

Device-profile JSON files live next to the apps in `music/`. Each is one device, created by `midi-monitor.html` via **export**, loadable by any app via **import**. Read the files directly when building an app — they are the source of truth.

### The user's controllers (canonical mappings, as of 2026-05-25)

Always reference these via the JSON files, not by hardcoding values — re-read them at the start of each session in case they've been updated.

- **`music/Akai.json`** — Akai pad/knob controller. 8 pads (channel 1, notes 36–43 → pad1–pad8, sequential — matches the physical labels on the device) + 8 knobs (channel 1, CC 1–8 → knob1–knob8). Good for: triggering / dynamics / 8 continuous control parameters.
- **`music/Alesis.json`** — Alesis 25-key MIDI keyboard. 25 keys (channel 1, notes 36–60 = C2–C4) + 4 knobs (channel 1, CC 14–17 → knob1–knob4) + pitch bend (`pb-ch1` → "pb") + mod wheel (CC 1 → "mod") + sustain pedal (CC 64 → "sus"). Good for: pitched accompaniment, expressive controllers.

### Profile format

```json
{
  "deviceName": "Akai",
  "labels": {
    "note-36-ch1": "pad5",
    "cc-1-ch1":    "knob1",
    "pb-ch1":      "pb"
  },
  "exportedAt": "2026-05-25T18:05:55.318Z",
  "version": 1
}
```

### Label key format (deterministic — any app can look up by key)

- `note-<num>-ch<channel>` — note on/off (pad, keyboard key)
- `cc-<num>-ch<channel>` — control change (knob, slider, pedal)
- `pb-ch<channel>` — pitch bend
- `at-ch<channel>` — channel aftertouch
- `pat-<num>-ch<channel>` — poly aftertouch
- `pc-ch<channel>` — program change

### Conventions

- Filename: `<DeviceName>.json` at the top of `music/`. (The original `music/mappings/` subdirectory convention was relaxed once profiles started living alongside apps.)
- Apps that want friendly names should accept a mapping at startup — either via a file picker (import), or by reading a known profile by name (e.g. `fetch('Akai.json')` if served over http; otherwise prompt the user to pick the file).
- Fallback when no profile is loaded: show raw `cc74`/`note36`-style names.
- `midi-monitor.html` also persists labels in `localStorage` under `midi-monitor:labels:<deviceName>`, so labels survive reloads even before export.

## What to ask before building a new app

When the user asks to build a new music app, confirm:

1. The song or concept — what's the musical idea?
2. Drum input mode — MIDI e-kit, mic'd acoustic, or both?
3. What MIDI controllers (if any) and how they map to parameters.
4. The intended role of the accompaniment — bass? pads? counter-rhythm? full arrangement?

Don't guess these. They define the app.

## Apps

### grid-conductor.html — status: built, awaiting in-person test

The drummer's mic'd hits steer a polyphonic step sequencer via a 2D **(freq, dB) action grid**. Each hit's peak-bin frequency × onset energy (over EMA baseline) lands inside one rectangle on the grid; that rectangle's "action" mutates sequencer state. **Actions never sound directly** — sound emerges from the sequencer loop, which on each step reads live per-voice state (degree, octave, additive harmonic stack, filter, envelope, FX, master gain) and triggers a synth note. Scale is global; everything else is selected-voice or per-voice.

#### Inputs

- **Acoustic drum audio** (any input device, picked at start).
- **Alesis 25-key MIDI** (channel 1):
  - Notes **36 / 38 / 42 / 46** → voice-select pads (stack order: voice 1 / 2 / 3 / 4). Pressing only valid if that stack position has a voice.
  - Notes **37, 39, 40, 41, 43, 44, 45, 47** → unused.
  - Notes **48–60** (C3–C4) → tonic + song start. First press starts the loop; subsequent presses only update the tonic.
  - CCs **14 / 15 / 16 / 17** → voice 1 / 2 / 3 / 4 master volume (linear 0..1). Buffered for not-yet-added voices.
  - CC **64** → tap-tempo (each value=127 is a tap). BPM = 60 / mean(last 4 inter-tap intervals).
- Mod wheel / pitch bend / Akai / Alesis knobs not listed above are unused in v1.

#### Pad LEDs

On voice-selection change, send Note Off at the old pad note and Note On at the new pad note (velocity 127) to the Alesis **output** port. Auto-pick Alesis on both input and output by name; dropdowns let the user override.

#### Voice stack (LIFO)

Voices are added and removed **only at the top of the stack**. Pad note for voice `i` is fixed: `[36, 38, 42, 46][i-1]`. Knob `i` always controls voice `i`'s gain.

#### Synthesis (per voice)

Additive harmonic stack, up to 5 layers. Layer `i` plays at `fund × (i+1)` Hz with amplitude `0.5^i` (so −6 dB per layer). Waveform-append action pushes one of: `sine`, `triangle`, `sawtooth`, `square`, `pulse` (custom PeriodicWave, duty 0.25). Clear-stack resets to the first layer. Each voice has its own lowpass biquad (cutoff, Q), AHR envelope (A, D, R), and delay/reverb sends into shared FX. Shared delay is dotted-eighth at current BPM with ~0.35 feedback; reverb is a 1.4 s exponential-noise ConvolverNode IR.

#### Action scopes

- **Selected voice only**: set-degree (7), set-octave (9), push-wave (5), clear-stack (1), filter cutoff (6), filter Q (6), envelope A / D / R (6 each), delay-send (6), reverb-send (6).
- **Global**: mode-swap (10 — applies the new scale to *all* voices), structural (add voice / remove voice / add measure / remove measure).
- **Per-voice tagged**: step toggles (each voice has its own pattern; band 5 has one sub-column per voice).

#### Action grid bands (`x` = log freq, `y` = energy-over-EMA, both normalized 0..1)

| Band Hz       | Drum-natural       | Contents                                                                                   |
|---------------|--------------------|--------------------------------------------------------------------------------------------|
| 30 – 120      | kick / floor tom   | Structural 2×2: add voice, remove voice, add measure, remove measure                       |
| 120 – 400     | snare body         | Three full-width strips: **mode** (10, top), **octave** (9, middle), **degree** (5–7, bottom — sized to current scale) |
| 400 – 1500    | snare top / tom    | Seven full-width strips of 6 cells each (top→bottom): reverb-send, delay-send, R, D, A, Q, cutoff |
| 1500 – 5000   | hi-hat / splash    | Top: 5 waveform-append cells. Bottom: 1 clear-stack cell                                   |
| 5000 – 15000  | ride / cymbals     | 4 voice sub-columns × (measures rows × 16 step cols) — per-voice toggles                   |

Initial action count (1 voice, 1 measure): ~94. Max (4 voices, 8 measures): ~590. `recomputeGrid()` rebuilds the zone list on every state change; lookup is a linear scan, trivial.

#### Sequencer

Chris-Wilson lookahead scheduler. Step = 16th note (`60 / bpm / 4` s). Sequencer is **gated on `state.started`** — first tonic-range key press flips `started = true` and seeds `nextStepTime`. Default 120 BPM if no taps yet. Pattern length is `measures × 16` per voice; adding a measure extends with `false`, removing truncates from the end.

#### Audio analysis

Two `AnalyserNode`s on the same source. `fast` (fftSize 512, smoothing 0) for onset detection; `slow` (fftSize 2048, smoothing 0.2) for peak-bin frequency. EMA threshold = 6 dB; EMA only updates between onsets so the baseline tracks noise floor, not the hit itself. `y` is `clamp(energy − ema, 0, 40) / 40`. Do **not** connect any analyser to `ctx.destination` (mic feedback). `getUserMedia` runs with `echoCancellation/noiseSuppression/autoGainControl` all `false`.

#### `verify` command

When the user types **`verify`**, `verify grid-conductor`, or anything close (e.g. "let's verify", "run the verification"), output the steps below verbatim:

```
Verification steps for grid-conductor.html — follow in order.

1. Open music/grid-conductor.html in Chrome (Web MIDI requires Chromium-family).
2. Click Start. Pick audio input (Scarlett if connected, else built-in). Confirm Alesis auto-selects for both MIDI in and out.
3. Tap CC64 four times at desired tempo → BPM display updates (sequencer still paused).
4. Press a key in C3–C4 → tonic display updates, sequencer starts, voice 1 plays 8th notes on that tonic. Pad at note 36 lights up.
5. Press note 38 → second voice's pad lights up. But there's only one voice, so the press is ignored (sanity check).
6. Hit drum in band 1 (low freq, kick zone) → 'add voice' action fires; new voice 2 appears. Press 38 → it now selects voice 2 (LED moves).
7. Hit drum in band 2 (snare body, mid energy) → degree / octave / mode zones fire on the selected voice (degree, octave) or globally (mode).
8. Hit cymbals (high freq) in voice 1's column of band 5 → step toggles flip on/off in voice 1's pattern.
9. Wiggle knob CC14 → voice 1's volume bar moves and you hear it. Wiggle CC15 → voice 2.
10. Hit 'add measure' zone → toggle band 5 gets a second row of cells for each voice.
11. Stop, change input device, restart — confirm device choices persist across reloads.

Report back what passes / fails per step — issues with onset detection or zone hit rate are expected first-pass and we'll iterate.
```

Don't reformat or summarize them — show all 11 steps verbatim each time so they can be ticked off one-for-one.
