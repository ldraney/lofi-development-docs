# DJ Protocol

How the dj-overlay controller discovers, loads, and crossfades between songs and visuals.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         dj-overlay                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐     ┌──────────┐                                 │
│  │  Deck A  │     │  Deck B  │     Song Instances               │
│  │  (Song)  │     │  (Song)  │                                  │
│  └────┬─────┘     └────┬─────┘                                 │
│       │                │                                        │
│       └───────┬────────┘                                       │
│               ▼                                                 │
│       ┌───────────────┐                                        │
│       │  CrossFade    │     Tone.CrossFade node                │
│       └───────┬───────┘                                        │
│               ▼                                                 │
│       ┌───────────────┐                                        │
│       │   Analyser    │     FFT + Waveform                     │
│       └───────┬───────┘                                        │
│               ▼                                                 │
│       ┌───────────────┐                                        │
│       │    Master     │     Volume control                     │
│       └───────┬───────┘                                        │
│               ▼                                                 │
│       ┌───────────────┐                                        │
│       │  Destination  │     Speakers                           │
│       └───────────────┘                                        │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    Visual Stack                            ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                    ││
│  │  │ Visual  │  │ Visual  │  │ Visual  │   Canvas layers    ││
│  │  │   BG    │  │   Mid   │  │   FG    │                    ││
│  │  └─────────┘  └─────────┘  └─────────┘                    ││
│  └────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Discovery

### Local Mode

Controller scans configured directories for song/visual repos:

```javascript
const config = {
  songPaths: [
    '../lofi-*-song',
    '~/music/lofi-songs/*'
  ],
  visualPaths: [
    '../visual-*',
    '~/visuals/*'
  ]
};

async function discoverSongs() {
  const songs = [];

  for (const pattern of config.songPaths) {
    const dirs = await glob(pattern);
    for (const dir of dirs) {
      const manifestPath = path.join(dir, 'manifest.json');
      if (await exists(manifestPath)) {
        const manifest = await readJSON(manifestPath);
        songs.push({
          ...manifest,
          path: dir,
          entryPath: path.join(dir, manifest.entry)
        });
      }
    }
  }

  return songs;
}
```

### Registry Mode (Future)

Optional central registry for remote song/visual loading:

```javascript
const registry = {
  songs: [
    { name: 'chill-vibes', url: 'https://cdn.example.com/songs/chill-vibes.js' },
    { name: 'rainy-day', url: 'https://cdn.example.com/songs/rainy-day.js' }
  ],
  visuals: [
    { name: 'waveform', url: 'https://cdn.example.com/visuals/waveform.js' }
  ]
};
```

## Loading Sequence

```javascript
class DJController {
  constructor() {
    this.deckA = null;
    this.deckB = null;
    this.activeDeck = 'A';
    this.crossfade = null;
    this.analyser = null;
    this.visuals = [];
  }

  async init() {
    // Audio routing
    this.crossfade = new Tone.CrossFade();
    this.analyser = new Tone.Analyser('fft', 1024);
    this.waveformAnalyser = new Tone.Analyser('waveform', 1024);
    this.meter = new Tone.Meter();
    this.master = new Tone.Volume(0);

    this.crossfade.connect(this.analyser);
    this.analyser.connect(this.waveformAnalyser);
    this.waveformAnalyser.connect(this.meter);
    this.meter.connect(this.master);
    this.master.toDestination();

    // Discover available content
    this.availableSongs = await discoverSongs();
    this.availableVisuals = await discoverVisuals();
  }

  async loadSong(songManifest, deck = 'A') {
    // Dynamic import of song module
    const SongModule = await import(songManifest.entryPath);
    const song = new SongModule.default();
    await song.init();

    // Connect to crossfade
    if (deck === 'A') {
      if (this.deckA) this.deckA.dispose();
      this.deckA = song;
      song.getMasterOutput().connect(this.crossfade.a);
    } else {
      if (this.deckB) this.deckB.dispose();
      this.deckB = song;
      song.getMasterOutput().connect(this.crossfade.b);
    }

    // Forward song events to visuals
    song.on('sectionChange', (section) => {
      this.visuals.forEach(v => v.onSectionChange?.(section));
    });

    return song;
  }

  async loadVisual(visualManifest, layer = 0) {
    const VisualModule = await import(visualManifest.entryPath);
    const visual = new VisualModule.default();

    const canvas = document.createElement('canvas');
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;
    canvas.style.position = 'absolute';
    canvas.style.zIndex = layer;
    document.getElementById('visual-container').appendChild(canvas);

    visual.init(canvas);
    visual._canvas = canvas;
    this.visuals.push(visual);

    return visual;
  }
}
```

