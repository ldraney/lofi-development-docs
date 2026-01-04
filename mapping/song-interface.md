# Song Interface

Standard interface for lofi song modules in the distributed ecosystem.

## TypeScript Interface

```typescript
interface SongState {
  isPlaying: boolean;
  currentBar: number;
  currentSection: string;
  bpm: number;
}

interface SectionConfig {
  drums: boolean;
  bass: boolean;
  chords: boolean;
  melody: boolean;
  filterFreq: number;    // Hz (400-3000)
  reverbWet: number;     // 0-1
  chordsVol?: number;    // dB
}

interface Song {
  // === Metadata ===
  readonly name: string;
  readonly bpm: number;
  readonly sections: Record<string, SectionConfig>;

  // === Lifecycle ===
  init(): Promise<void>;      // Initialize audio, await reverb.generate()
  dispose(): void;            // Clean up all Tone.js objects

  // === Transport ===
  play(): Promise<void>;      // Start playback (calls Tone.start())
  pause(): void;              // Pause transport
  stop(): void;               // Stop and reset to beginning

  // === Navigation ===
  jumpToSection(name: string): void;
  jumpToBar(bar: number): void;

  // === Track Control ===
  muteTrack(name: string): void;
  unmuteTrack(name: string): void;
  setTrackVolume(name: string, db: number): void;

  // === Global Controls ===
  setTempo(bpm: number, rampTime?: number): void;
  setFilterFreq(hz: number, rampTime?: number): void;
  setReverbWet(wet: number, rampTime?: number): void;

  // === State ===
  getState(): SongState;

  // === Audio Output ===
  getMasterOutput(): Tone.Volume;
  getAnalyser?(): Tone.Analyser;

  // === Events ===
  on(event: 'sectionChange', callback: (section: string) => void): void;
  on(event: 'bar', callback: (bar: number) => void): void;
  off(event: string, callback: Function): void;
}
```

## Example Implementation

