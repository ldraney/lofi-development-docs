# Lofi Ecosystem Mapping

This directory defines the interfaces and protocols for the distributed lofi system.

## The Ecosystem

```
~/lofi-development-docs/mapping/  <-- You are here (philosophy/standards)
~/dj-overlay/                     <-- Controller
~/lofi-*-song/                    <-- Song repos
~/visual-*/                       <-- Visual repos
```

---

## Open Questions

### For Songs

- What does a song expose? (instruments, patterns, sections, BPM)
- What can be controlled externally? (play, pause, skip section, mute track)
- How does it connect to the audio chain?

### For Visuals

- Are they song-aware (know about sections/BPM) or just audio-reactive?
- Do they render to their own canvas or share one?
- Can multiple visuals layer/stack?

### For the Controller

- How does it wire song audio -> analysers -> visuals?
- How does crossfade work between two songs?

---

## Answers

### Songs

**What does a song expose?**
- `bpm` - Current tempo (read/write)
- `sections` - Object mapping section names to configurations
- `instruments` - Object with named Tone.js synths (kick, snare, hihat, chords, bass, melody)
- `patterns` - Tone.Part instances for each layer
- `currentSection` - Currently active section name
- `currentBar` - Transport position in bars
- `isPlaying` - Boolean playback state

**What can be controlled externally?**
- `play()`, `pause()`, `stop()` - Transport control
- `jumpToSection(name)` - Navigate to section
- `jumpToBar(bar)` - Navigate to bar number
- `muteTrack(name)`, `unmuteTrack(name)` - Per-instrument control
- `setTempo(bpm)` - Change tempo with ramp
- `setFilterFreq(hz)` - Master filter control
- `setReverbWet(0-1)` - Reverb wetness
- Events: `on('sectionChange', cb)`, `on('bar', cb)`

**How does it connect to the audio chain?**
- Song exposes `getMasterOutput()` returning `Tone.Volume` node
- Controller connects song output → crossfader → master analyser → speakers
- Song can also expose `getAnalyser()` for per-song frequency data

### Visuals

**Are they song-aware or just audio-reactive?**
- Both modes supported:
  - Audio-reactive: receives `frequencyData` and `waveformData` arrays
  - Song-aware: receives `{ section, bar, beat, bpm }` metadata

**Do they render to their own canvas or share one?**
- Each visual owns its canvas
- Controller layers canvases with CSS (z-index stacking)
- Visuals export: `init(canvas)`, `render(audioData, songState)`, `dispose()`

**Can multiple visuals layer/stack?**
- Yes, via canvas layering with transparency
- Controller manages the stack order

### Controller

**How does it wire song audio → analysers → visuals?**
```
Song.getMasterOutput()
  → Tone.CrossFade (for A/B deck mixing)
  → Tone.Volume (master)
  → Tone.Destination

Analysers are taps (not pass-through):
  CrossFade → Tone.Analyser (FFT)
  CrossFade → Tone.Analyser (waveform)
  CrossFade → Tone.Meter (volume)
```
- Analyser data passed to visuals each animation frame
- Song events forwarded to visuals for section awareness

**How does crossfade work between two songs?**
- Two song instances (deck A, deck B)
- `Tone.CrossFade` node blends their outputs
- Controller calls `crossfade.fade.rampTo(1, duration)` to transition
- When fade complete, stop deck A, load next song into A

---

## Interface Specifications

- [song-interface.md](./song-interface.md) - Standard for song modules
- [visual-interface.md](./visual-interface.md) - Standard for visual modules
- [dj-protocol.md](./dj-protocol.md) - How controller discovers/loads/crossfades

## Implementations

| Repo | Description | Status |
|------|-------------|--------|
| `~/dj-overlay/` | Controller with dual decks, crossfade, visual stack | Done |
| `~/lofi-demo-song/` | Demo song with drums, chords, 2 sections | Done |
| `~/visual-waveform/` | Oscilloscope-style waveform visual | Done |

## Running the System

```bash
# From parent directory containing all repos
cd ~
npx serve . -p 3000

# Open in browser
# http://localhost:3000/dj-overlay/
```

1. Click **Load Song** to load lofi-demo-song
2. Click **Load Visual** to load waveform visual
3. Click **Play** to start
