# Visual Interface

Standard interface for visual modules in the distributed lofi ecosystem.

## TypeScript Interface

```typescript
interface AudioData {
  frequencyData: Float32Array;   // FFT bins (typically 1024 values, -100 to 0 dB)
  waveformData: Float32Array;    // Waveform samples (-1 to 1)
  volume: number;                // Current RMS level (0-1)
}

interface SongState {
  section: string;      // Current section name
  bar: number;          // Current bar number
  beat: number;         // Current beat within bar (0-3)
  bpm: number;          // Tempo
  isPlaying: boolean;
}

interface Visual {
  // === Metadata ===
  readonly name: string;
  readonly description?: string;

  // === Lifecycle ===
  init(canvas: HTMLCanvasElement): void;
  dispose(): void;

  // === Rendering ===
  render(audioData: AudioData, songState: SongState): void;

  // === Optional Configuration ===
  setOption?(key: string, value: any): void;
  getOptions?(): Record<string, any>;
}
```

## Rendering Modes

### Audio-Reactive Only
Visual responds purely to audio data (frequency, waveform, volume):

```javascript
render(audioData, songState) {
  const { frequencyData, volume } = audioData;

  // React to bass (low frequencies)
  const bass = frequencyData.slice(0, 10).reduce((a, b) => a + b) / 10;
  const bassNormalized = (bass + 100) / 100; // Normalize -100..0 to 0..1

  // Scale circle based on bass
  this.ctx.beginPath();
  this.ctx.arc(this.cx, this.cy, 50 + bassNormalized * 100, 0, Math.PI * 2);
  this.ctx.fill();
}
```

### Song-Aware
Visual also responds to section changes and beat timing:

```javascript
render(audioData, songState) {
  const { section, beat, bpm } = songState;

  // Different color palette per section
  const palettes = {
    intro: ['#1a1a2e', '#16213e'],
    verse: ['#0f3460', '#533483'],
    climax: ['#e94560', '#ff6b6b'],
    outro: ['#1a1a2e', '#16213e']
  };

  const colors = palettes[section] || palettes.intro;

  // Pulse on beat
  const beatPulse = Math.sin(beat * Math.PI / 2);

  this.drawBackground(colors, beatPulse);
}
```

## Example Implementation

```javascript
class WaveformVisual {
  constructor() {
    this.name = 'waveform';
    this.description = 'Classic oscilloscope-style waveform display';
  }

  init(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.width = canvas.width;
    this.height = canvas.height;

    // Options with defaults
    this.options = {
      lineColor: '#00ff88',
      lineWidth: 2,
      backgroundColor: 'transparent'
    };
  }

  render(audioData, songState) {
    const { waveformData } = audioData;
    const { ctx, width, height } = this;

    // Clear
    ctx.clearRect(0, 0, width, height);
    if (this.options.backgroundColor !== 'transparent') {
      ctx.fillStyle = this.options.backgroundColor;
      ctx.fillRect(0, 0, width, height);
    }

    // Draw waveform
    ctx.strokeStyle = this.options.lineColor;
    ctx.lineWidth = this.options.lineWidth;
    ctx.beginPath();

    const sliceWidth = width / waveformData.length;
    let x = 0;

    for (let i = 0; i < waveformData.length; i++) {
      const v = waveformData[i];
      const y = (v + 1) / 2 * height;

      if (i === 0) {
        ctx.moveTo(x, y);
      } else {
        ctx.lineTo(x, y);
      }
      x += sliceWidth;
    }

    ctx.stroke();
  }

  setOption(key, value) {
    if (this.options.hasOwnProperty(key)) {
      this.options[key] = value;
    }
  }

  getOptions() {
    return { ...this.options };
  }

  dispose() {
    this.ctx = null;
    this.canvas = null;
  }
}

export default WaveformVisual;
```

## Frequency Bar Visual Example

```javascript
class FrequencyBars {
  constructor() {
    this.name = 'frequency-bars';
  }

  init(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.barCount = 64;
  }

  render(audioData, songState) {
    const { frequencyData } = audioData;
    const { section } = songState;
    const { ctx, canvas, barCount } = this;

    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Section-based color
    const hueBase = {
      intro: 200,
      verse: 280,
      climax: 0,
      outro: 200
    }[section] || 200;

    const barWidth = canvas.width / barCount;
    const step = Math.floor(frequencyData.length / barCount);

    for (let i = 0; i < barCount; i++) {
      const value = frequencyData[i * step];
      const percent = (value + 100) / 100; // -100..0 -> 0..1
      const barHeight = percent * canvas.height * 0.8;

      const hue = hueBase + (i / barCount) * 60;
      ctx.fillStyle = `hsl(${hue}, 70%, ${40 + percent * 30}%)`;

      ctx.fillRect(
        i * barWidth,
        canvas.height - barHeight,
        barWidth - 1,
        barHeight
      );
    }
  }

  dispose() {
    this.ctx = null;
  }
}

export default FrequencyBars;
```

## Visual Repository Structure

Each `visual-*/` repo should contain:

```
visual-waveform/
  manifest.json      # Visual metadata
  index.js           # Main export (implements Visual interface)
  README.md          # Optional description
```

### manifest.json

```json
{
  "name": "waveform",
  "displayName": "Waveform Display",
  "description": "Classic oscilloscope-style waveform visualization",
  "tags": ["classic", "minimal", "audio-reactive"],
  "songAware": false,
  "entry": "index.js",
  "options": {
    "lineColor": {
      "type": "color",
      "default": "#00ff88"
    },
    "lineWidth": {
      "type": "number",
      "default": 2,
      "min": 1,
      "max": 10
    }
  }
}
```

## Canvas Layering

The controller stacks multiple visual canvases:

```html
<div id="visual-container" style="position: relative;">
  <canvas id="visual-bg" style="position: absolute; z-index: 0;"></canvas>
  <canvas id="visual-mid" style="position: absolute; z-index: 1;"></canvas>
  <canvas id="visual-fg" style="position: absolute; z-index: 2;"></canvas>
</div>
```

Each visual should:
- Clear its canvas each frame (or use transparent background)
- Not assume it's the only visual rendering
- Support transparency for layering

## Animation Loop Integration

The controller calls `render()` on each visual every animation frame:

```javascript
function animate() {
  const audioData = {
    frequencyData: analyser.getValue(),
    waveformData: waveformAnalyser.getValue(),
    volume: meter.getValue()
  };

  const songState = activeSong.getState();

  visuals.forEach(visual => {
    visual.render(audioData, songState);
  });

  requestAnimationFrame(animate);
}
```
