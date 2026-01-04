# Lofi Development Docs - Claude Guide

This is an Obsidian knowledge vault for learning and creating lofi music with Tone.js.

## Purpose

Use this vault as a **reference cookbook** when building lofi tracks. The docs contain tested recipes, patterns, and theory to draw from.

## Key Files to Reference

- `reference/lofi-recipes.md` - Copy-paste ready code snippets
- `reference/tone-js-cheatsheet.md` - Quick API reference
- `07-projects/05-epic-song.md` - Full song architecture example

## Lofi Aesthetic Principles

### The Feel
- **Tempo**: 70-85 BPM (sweet spot: 75-80)
- **Swing**: 0.15-0.25 on 16th notes for groove
- **Humanization**: Timing variance 15-25ms, velocity variance 10-20%
- **Space**: Things should breathe, not fill every gap

### The Sound
- **Warmth**: Lowpass filter (2000-4000 Hz cutoff)
- **Depth**: Reverb (decay 2-4s, wet 0.3-0.5)
- **Texture**: Vinyl noise, tape wobble, bitcrushing
- **Chords**: Jazz voicings - 7ths, 9ths (Dm7, Cmaj7, Am7, G7)

## Tone.js Patterns

### Instruments
```javascript
// Kick - MembraneSynth with fast pitch decay
const kick = new Tone.MembraneSynth({ pitchDecay: 0.05, octaves: 6 })

// Snare/Hats - NoiseSynth with short envelope
const hihat = new Tone.NoiseSynth({ noise: { type: 'pink' }, envelope: { decay: 0.05 } })

// Chords - PolySynth with triangle oscillator
const chords = new Tone.PolySynth(Tone.Synth, { oscillator: { type: 'triangle' } })

// Bass - Synth with sine/triangle
const bass = new Tone.Synth({ oscillator: { type: 'sine' } })
```

### Effects Chain
```javascript
// Standard lofi chain: source -> filter -> reverb -> destination
const filter = new Tone.Filter({ frequency: 3000, type: 'lowpass' });
const reverb = new Tone.Reverb({ decay: 2.5, wet: 0.35 });
await reverb.generate();  // REQUIRED for Tone.Reverb
```

### Humanization Helper
```javascript
function humanize(time, ms = 20) {
  return time + (Math.random() - 0.5) * (ms / 1000);
}
function humanVelocity(base = 0.8, variance = 0.2) {
  return Math.max(0.3, Math.min(1, base + (Math.random() - 0.5) * variance));
}
```

### Transport Setup
```javascript
Tone.Transport.bpm.value = 78;
Tone.Transport.swing = 0.2;
Tone.Transport.swingSubdivision = '16n';
```

## Song Structure Template

```
|  INTRO  |  VERSE A  |  VERSE B  | BREAKDOWN |  CLIMAX  |  OUTRO  |
   8 bars    16 bars     16 bars     8 bars     16 bars    8 bars

 filtered    full        +melody    stripped    peak       fade
 sparse      groove      variation  tension     energy     out
```

### Section Config Pattern
```javascript
const sections = {
  intro:     { drums: false, bass: false, filterFreq: 600,  reverbWet: 0.6 },
  verse:     { drums: true,  bass: true,  filterFreq: 1400, reverbWet: 0.35 },
  climax:    { drums: true,  bass: true,  filterFreq: 2800, reverbWet: 0.45 },
  outro:     { drums: false, bass: false, filterFreq: 500,  reverbWet: 0.6 }
};
```

## Classic Chord Progressions

```javascript
// ii-V-I-vi in C (the lofi standard)
const classic = [
  ['D3', 'F3', 'A3', 'C4'],   // Dm7
  ['G3', 'B3', 'D4', 'F4'],   // G7
  ['C3', 'E3', 'G3', 'B3'],   // Cmaj7
  ['A3', 'C4', 'E4', 'G4']    // Am7
];

// Emotional (vi-IV-I-V in A minor)
const emotional = [
  ['A2', 'C3', 'E3', 'G3'],   // Am7
  ['F2', 'A2', 'C3', 'E3'],   // Fmaj7
  ['C3', 'E3', 'G3', 'B3'],   // Cmaj7
  ['G2', 'B2', 'D3', 'F3']    // G7
];
```

## Volume Staging (dB)

| Element | Range | Notes |
|---------|-------|-------|
| Kick | -4 to -8 | Loudest, anchor the groove |
| Snare | -8 to -12 | Punchy but not dominant |
| Bass | -6 to -10 | Felt more than heard |
| Chords | -8 to -14 | Main body, leave room |
| Hihats | -14 to -20 | Background texture |
| Melody | -10 to -14 | Sits on top but not harsh |
| Vinyl | -22 to -28 | Subtle crackle |

## When Building Songs

1. Start with chord progression and tempo
2. Add drums with swing and humanization
3. Layer bass following root notes
4. Create melody for peak sections
5. Build effects chain (filter, reverb, delay)
6. Structure sections with filter automation
7. Add transitions (filter sweeps, drum fills)
8. Fine-tune levels and add vinyl texture

## File Format for Songs

Single HTML file with embedded JavaScript:
```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://unpkg.com/tone"></script>
</head>
<body>
  <button id="play">Play</button>
  <script>
    // Song code here
    document.getElementById('play').onclick = async () => {
      await Tone.start();
      Tone.Transport.start();
    };
  </script>
</body>
</html>
```

## Remember

- Always `await Tone.start()` on user interaction (browser requirement)
- Always `await reverb.generate()` before using Tone.Reverb
- Pass `time` parameter to triggers inside scheduled callbacks
- Lofi is about imperfection - embrace the wobble

## Ecosystem Integration

Songs built with this guide can be integrated into the dj-overlay controller system.

See `mapping/` for interface specifications:
- `song-interface.md` - How to structure songs as modules
- `visual-interface.md` - How to create audio-reactive visuals
- `dj-protocol.md` - How the controller discovers and crossfades songs

### Quick Reference

To make your song controller-compatible:

1. Export as a class implementing the Song interface
2. Expose `getMasterOutput()` returning a `Tone.Volume` node
3. Implement `play()`, `pause()`, `stop()`, `getState()`
4. Emit events for section changes: `on('sectionChange', cb)`
5. Define sections in a `sections` object with filter/reverb/mute configs