```javascript
class LofiSong {
  constructor() {
    this.name = 'chill-vibes';
    this.bpm = 78;
    this.sections = {
      intro:  { drums: false, bass: false, chords: true, melody: false, filterFreq: 600, reverbWet: 0.6 },
      verse:  { drums: true,  bass: true,  chords: true, melody: false, filterFreq: 1400, reverbWet: 0.35 },
      climax: { drums: true,  bass: true,  chords: true, melody: true,  filterFreq: 2800, reverbWet: 0.45 },
      outro:  { drums: false, bass: false, chords: true, melody: false, filterFreq: 500, reverbWet: 0.6 }
    };

    this._listeners = { sectionChange: [], bar: [] };
    this._currentSection = 'intro';
    this._isInitialized = false;
  }

  async init() {
    // Master output
    this.masterVolume = new Tone.Volume(0);
    this.masterFilter = new Tone.Filter({ frequency: 600, type: 'lowpass' });
    this.reverb = new Tone.Reverb({ decay: 2.5, wet: 0.35 });
    await this.reverb.generate();

    // Instruments
    this.instruments = {
      kick: new Tone.MembraneSynth({ pitchDecay: 0.05, octaves: 6 }),
      snare: new Tone.NoiseSynth({ noise: { type: 'pink' }, envelope: { decay: 0.15 } }),
      hihat: new Tone.MetalSynth({ frequency: 200, envelope: { decay: 0.05 } }),
      chords: new Tone.PolySynth(Tone.Synth, { oscillator: { type: 'triangle' } }),
      bass: new Tone.Synth({ oscillator: { type: 'sine' } }),
      melody: new Tone.Synth({ oscillator: { type: 'triangle' } })
    };

    // Connect all to master chain
    Object.values(this.instruments).forEach(inst => {
      inst.connect(this.masterFilter);
    });
    this.masterFilter.connect(this.reverb);
    this.reverb.connect(this.masterVolume);

    // Set up patterns (Tone.Part for each layer)
    this._setupPatterns();

    // Set up section scheduling
    this._scheduleSections();

    // Transport config
    Tone.Transport.bpm.value = this.bpm;
    Tone.Transport.swing = 0.2;
    Tone.Transport.swingSubdivision = '16n';

    this._isInitialized = true;
  }

  _setupPatterns() {
    // Chord progression
    this.patterns = {};
    this.patterns.chords = new Tone.Part((time, chord) => {
      this.instruments.chords.triggerAttackRelease(chord.notes, chord.dur, time);
    }, [
      { time: '0:0:0', notes: ['D3', 'F3', 'A3', 'C4'], dur: '2n' },
      { time: '1:0:0', notes: ['G3', 'B3', 'D4', 'F4'], dur: '2n' },
      { time: '2:0:0', notes: ['C3', 'E3', 'G3', 'B3'], dur: '2n' },
      { time: '3:0:0', notes: ['A3', 'C4', 'E4', 'G4'], dur: '2n' }
    ]);
    this.patterns.chords.loop = true;
    this.patterns.chords.loopEnd = '4m';

    // Drums, bass, melody patterns...
    // (abbreviated for brevity)
  }

  _scheduleSections() {
    // Example: 8-bar intro, 16-bar verse, etc.
    Tone.Transport.schedule((time) => this._applySection('intro', time), '0:0:0');
    Tone.Transport.schedule((time) => this._applySection('verse', time), '8:0:0');
    Tone.Transport.schedule((time) => this._applySection('climax', time), '24:0:0');
    Tone.Transport.schedule((time) => this._applySection('outro', time), '40:0:0');
  }

  _applySection(name, time) {
    const config = this.sections[name];
    this._currentSection = name;

    // Filter and reverb
    this.masterFilter.frequency.rampTo(config.filterFreq, 0.5, time);
    this.reverb.wet.rampTo(config.reverbWet, 0.5, time);

    // Mute/unmute tracks
    this.instruments.kick.volume.rampTo(config.drums ? -8 : -Infinity, 0.3, time);
    this.instruments.snare.volume.rampTo(config.drums ? -10 : -Infinity, 0.3, time);
    this.instruments.hihat.volume.rampTo(config.drums ? -16 : -Infinity, 0.3, time);
    this.instruments.bass.volume.rampTo(config.bass ? -8 : -Infinity, 0.3, time);
    this.instruments.melody.volume.rampTo(config.melody ? -12 : -Infinity, 0.3, time);

    // Emit event
    this._emit('sectionChange', name);
  }

  async play() {
    await Tone.start();
    Object.values(this.patterns).forEach(p => p.start(0));
    Tone.Transport.start();
  }

  pause() {
    Tone.Transport.pause();
  }

  stop() {
    Tone.Transport.stop();
    Tone.Transport.position = 0;
  }

  jumpToSection(name) {
    const sectionBars = { intro: 0, verse: 8, climax: 24, outro: 40 };
    if (sectionBars[name] !== undefined) {
      Tone.Transport.position = `${sectionBars[name]}:0:0`;
      this._applySection(name, Tone.now());
    }
  }

  jumpToBar(bar) {
    Tone.Transport.position = `${bar}:0:0`;
  }

  muteTrack(name) {
    if (this.instruments[name]) {
      this.instruments[name].volume.rampTo(-Infinity, 0.3);
    }
  }

  unmuteTrack(name) {
    if (this.instruments[name]) {
      this.instruments[name].volume.rampTo(-10, 0.3);
    }
  }

  setTrackVolume(name, db) {
    if (this.instruments[name]) {
      this.instruments[name].volume.rampTo(db, 0.3);
    }
  }

  setTempo(bpm, rampTime = 2) {
    Tone.Transport.bpm.rampTo(bpm, rampTime);
  }

  setFilterFreq(hz, rampTime = 0.5) {
    this.masterFilter.frequency.rampTo(hz, rampTime);
  }

  setReverbWet(wet, rampTime = 0.5) {
    this.reverb.wet.rampTo(wet, rampTime);
  }

  getState() {
    const [bars] = Tone.Transport.position.split(':').map(Number);
    return {
      isPlaying: Tone.Transport.state === 'started',
      currentBar: bars,
      currentSection: this._currentSection,
      bpm: Tone.Transport.bpm.value
    };
  }

  getMasterOutput() {
    return this.masterVolume;
  }

  on(event, callback) {
    if (this._listeners[event]) {
      this._listeners[event].push(callback);
    }
  }

  off(event, callback) {
    if (this._listeners[event]) {
      this._listeners[event] = this._listeners[event].filter(cb => cb !== callback);
    }
  }

  _emit(event, data) {
    if (this._listeners[event]) {
      this._listeners[event].forEach(cb => cb(data));
    }
  }

  dispose() {
    Tone.Transport.stop();
    Tone.Transport.cancel();
    Object.values(this.patterns).forEach(p => p.dispose());
    Object.values(this.instruments).forEach(i => i.dispose());
    this.masterFilter.dispose();
    this.reverb.dispose();
    this.masterVolume.dispose();
  }
}

export default LofiSong;
```

## Song Repository Structure

Each `lofi-*-song/` repo should contain:

```
lofi-example-song/
  manifest.json      # Song metadata
  index.js           # Main export (implements Song interface)
  README.md          # Optional description
```

### manifest.json

```json
{
  "name": "chill-vibes",
  "displayName": "Chill Vibes",
  "bpm": 78,
  "duration": "3:24",
  "sections": ["intro", "verse", "climax", "outro"],
  "tags": ["chill", "jazz", "night"],
  "entry": "index.js"
}
```