## Crossfade State Machine

```
     ┌─────────────┐
     │   IDLE_A    │  Deck A playing, Deck B empty
     └──────┬──────┘
            │ loadSong(B)
            ▼
     ┌─────────────┐
     │  READY_AB   │  Both decks loaded, A playing
     └──────┬──────┘
            │ startCrossfade(duration)
            ▼
     ┌─────────────┐
     │  FADING     │  Crossfading A → B
     └──────┬──────┘
            │ fade complete
            ▼
     ┌─────────────┐
     │   IDLE_B    │  Deck B playing, Deck A empty
     └──────┬──────┘
            │ loadSong(A)
            ▼
     ┌─────────────┐
     │  READY_BA   │  Both decks loaded, B playing
     └─────────────┘
            ...
```

### Crossfade Implementation

```javascript
class DJController {
  async startCrossfade(duration = 4) {
    const targetValue = this.activeDeck === 'A' ? 1 : 0;
    const targetDeck = this.activeDeck === 'A' ? 'B' : 'A';

    // Start the inactive deck playing
    const inactiveSong = this.activeDeck === 'A' ? this.deckB : this.deckA;
    await inactiveSong.play();

    // Fade
    this.crossfade.fade.rampTo(targetValue, duration);

    // After fade completes, clean up
    setTimeout(() => {
      const oldSong = this.activeDeck === 'A' ? this.deckA : this.deckB;
      oldSong.stop();

      this.activeDeck = targetDeck;

      // Emit event for UI
      this.emit('crossfadeComplete', { activeDeck: targetDeck });
    }, duration * 1000);
  }

  // Instant switch (no fade)
  switchDeck() {
    const targetValue = this.activeDeck === 'A' ? 1 : 0;
    this.crossfade.fade.value = targetValue;
    this.activeDeck = this.activeDeck === 'A' ? 'B' : 'A';
  }
}
```

## Event Bus

Communication between songs, visuals, and controller:

```javascript
class EventBus {
  constructor() {
    this.listeners = {};
  }

  on(event, callback) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(callback);
  }

  off(event, callback) {
    if (!this.listeners[event]) return;
    this.listeners[event] = this.listeners[event].filter(cb => cb !== callback);
  }

  emit(event, data) {
    if (!this.listeners[event]) return;
    this.listeners[event].forEach(cb => cb(data));
  }
}

// Global bus
const bus = new EventBus();

// Song emits
song.on('sectionChange', (section) => bus.emit('section', section));
song.on('bar', (bar) => bus.emit('bar', bar));

// Visual listens
bus.on('section', (section) => visual.onSectionChange(section));
bus.on('bar', (bar) => visual.onBar(bar));

// Controller emits
bus.emit('crossfadeStart', { from: 'A', to: 'B', duration: 4 });
bus.emit('crossfadeComplete', { activeDeck: 'B' });
```

## Animation Loop

Main render loop that drives visuals:

```javascript
class DJController {
  startRenderLoop() {
    const render = () => {
      // Get audio data
      const audioData = {
        frequencyData: this.analyser.getValue(),
        waveformData: this.waveformAnalyser.getValue(),
        volume: this.meter.getValue()
      };

      // Get song state from active deck
      const activeSong = this.activeDeck === 'A' ? this.deckA : this.deckB;
      const songState = activeSong ? activeSong.getState() : {
        section: 'unknown',
        bar: 0,
        beat: 0,
        bpm: 75,
        isPlaying: false
      };

      // Render all visuals
      this.visuals.forEach(visual => {
        visual.render(audioData, songState);
      });

      requestAnimationFrame(render);
    };

    render();
  }
}
```

## UI Integration

The controller exposes methods for UI controls:

```javascript
// Transport
controller.play();
controller.pause();
controller.stop();

// Deck management
controller.loadSong(songManifest, 'A');
controller.loadSong(songManifest, 'B');
controller.startCrossfade(4);

// Song control (forwarded to active deck)
controller.jumpToSection('climax');
controller.muteTrack('drums');
controller.setTempo(85);

// Visual management
controller.loadVisual(visualManifest, layer);
controller.removeVisual(visual);
controller.setVisualOption(visual, 'lineColor', '#ff0000');

// State
controller.getState();  // { activeDeck, deckA, deckB, visuals, ... }
```

## Directory Structure

```
dj-overlay/
  src/
    controller.js      # DJController class
    discovery.js       # Song/visual discovery
    audio-chain.js     # Tone.js routing setup
    event-bus.js       # Event communication
  index.html           # UI shell
  styles.css           # DJ interface styles
  manifest.json        # Controller metadata
```
