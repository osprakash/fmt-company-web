# Technical Specification Document
## ClipFlow.ai - Short-Form Video Editor with Hybrid Processing

**Version:** 3.0  
**Date:** February 28, 2026  
**Status:** Draft (Updated with realistic browser limits and hybrid cloud architecture)

---

## 1. System Overview

### 1.1 Architecture Philosophy
ClipFlow.ai uses a **hybrid processing architecture**:
- **Browser-first (default)**: Videos under 20 min / 400MB process entirely client-side using WebCodecs, FFmpeg.wasm, and Whisper Web. Zero upload, instant start, complete privacy.
- **Cloud fallback (optional)**: Videos over 20 min / 400MB can upload to ClipFlow servers for processing. Same features, but requires upload time and credits.

This hybrid approach acknowledges browser memory limits (~2GB per tab) while still enabling the full use case for longer content.

### 1.2 Why Hybrid?
| Processing Mode | Best For | Tradeoffs |
|----------------|----------|-----------|
| **Browser** | Videos <20 min, <400MB | Instant start, private, free | Limited by browser memory |
| **Cloud** | Videos 20 min - 4 hours, 400MB - 5GB | Handles any size | Requires upload, uses credits |

**The browser is not a server.** Attempting to process 1-2GB files in a browser tab will crash. We embrace this limit rather than fight it.

### 1.3 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Browser)                                │
│                         For videos <20 min / <400MB                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐    │
│  │  React    │ │  FFmpeg   │ │  Whisper  │ │ MediaPipe │ │ Web Audio │    │
│  │  Frontend │ │  WASM     │ │  Web      │ │ Face Det. │ │ API       │    │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ └─────┬─────┘    │
│        │              │             │              │              │          │
│        └──────────────┴─────────────┴──────────────┴──────────────┘          │
│                                     │                                        │
│                           ┌─────────▼─────────┐                             │
│                           │   Video Pipeline   │                             │
│                           │   Orchestrator     │                             │
│                           └─────────┬─────────┘                             │
│                                     │                                        │
│  ┌────────────┐  ┌────────────┐  ┌──▼──────────┐  ┌────────────┐           │
│  │ Transcript │  │  Filler    │  │   Caption   │  │   Size     │           │
│  │ Editor     │  │  Detector  │  │   Renderer  │  │   Checker  │           │
│  └────────────┘  └────────────┘  └─────────────┘  └──────┬─────┘           │
│                                                           │                  │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐         │                  │
│  │  Brand     │  │  WebCodecs │  │  Web        │    Too large?              │
│  │  Template  │  │  (Decode)  │  │  Worker     │         │                  │
│  │  Manager   │  └─────────────┘  └────────────┘         │                  │
│  └────────────┘                                          ▼                  │
│  ┌────────────┐                              ┌────────────────────┐         │
│  │ IndexedDB  │                              │  Cloud Upload      │         │
│  │ (Cache)    │                              │  Prompt            │         │
│  └────────────┘                              └─────────┬──────────┘         │
│                                                        │                    │
└────────────────────────────────────────────────────────┼────────────────────┘
                                                         │
                                       ┌─────────────────┴─────────────────┐
                                       │                                   │
                                       ▼                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SERVER                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Next.js   │  │  Supabase   │  │   Stripe    │  │  Cloud      │        │
│  │   API       │  │  (Auth/DB)  │  │  (Payments) │  │  Storage    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  │  (R2/S3)    │        │
│                                                      └──────┬──────┘        │
│                                                             │               │
│  ┌──────────────────────────────────────────────────────────▼─────────┐    │
│  │                    CLOUD PROCESSING (for large videos)              │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │    │
│  │  │  Modal /    │  │  FFmpeg     │  │  Whisper    │                 │    │
│  │  │  Replicate  │  │  (Native)   │  │  (GPU)      │                 │    │
│  │  │  GPU        │  │             │  │             │                 │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                 │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Technology Stack

### 2.1 Frontend

| Layer | Technology | Purpose |
|-------|------------|---------|
| Framework | Next.js 14+ (App Router) | React framework with SSR/SSG |
| Language | TypeScript 5.x | Type safety |
| Styling | Tailwind CSS + shadcn/ui | Rapid UI development |
| State | Zustand | Lightweight state management |
| Video Processing | FFmpeg.wasm 0.12+ | Video encoding/decoding |
| Transcription | @remotion/whisper-web | Client-side speech-to-text |
| Face Detection | @mediapipe/tasks-vision | Subject tracking |
| Video Decode | WebCodecs API | Frame-level video access |
| Storage | IndexedDB (via Dexie.js) | Client-side file caching |
| Workers | Web Workers + Comlink | Off-main-thread processing |

### 2.2 Backend

| Layer | Technology | Purpose |
|-------|------------|---------|
| API | Next.js API Routes | Serverless functions |
| Database | Supabase (PostgreSQL) | User data, project metadata |
| Auth | Supabase Auth | Email + OAuth providers |
| Payments | Stripe | Subscriptions + credit purchases |
| Storage | Supabase Storage / R2 | Optional cloud video storage |
| AI (Future) | Replicate / Modal | Server-side AI processing |

### 2.3 Infrastructure

| Component | Service | Purpose |
|-----------|---------|---------|
| Hosting | Vercel | Frontend + API deployment |
| CDN | Vercel Edge / Cloudflare | Static assets + model files |
| Monitoring | Sentry | Error tracking |
| Analytics | PostHog | Product analytics |

---

## 3. Core Module Specifications

### 3.1 Video Pipeline Orchestrator

The central coordinator for all video processing operations.

```typescript
// src/lib/video/pipeline.ts

interface PipelineConfig {
  inputFile: File;
  operations: Operation[];
  outputFormat: OutputFormat;
  onProgress: (progress: PipelineProgress) => void;
}

interface Operation {
  type: 'trim' | 'caption' | 'reframe' | 'text-overlay' | 'filler-removal' | 'transcript-edit';
  params: TrimParams | CaptionParams | ReframeParams | TextOverlayParams | FillerRemovalParams | TranscriptEditParams;
}

interface PipelineProgress {
  stage: 'loading' | 'transcribing' | 'analyzing' | 'processing' | 'encoding' | 'complete';
  percent: number;
  currentOperation?: string;
  estimatedTimeRemaining?: number;
}

class VideoPipeline {
  private ffmpeg: FFmpeg;
  private whisper: WhisperWeb;
  private faceDetector: FaceDetector;
  
  async initialize(): Promise<void>;
  async process(config: PipelineConfig): Promise<Blob>;
  async abort(): void;
}
```

**Responsibilities**:
1. Load and initialize all processing libraries
2. Orchestrate operation sequence
3. Manage memory and cleanup
4. Report progress to UI
5. Handle errors gracefully

### 3.2 FFmpeg.wasm Module

Handles video encoding, decoding, trimming, and format conversion.

```typescript
// src/lib/video/ffmpeg-service.ts

interface FFmpegService {
  // Initialization
  load(): Promise<void>;
  isLoaded(): boolean;
  
  // Core operations
  trim(input: Uint8Array, start: number, end: number): Promise<Uint8Array>;
  crop(input: Uint8Array, cropRect: CropRect): Promise<Uint8Array>;
  addSubtitles(input: Uint8Array, assContent: string): Promise<Uint8Array>;
  
  // Export
  encode(input: Uint8Array, options: EncodeOptions): Promise<Uint8Array>;
  
  // Utilities
  getMetadata(input: Uint8Array): Promise<VideoMetadata>;
  extractAudio(input: Uint8Array): Promise<Uint8Array>;
}

interface EncodeOptions {
  format: 'mp4' | 'webm' | 'mov';
  codec: 'h264' | 'vp9' | 'hevc';
  resolution: '720p' | '1080p' | '4k';
  bitrate?: number;
  fps?: number;
}
```

**FFmpeg Commands Used**:

```bash
# Trim video
-ss {start} -to {end} -i input.mp4 -c copy output.mp4

# Crop/reframe (with re-encode)
-i input.mp4 -vf "crop={w}:{h}:{x}:{y}" -c:a copy output.mp4

# Add ASS subtitles (burned in)
-i input.mp4 -vf "ass=subtitles.ass" -c:a copy output.mp4

# Export with quality settings
-i input.mp4 -c:v libx264 -preset medium -crf 23 -c:a aac -b:a 128k output.mp4
```

**Memory Management**:
```typescript
// Use chunked processing for large files
const CHUNK_SIZE = 50 * 1024 * 1024; // 50MB chunks

async function processLargeFile(file: File): Promise<void> {
  const chunks = Math.ceil(file.size / CHUNK_SIZE);
  for (let i = 0; i < chunks; i++) {
    const chunk = file.slice(i * CHUNK_SIZE, (i + 1) * CHUNK_SIZE);
    await processChunk(chunk);
    // Allow garbage collection between chunks
    await new Promise(resolve => setTimeout(resolve, 100));
  }
}
```

### 3.3 Whisper Web Transcription Module

Client-side speech-to-text using OpenAI's Whisper model.

```typescript
// src/lib/transcription/whisper-service.ts

interface WhisperService {
  // Initialization
  loadModel(size: 'tiny' | 'base' | 'small'): Promise<void>;
  isModelLoaded(): boolean;
  
  // Transcription
  transcribe(audioData: Float32Array, options: TranscribeOptions): Promise<Transcript>;
  
  // Streaming (for real-time preview)
  transcribeStream(
    audioStream: ReadableStream<Float32Array>,
    onSegment: (segment: TranscriptSegment) => void
  ): Promise<void>;
}

interface TranscribeOptions {
  language?: string; // Auto-detect if not specified
  task: 'transcribe' | 'translate';
  wordTimestamps: boolean; // Required for animated captions
}

interface Transcript {
  text: string;
  language: string;
  segments: TranscriptSegment[];
  words: TranscriptWord[];
}

interface TranscriptWord {
  word: string;
  start: number; // seconds
  end: number;
  confidence: number;
}
```

**Model Selection Strategy**:
| Model | Size | Speed | Accuracy | Use Case |
|-------|------|-------|----------|----------|
| tiny | ~75MB | 10x realtime | ~90% | Quick preview |
| base | ~150MB | 5x realtime | ~94% | Default |
| small | ~500MB | 2x realtime | ~97% | High accuracy |

**Cross-Origin Isolation**:
Whisper Web requires SharedArrayBuffer, which needs these headers:

```typescript
// next.config.js
async headers() {
  return [
    {
      source: '/(.*)',
      headers: [
        { key: 'Cross-Origin-Opener-Policy', value: 'same-origin' },
        { key: 'Cross-Origin-Embedder-Policy', value: 'require-corp' },
      ],
    },
  ];
}
```

### 3.4 Face Detection Module

MediaPipe-based face detection for intelligent reframing.

```typescript
// src/lib/detection/face-detector-service.ts

interface FaceDetectorService {
  // Initialization
  initialize(): Promise<void>;
  
  // Detection
  detectFaces(frame: ImageData): Promise<FaceDetection[]>;
  
  // Tracking across frames
  trackFaces(
    videoElement: HTMLVideoElement,
    onFrame: (faces: FaceDetection[], timestamp: number) => void,
    options: TrackingOptions
  ): Promise<FaceTrack[]>;
}

interface FaceDetection {
  boundingBox: BoundingBox;
  keypoints: FaceKeypoint[];
  confidence: number;
}

interface FaceTrack {
  id: string;
  frames: Array<{
    timestamp: number;
    boundingBox: BoundingBox;
  }>;
  averagePosition: Point;
  screenTime: number; // Total time face is visible
}

interface TrackingOptions {
  sampleRate: number; // Frames per second to analyze
  minConfidence: number;
  smoothing: boolean; // Apply temporal smoothing
}
```

**Reframe Algorithm**:
```typescript
function calculateCropRegion(
  faces: FaceTrack[],
  sourceAspect: number,
  targetAspect: number
): CropRegion {
  // 1. Find primary face (most screen time)
  const primaryFace = faces.reduce((a, b) => 
    a.screenTime > b.screenTime ? a : b
  );
  
  // 2. Calculate center point with padding
  const center = primaryFace.averagePosition;
  const padding = 0.2; // 20% padding around face
  
  // 3. Calculate crop dimensions
  const cropWidth = sourceWidth * (targetAspect / sourceAspect);
  const cropHeight = sourceHeight;
  
  // 4. Position crop to keep face centered
  let cropX = center.x - cropWidth / 2;
  cropX = Math.max(0, Math.min(cropX, sourceWidth - cropWidth));
  
  return { x: cropX, y: 0, width: cropWidth, height: cropHeight };
}
```

### 3.5 Caption Renderer Module

Generates styled, animated captions.

```typescript
// src/lib/captions/caption-renderer.ts

interface CaptionStyle {
  fontFamily: string;
  fontSize: number;
  fontWeight: number;
  color: string;
  backgroundColor?: string;
  strokeColor?: string;
  strokeWidth?: number;
  position: 'top' | 'center' | 'bottom';
  animation: CaptionAnimation;
}

type CaptionAnimation = 
  | { type: 'none' }
  | { type: 'karaoke'; highlightColor: string }
  | { type: 'bounce'; intensity: number }
  | { type: 'fade'; duration: number }
  | { type: 'typewriter'; speed: number }
  | { type: 'glow'; color: string; intensity: number };

interface CaptionRenderer {
  // Generate ASS subtitle file for FFmpeg
  generateASS(transcript: Transcript, style: CaptionStyle): string;
  
  // Generate WebVTT for preview
  generateVTT(transcript: Transcript): string;
  
  // Render to canvas for real-time preview
  renderFrame(
    ctx: CanvasRenderingContext2D,
    transcript: Transcript,
    currentTime: number,
    style: CaptionStyle
  ): void;
}
```

**ASS Subtitle Generation**:
```typescript
function generateASS(transcript: Transcript, style: CaptionStyle): string {
  const header = `[Script Info]
ScriptType: v4.00+
PlayResX: 1920
PlayResY: 1080

[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, SecondaryColour, OutlineColour, BackColour, Bold, Italic, Underline, StrikeOut, ScaleX, ScaleY, Spacing, Angle, BorderStyle, Outline, Shadow, Alignment, MarginL, MarginR, MarginV, Encoding
Style: Default,${style.fontFamily},${style.fontSize},&H00FFFFFF,&H000000FF,&H00000000,&H80000000,0,0,0,0,100,100,0,0,1,2,0,2,10,10,30,1

[Events]
Format: Layer, Start, End, Style, Name, MarginL, MarginR, MarginV, Effect, Text
`;

  const events = transcript.segments.map(segment => {
    const start = formatASSTime(segment.start);
    const end = formatASSTime(segment.end);
    const text = applyAnimation(segment.text, style.animation);
    return `Dialogue: 0,${start},${end},Default,,0,0,0,,${text}`;
  }).join('\n');

  return header + events;
}
```

### 3.6 Transcript Editor Module

Enables video editing through transcript text manipulation.

```typescript
// src/lib/transcript/transcript-editor.ts

interface TranscriptEdit {
  id: string;
  type: 'delete' | 'split' | 'merge';
  wordRange: { startIdx: number; endIdx: number };
  timeRange: { start: number; end: number };
  originalText: string;
}

interface CutList {
  excludedRanges: Array<{ start: number; end: number }>;
  totalRemovedDuration: number;
}

interface TranscriptEditorService {
  // Convert word selection to time ranges
  wordRangeToTimeRange(
    words: TranscriptWord[],
    startIdx: number,
    endIdx: number
  ): { start: number; end: number };

  // Apply edits and generate cut list
  generateCutList(edits: TranscriptEdit[]): CutList;

  // Build FFmpeg concat filter from cut list
  generateConcatFilter(
    totalDuration: number,
    cutList: CutList
  ): string[];

  // Search transcript
  searchTranscript(
    words: TranscriptWord[],
    query: string
  ): Array<{ wordIdx: number; word: TranscriptWord }>;
}

interface TranscriptEditParams {
  edits: TranscriptEdit[];
  crossfadeDuration: number; // ms of crossfade between cuts (0 = hard cut)
}
```

**FFmpeg Concat Approach**:
```bash
# Generate kept segments as individual files, then concatenate
-i input.mp4 -ss 0 -to 5.2 -c copy segment_0.mp4
-i input.mp4 -ss 8.1 -to 15.0 -c copy segment_1.mp4
-i input.mp4 -ss 18.3 -to 45.0 -c copy segment_2.mp4

# Concat demuxer
-f concat -safe 0 -i segments.txt -c copy output.mp4
```

**Cut Point Audio Smoothing**:
```typescript
// Apply short crossfade at cut points to avoid audio pops
function generateCutWithCrossfade(
  cutList: CutList,
  crossfadeMs: number = 20
): string[] {
  const ffmpegArgs: string[] = [];
  const segments = getKeptSegments(cutList);

  segments.forEach((seg, i) => {
    const fadeIn = i > 0 ? crossfadeMs / 1000 : 0;
    const fadeOut = i < segments.length - 1 ? crossfadeMs / 1000 : 0;

    if (fadeIn > 0 || fadeOut > 0) {
      ffmpegArgs.push(
        `-af`, `afade=t=in:d=${fadeIn},afade=t=out:st=${seg.duration - fadeOut}:d=${fadeOut}`
      );
    }
  });

  return ffmpegArgs;
}
```

### 3.7 Filler Word & Silence Detector Module

Detects filler words and silence gaps for automatic cleanup.

```typescript
// src/lib/analysis/filler-detector.ts

interface FillerDetection {
  id: string;
  type: 'filler-word' | 'silence';
  word?: string;
  start: number;
  end: number;
  confidence: number;
}

interface FillerDetectorConfig {
  fillerWords: string[];
  silenceThresholdDb: number;     // default: -40 dB
  minSilenceDuration: number;     // default: 0.5 seconds
  contextWindow: number;          // words before/after to check for false positives
}

const DEFAULT_FILLER_WORDS = [
  'um', 'uh', 'uh,', 'um,', 'ah', 'er',
  'like,', 'you know,', 'you know', 'so,',
  'basically,', 'basically', 'literally,', 'literally',
  'right,', 'right?', 'i mean,', 'i mean',
  'kind of', 'sort of', 'actually,', 'actually',
];

interface FillerDetectorService {
  // Detect filler words from transcript
  detectFillerWords(
    words: TranscriptWord[],
    config?: Partial<FillerDetectorConfig>
  ): FillerDetection[];

  // Detect silence from audio data
  detectSilence(
    audioData: Float32Array,
    sampleRate: number,
    config?: Partial<FillerDetectorConfig>
  ): FillerDetection[];

  // Combined detection
  detectAll(
    words: TranscriptWord[],
    audioData: Float32Array,
    sampleRate: number,
    config?: Partial<FillerDetectorConfig>
  ): FillerDetection[];

  // Convert detections to cut list
  detectionsToCutList(detections: FillerDetection[]): CutList;

  // Statistics
  getStats(detections: FillerDetection[]): FillerStats;
}

interface FillerStats {
  totalFillerWords: number;
  totalSilenceGaps: number;
  fillerDuration: number;
  silenceDuration: number;
  totalSavings: number;
  fillerWordBreakdown: Record<string, number>;
}

interface FillerRemovalParams {
  removeFillerWords: boolean;
  removeSilence: boolean;
  silenceThresholdDb: number;
  minSilenceDuration: number;
  customFillerWords: string[];
}
```

**Silence Detection Algorithm**:
```typescript
function detectSilence(
  audioData: Float32Array,
  sampleRate: number,
  thresholdDb: number = -40,
  minDuration: number = 0.5
): FillerDetection[] {
  const windowSize = Math.floor(sampleRate * 0.05); // 50ms windows
  const threshold = Math.pow(10, thresholdDb / 20);
  const detections: FillerDetection[] = [];

  let silenceStart: number | null = null;

  for (let i = 0; i < audioData.length; i += windowSize) {
    const window = audioData.slice(i, i + windowSize);
    const rms = Math.sqrt(
      window.reduce((sum, sample) => sum + sample * sample, 0) / window.length
    );

    const timeSeconds = i / sampleRate;

    if (rms < threshold) {
      if (silenceStart === null) silenceStart = timeSeconds;
    } else {
      if (silenceStart !== null) {
        const duration = timeSeconds - silenceStart;
        if (duration >= minDuration) {
          detections.push({
            id: generateId(),
            type: 'silence',
            start: silenceStart,
            end: timeSeconds,
            confidence: 1.0,
          });
        }
        silenceStart = null;
      }
    }
  }

  return detections;
}
```

### 3.8 Text Overlay Renderer Module

Manages text overlays, titles, and lower-third graphics.

```typescript
// src/lib/overlays/text-overlay-renderer.ts

interface TextOverlay {
  id: string;
  text: string;
  position: { x: number; y: number }; // 0-1 normalized coordinates
  startTime: number;
  endTime: number;
  style: TextOverlayStyle;
  animation: OverlayAnimation;
}

interface TextOverlayStyle {
  fontFamily: string;
  fontSize: number;
  fontWeight: number;
  color: string;
  backgroundColor?: string;
  backgroundPadding?: number;
  strokeColor?: string;
  strokeWidth?: number;
  shadowColor?: string;
  shadowBlur?: number;
  shadowOffsetX?: number;
  shadowOffsetY?: number;
  textAlign: 'left' | 'center' | 'right';
  maxWidth?: number;
}

type OverlayAnimation =
  | { type: 'none' }
  | { type: 'fade'; fadeInDuration: number; fadeOutDuration: number }
  | { type: 'slide-up'; duration: number }
  | { type: 'slide-down'; duration: number }
  | { type: 'typewriter'; speed: number }
  | { type: 'pop'; intensity: number };

interface LowerThirdTemplate {
  id: string;
  name: string;
  nameStyle: TextOverlayStyle;
  titleStyle: TextOverlayStyle;
  barColor: string;
  barHeight: number;
  position: { x: number; y: number };
  animation: OverlayAnimation;
}

interface TextOverlayRenderer {
  // Canvas preview rendering
  renderFrame(
    ctx: CanvasRenderingContext2D,
    overlays: TextOverlay[],
    currentTime: number,
    canvasSize: { width: number; height: number }
  ): void;

  // Generate FFmpeg drawtext filter for export
  generateDrawtextFilter(overlays: TextOverlay[]): string;

  // Hit testing for drag interaction
  hitTest(
    overlays: TextOverlay[],
    point: { x: number; y: number },
    currentTime: number
  ): TextOverlay | null;
}

interface TextOverlayParams {
  overlays: TextOverlay[];
}
```

**Canvas Rendering with Animation**:
```typescript
function renderOverlayWithAnimation(
  ctx: CanvasRenderingContext2D,
  overlay: TextOverlay,
  currentTime: number,
  canvasSize: { width: number; height: number }
): void {
  const elapsed = currentTime - overlay.startTime;
  const remaining = overlay.endTime - currentTime;

  ctx.save();

  // Apply animation transforms
  switch (overlay.animation.type) {
    case 'fade': {
      const { fadeInDuration, fadeOutDuration } = overlay.animation;
      let alpha = 1;
      if (elapsed < fadeInDuration) alpha = elapsed / fadeInDuration;
      if (remaining < fadeOutDuration) alpha = Math.min(alpha, remaining / fadeOutDuration);
      ctx.globalAlpha = Math.max(0, Math.min(1, alpha));
      break;
    }
    case 'slide-up': {
      const progress = Math.min(1, elapsed / overlay.animation.duration);
      const eased = 1 - Math.pow(1 - progress, 3); // ease-out cubic
      const offsetY = (1 - eased) * 50;
      ctx.translate(0, offsetY);
      ctx.globalAlpha = eased;
      break;
    }
    case 'pop': {
      const progress = Math.min(1, elapsed / 0.3);
      const scale = progress < 0.6
        ? 1 + overlay.animation.intensity * Math.sin(progress / 0.6 * Math.PI)
        : 1;
      const px = overlay.position.x * canvasSize.width;
      const py = overlay.position.y * canvasSize.height;
      ctx.translate(px, py);
      ctx.scale(scale, scale);
      ctx.translate(-px, -py);
      break;
    }
  }

  // Draw text with style
  const { style } = overlay;
  ctx.font = `${style.fontWeight} ${style.fontSize}px ${style.fontFamily}`;
  ctx.textAlign = style.textAlign;

  const x = overlay.position.x * canvasSize.width;
  const y = overlay.position.y * canvasSize.height;

  if (style.backgroundColor) {
    const metrics = ctx.measureText(overlay.text);
    const pad = style.backgroundPadding ?? 8;
    ctx.fillStyle = style.backgroundColor;
    ctx.fillRect(
      x - pad, y - style.fontSize - pad,
      metrics.width + pad * 2, style.fontSize + pad * 2
    );
  }

  if (style.strokeColor && style.strokeWidth) {
    ctx.strokeStyle = style.strokeColor;
    ctx.lineWidth = style.strokeWidth;
    ctx.strokeText(overlay.text, x, y);
  }

  ctx.fillStyle = style.color;
  ctx.fillText(overlay.text, x, y);
  ctx.restore();
}
```

### 3.9 Brand Template Manager Module

Persists and applies reusable style configurations.

```typescript
// src/lib/brand/brand-template-manager.ts

interface BrandTemplate {
  id: string;
  name: string;
  createdAt: Date;
  updatedAt: Date;

  // Caption settings
  captionStyle: CaptionStyle;

  // Text overlay defaults
  overlayDefaults: TextOverlayStyle;

  // Color palette
  colorPalette: {
    primary: string;
    secondary: string;
    accent: string;
    text: string;
    background: string;
  };

  // Font selections
  fonts: {
    heading: string;
    body: string;
    caption: string;
  };

  // Logo/watermark config (optional)
  logoConfig?: {
    imageDataUrl: string; // base64 data URL stored locally
    position: { x: number; y: number };
    size: { width: number; height: number };
    opacity: number;
  };

  // Thumbnail preview (auto-generated)
  thumbnailDataUrl?: string;
}

interface BrandTemplateManager {
  // CRUD operations
  listTemplates(): Promise<BrandTemplate[]>;
  getTemplate(id: string): Promise<BrandTemplate | null>;
  saveTemplate(template: BrandTemplate): Promise<void>;
  deleteTemplate(id: string): Promise<void>;

  // Apply to editor state
  applyTemplate(templateId: string, editorState: EditorState): Partial<EditorState>;

  // Import/Export
  exportTemplate(id: string): Promise<string>; // JSON string
  importTemplate(json: string): Promise<BrandTemplate>;

  // Sync with server (for authenticated users)
  syncToServer(userId: string): Promise<void>;
  syncFromServer(userId: string): Promise<void>;

  // Generate preview thumbnail
  generateThumbnail(template: BrandTemplate): Promise<string>;
}
```

**Storage Strategy**:
```typescript
// IndexedDB for local persistence (works for free/anonymous users)
// Supabase for cloud sync (authenticated users)

class BrandTemplateStore {
  private local: Dexie.Table<BrandTemplate>;
  private remote: SupabaseClient;

  async save(template: BrandTemplate, userId?: string): Promise<void> {
    // Always save locally
    await this.local.put(template);

    // Sync to server if authenticated
    if (userId) {
      await this.remote.from('brand_templates').upsert({
        id: template.id,
        user_id: userId,
        name: template.name,
        config: JSON.stringify(template),
        updated_at: new Date().toISOString(),
      });
    }
  }

  async list(userId?: string): Promise<BrandTemplate[]> {
    const local = await this.local.toArray();

    if (userId) {
      const { data: remote } = await this.remote
        .from('brand_templates')
        .select('*')
        .eq('user_id', userId);

      return this.mergeTemplates(local, remote ?? []);
    }

    return local;
  }
}
```

### 3.10 Cloud Processing Service Module

Handles upload and server-side processing for videos that exceed browser limits.

```typescript
// src/lib/cloud/cloud-processing-service.ts

interface CloudProcessingConfig {
  maxFileSize: number;        // Max file size in bytes (tier-dependent)
  maxDuration: number;        // Max duration in seconds (tier-dependent)
  creditCostPerMinute: number; // Credits charged per minute of input video
}

interface CloudUploadProgress {
  stage: 'preparing' | 'uploading' | 'processing' | 'complete' | 'error';
  uploadPercent: number;
  processingPercent: number;
  estimatedTimeRemaining?: number;
  error?: string;
}

interface CloudProcessingJob {
  id: string;
  userId: string;
  status: 'pending' | 'uploading' | 'processing' | 'complete' | 'failed';
  inputFileUrl: string;
  outputFileUrl?: string;
  operations: Operation[];
  creditsCharged: number;
  createdAt: Date;
  completedAt?: Date;
}

interface CloudProcessingService {
  // Check if video requires cloud processing
  requiresCloudProcessing(file: File, userTier: string): {
    required: boolean;
    reason?: 'file_size' | 'duration';
    browserLimit: { maxSize: number; maxDuration: number };
  };

  // Estimate credit cost
  estimateCreditCost(durationSeconds: number): number;

  // Upload video for cloud processing
  uploadForProcessing(
    file: File,
    operations: Operation[],
    onProgress: (progress: CloudUploadProgress) => void
  ): Promise<CloudProcessingJob>;

  // Check job status
  getJobStatus(jobId: string): Promise<CloudProcessingJob>;

  // Download processed video
  downloadResult(jobId: string): Promise<Blob>;

  // Cancel job (if still uploading/pending)
  cancelJob(jobId: string): Promise<void>;
}

const BROWSER_LIMITS = {
  free:     { maxSize: 200 * 1024 * 1024, maxDuration: 10 * 60 },  // 200MB, 10 min
  starter:  { maxSize: 300 * 1024 * 1024, maxDuration: 15 * 60 },  // 300MB, 15 min
  pro:      { maxSize: 400 * 1024 * 1024, maxDuration: 20 * 60 },  // 400MB, 20 min
  business: { maxSize: 400 * 1024 * 1024, maxDuration: 20 * 60 },  // 400MB, 20 min
};

const CLOUD_LIMITS = {
  starter:  { maxSize: 1 * 1024 * 1024 * 1024, maxDuration: 60 * 60 },     // 1GB, 60 min
  pro:      { maxSize: 2 * 1024 * 1024 * 1024, maxDuration: 2 * 60 * 60 }, // 2GB, 2 hours
  business: { maxSize: 5 * 1024 * 1024 * 1024, maxDuration: 4 * 60 * 60 }, // 5GB, 4 hours
};
```

**Cloud Processing Flow**:
```typescript
async function processInCloud(
  file: File,
  operations: Operation[],
  userTier: string,
  onProgress: (progress: CloudUploadProgress) => void
): Promise<Blob> {
  // 1. Validate file against cloud limits
  const cloudLimits = CLOUD_LIMITS[userTier];
  if (file.size > cloudLimits.maxSize) {
    throw new Error(`File too large for ${userTier} tier. Max: ${cloudLimits.maxSize / 1024 / 1024}MB`);
  }

  // 2. Check credit balance
  const estimatedCredits = Math.ceil(file.duration / 60) * 2; // 2 credits per minute
  const balance = await getCreditsBalance();
  if (balance < estimatedCredits && userTier !== 'business') {
    throw new Error(`Insufficient credits. Need ${estimatedCredits}, have ${balance}`);
  }

  // 3. Upload to cloud storage (chunked, resumable)
  onProgress({ stage: 'uploading', uploadPercent: 0, processingPercent: 0 });
  const uploadUrl = await uploadToStorage(file, (percent) => {
    onProgress({ stage: 'uploading', uploadPercent: percent, processingPercent: 0 });
  });

  // 4. Submit processing job
  onProgress({ stage: 'processing', uploadPercent: 100, processingPercent: 0 });
  const job = await submitProcessingJob(uploadUrl, operations);

  // 5. Poll for completion
  while (job.status !== 'complete' && job.status !== 'failed') {
    await sleep(2000);
    const updated = await getJobStatus(job.id);
    onProgress({
      stage: 'processing',
      uploadPercent: 100,
      processingPercent: updated.processingPercent || 0,
    });
    if (updated.status === 'failed') {
      throw new Error(updated.error || 'Processing failed');
    }
  }

  // 6. Download result
  onProgress({ stage: 'complete', uploadPercent: 100, processingPercent: 100 });
  return await downloadResult(job.id);
}
```

**Server-Side Processing (Modal/Replicate)**:
```typescript
// Server-side handler (runs on Modal/Replicate GPU)
async function processVideoOnServer(
  inputUrl: string,
  operations: Operation[]
): Promise<string> {
  // Download input from cloud storage
  const inputPath = await downloadFromStorage(inputUrl);
  
  // Run FFmpeg with native performance (not WASM)
  // A 10-minute video processes in ~30 seconds vs 5+ minutes in browser
  const outputPath = await runFFmpegNative(inputPath, operations);
  
  // Run Whisper with GPU acceleration if needed
  if (operations.some(op => op.type === 'caption' || op.type === 'transcript-edit')) {
    const transcript = await runWhisperGPU(inputPath);
    // Apply transcript-based operations
  }
  
  // Upload result to cloud storage
  const outputUrl = await uploadToStorage(outputPath);
  
  // Cleanup
  await deleteFile(inputPath);
  await deleteFile(outputPath);
  
  return outputUrl;
}
```

---

## 4. Data Models

### 4.1 Database Schema (Supabase/PostgreSQL)

```sql
-- Users (managed by Supabase Auth)
-- Additional user data
CREATE TABLE user_profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id),
  display_name TEXT,
  avatar_url TEXT,
  tier TEXT DEFAULT 'free' CHECK (tier IN ('free', 'starter', 'pro', 'business')),
  credits_balance INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Projects (video editing sessions)
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES user_profiles(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft', 'processing', 'completed', 'failed')),
  
  -- Source video metadata (stored client-side, metadata only here)
  source_filename TEXT,
  source_duration FLOAT,
  source_resolution TEXT,
  source_size_bytes BIGINT,
  
  -- Project settings (JSON for flexibility)
  settings JSONB DEFAULT '{}',
  
  -- Timestamps
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  last_exported_at TIMESTAMPTZ
);

-- Export history
CREATE TABLE exports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,
  user_id UUID REFERENCES user_profiles(id) ON DELETE CASCADE,
  
  -- Export details
  format TEXT NOT NULL,
  resolution TEXT NOT NULL,
  duration FLOAT,
  file_size_bytes BIGINT,
  
  -- Processing info
  processing_location TEXT CHECK (processing_location IN ('browser', 'server')),
  credits_used INTEGER DEFAULT 0,
  processing_time_ms INTEGER,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Credit transactions
CREATE TABLE credit_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES user_profiles(id) ON DELETE CASCADE,
  
  type TEXT NOT NULL CHECK (type IN ('purchase', 'subscription', 'usage', 'refund', 'bonus')),
  amount INTEGER NOT NULL, -- Positive for additions, negative for usage
  balance_after INTEGER NOT NULL,
  
  -- Reference to related entity
  reference_type TEXT, -- 'export', 'subscription', 'purchase'
  reference_id UUID,
  
  description TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Brand templates (synced for authenticated users)
CREATE TABLE brand_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES user_profiles(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  config JSONB NOT NULL, -- Full BrandTemplate JSON
  thumbnail_url TEXT,
  is_public BOOLEAN DEFAULT FALSE, -- For future community gallery
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_projects_user ON projects(user_id);
CREATE INDEX idx_exports_user ON exports(user_id);
CREATE INDEX idx_exports_project ON exports(project_id);
CREATE INDEX idx_credit_transactions_user ON credit_transactions(user_id);
CREATE INDEX idx_brand_templates_user ON brand_templates(user_id);
```

### 4.2 Client-Side State (Zustand)

```typescript
// src/store/editor-store.ts

interface EditorState {
  // Project
  project: Project | null;
  isDirty: boolean;
  
  // Video source
  sourceFile: File | null;
  sourceMetadata: VideoMetadata | null;
  audioData: Float32Array | null;
  
  // Timeline
  trimStart: number;
  trimEnd: number;
  currentTime: number;
  isPlaying: boolean;
  
  // Transcript
  transcript: Transcript | null;
  isTranscribing: boolean;
  transcriptEdits: Map<string, string>; // word id -> edited text
  
  // Transcript-Based Editing
  transcriptCutEdits: TranscriptEdit[];
  cutList: CutList | null;
  
  // Captions
  captionStyle: CaptionStyle;
  captionsEnabled: boolean;
  
  // Reframe
  reframeEnabled: boolean;
  targetAspectRatio: AspectRatio;
  cropRegion: CropRegion | null;
  faceTracking: FaceTrack[] | null;
  
  // Filler Detection
  fillerDetections: FillerDetection[];
  fillerDetectionConfig: FillerDetectorConfig;
  isAnalyzingFillers: boolean;
  
  // Text Overlays
  textOverlays: TextOverlay[];
  selectedOverlayId: string | null;
  
  // Brand Templates
  activeBrandTemplate: BrandTemplate | null;
  
  // Smart Clip Suggestions
  clipSuggestions: ClipSuggestion[];
  isAnalyzingSuggestions: boolean;
  clipSuggestionConfig: ClipSuggesterConfig;
  
  // Export
  exportSettings: ExportSettings;
  exportProgress: ExportProgress | null;
  
  // Actions — Core
  setSourceFile: (file: File) => Promise<void>;
  setTrimRange: (start: number, end: number) => void;
  updateCaptionStyle: (style: Partial<CaptionStyle>) => void;
  setTargetAspectRatio: (ratio: AspectRatio) => void;
  startExport: () => Promise<Blob>;
  
  // Actions — Transcript Editing
  addTranscriptCut: (startIdx: number, endIdx: number) => void;
  removeTranscriptCut: (editId: string) => void;
  clearAllCuts: () => void;
  
  // Actions — Filler Detection
  runFillerDetection: () => Promise<void>;
  removeAllFillers: () => void;
  toggleFillerDetection: (detectionId: string) => void;
  
  // Actions — Text Overlays
  addTextOverlay: (overlay: TextOverlay) => void;
  updateTextOverlay: (id: string, updates: Partial<TextOverlay>) => void;
  removeTextOverlay: (id: string) => void;
  selectOverlay: (id: string | null) => void;
  
  // Actions — Brand Templates
  applyBrandTemplate: (templateId: string) => Promise<void>;
  saveCurrentAsBrandTemplate: (name: string) => Promise<void>;
  
  // Actions — Clip Suggestions
  runClipAnalysis: () => Promise<void>;
  applySuggestion: (suggestionId: string) => void;
}
```

### 4.3 IndexedDB Schema (Dexie.js)

```typescript
// src/lib/storage/db.ts

import Dexie, { Table } from 'dexie';

interface CachedVideo {
  id: string;
  filename: string;
  data: ArrayBuffer;
  metadata: VideoMetadata;
  createdAt: Date;
  lastAccessedAt: Date;
}

interface CachedModel {
  id: string;
  name: string;
  version: string;
  data: ArrayBuffer;
  downloadedAt: Date;
}

interface ProjectDraft {
  id: string;
  projectId: string;
  state: EditorState;
  savedAt: Date;
}

interface LocalBrandTemplate {
  id: string;
  name: string;
  config: BrandTemplate;
  thumbnailDataUrl?: string;
  createdAt: Date;
  updatedAt: Date;
  syncedAt?: Date; // null = never synced to server
}

class ClipFlowDB extends Dexie {
  videos!: Table<CachedVideo>;
  models!: Table<CachedModel>;
  drafts!: Table<ProjectDraft>;
  brandTemplates!: Table<LocalBrandTemplate>;

  constructor() {
    super('clipflow');
    this.version(2).stores({
      videos: 'id, filename, lastAccessedAt',
      models: 'id, name, version',
      drafts: 'id, projectId, savedAt',
      brandTemplates: 'id, name, updatedAt'
    });
  }
}

export const db = new ClipFlowDB();
```

---

## 5. API Specifications

### 5.1 REST API Endpoints

```typescript
// Authentication (handled by Supabase)
POST   /auth/signup
POST   /auth/login
POST   /auth/logout
POST   /auth/refresh

// User Profile
GET    /api/user/profile
PATCH  /api/user/profile
GET    /api/user/credits

// Projects
GET    /api/projects
POST   /api/projects
GET    /api/projects/:id
PATCH  /api/projects/:id
DELETE /api/projects/:id

// Exports
POST   /api/projects/:id/exports
GET    /api/exports
GET    /api/exports/:id

// Credits
GET    /api/credits/balance
GET    /api/credits/transactions
POST   /api/credits/purchase

// Subscriptions
GET    /api/subscription
POST   /api/subscription/checkout
POST   /api/subscription/portal

// Brand Templates (cloud sync for authenticated users)
GET    /api/brand-templates
POST   /api/brand-templates
GET    /api/brand-templates/:id
PATCH  /api/brand-templates/:id
DELETE /api/brand-templates/:id
POST   /api/brand-templates/sync          // Bulk sync local → server
```

### 5.2 API Response Formats

```typescript
// Success response
interface ApiResponse<T> {
  success: true;
  data: T;
}

// Error response
interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
  };
}

// Paginated response
interface PaginatedResponse<T> {
  success: true;
  data: T[];
  pagination: {
    page: number;
    pageSize: number;
    totalItems: number;
    totalPages: number;
  };
}
```

### 5.3 Stripe Integration

```typescript
// src/lib/payments/stripe.ts

// Credit purchase checkout
async function createCreditCheckout(
  userId: string,
  creditPackage: 'small' | 'medium' | 'large'
): Promise<{ checkoutUrl: string }> {
  const prices = {
    small: { credits: 50, priceId: 'price_xxx' },
    medium: { credits: 150, priceId: 'price_yyy' },
    large: { credits: 500, priceId: 'price_zzz' },
  };
  
  const session = await stripe.checkout.sessions.create({
    mode: 'payment',
    customer: stripeCustomerId,
    line_items: [{ price: prices[creditPackage].priceId, quantity: 1 }],
    success_url: `${baseUrl}/credits/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${baseUrl}/credits/cancel`,
    metadata: {
      userId,
      credits: prices[creditPackage].credits,
    },
  });
  
  return { checkoutUrl: session.url };
}

// Subscription checkout
async function createSubscriptionCheckout(
  userId: string,
  tier: 'starter' | 'pro' | 'business'
): Promise<{ checkoutUrl: string }> {
  const priceIds = {
    starter: 'price_starter_monthly',
    pro: 'price_pro_monthly',
    business: 'price_business_monthly',
  };
  
  const session = await stripe.checkout.sessions.create({
    mode: 'subscription',
    customer: stripeCustomerId,
    line_items: [{ price: priceIds[tier], quantity: 1 }],
    success_url: `${baseUrl}/subscription/success`,
    cancel_url: `${baseUrl}/pricing`,
  });
  
  return { checkoutUrl: session.url };
}

// Webhook handler
async function handleStripeWebhook(event: Stripe.Event): Promise<void> {
  switch (event.type) {
    case 'checkout.session.completed':
      await handleCheckoutComplete(event.data.object);
      break;
    case 'customer.subscription.updated':
      await handleSubscriptionUpdate(event.data.object);
      break;
    case 'customer.subscription.deleted':
      await handleSubscriptionCanceled(event.data.object);
      break;
    case 'invoice.payment_failed':
      await handlePaymentFailed(event.data.object);
      break;
  }
}
```

---

## 6. Security Considerations

### 6.1 Client-Side Security

```typescript
// File validation with browser vs cloud limits
const ALLOWED_VIDEO_TYPES = ['video/mp4', 'video/webm', 'video/quicktime'];

const BROWSER_LIMITS = {
  free:     { maxSize: 200 * 1024 * 1024, maxDuration: 10 * 60 },
  starter:  { maxSize: 300 * 1024 * 1024, maxDuration: 15 * 60 },
  pro:      { maxSize: 400 * 1024 * 1024, maxDuration: 20 * 60 },
  business: { maxSize: 400 * 1024 * 1024, maxDuration: 20 * 60 },
};

const CLOUD_LIMITS = {
  free:     null, // No cloud processing for free tier
  starter:  { maxSize: 1 * 1024 * 1024 * 1024, maxDuration: 60 * 60 },
  pro:      { maxSize: 2 * 1024 * 1024 * 1024, maxDuration: 2 * 60 * 60 },
  business: { maxSize: 5 * 1024 * 1024 * 1024, maxDuration: 4 * 60 * 60 },
};

interface ValidationResult {
  valid: boolean;
  processingMode: 'browser' | 'cloud' | 'rejected';
  error?: string;
  cloudRequired?: boolean;
  estimatedCredits?: number;
}

function validateVideoFile(file: File, duration: number, userTier: string): ValidationResult {
  if (!ALLOWED_VIDEO_TYPES.includes(file.type)) {
    return { valid: false, processingMode: 'rejected', error: 'Unsupported video format' };
  }
  
  const browserLimit = BROWSER_LIMITS[userTier];
  const cloudLimit = CLOUD_LIMITS[userTier];
  
  // Check if browser can handle it
  if (file.size <= browserLimit.maxSize && duration <= browserLimit.maxDuration) {
    return { valid: true, processingMode: 'browser' };
  }
  
  // Check if cloud can handle it
  if (cloudLimit && file.size <= cloudLimit.maxSize && duration <= cloudLimit.maxDuration) {
    const estimatedCredits = Math.ceil(duration / 60) * 2;
    return { 
      valid: true, 
      processingMode: 'cloud', 
      cloudRequired: true,
      estimatedCredits,
    };
  }
  
  // File too large for any processing mode
  if (userTier === 'free') {
    return { 
      valid: false, 
      processingMode: 'rejected', 
      error: 'File too large for free tier. Upgrade to Starter for cloud processing of larger files.' 
    };
  }
  
  return { 
    valid: false, 
    processingMode: 'rejected', 
    error: `File exceeds maximum limits for ${userTier} tier.` 
  };
}
```

### 6.2 Cross-Origin Headers

```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'Cross-Origin-Opener-Policy',
    value: 'same-origin',
  },
  {
    key: 'Cross-Origin-Embedder-Policy',
    value: 'require-corp',
  },
  {
    key: 'X-Content-Type-Options',
    value: 'nosniff',
  },
  {
    key: 'X-Frame-Options',
    value: 'DENY',
  },
  {
    key: 'X-XSS-Protection',
    value: '1; mode=block',
  },
];
```

### 6.3 API Security

```typescript
// Rate limiting
const rateLimits = {
  '/api/projects': { windowMs: 60000, max: 30 },
  '/api/exports': { windowMs: 60000, max: 10 },
  '/api/credits/purchase': { windowMs: 60000, max: 5 },
};

// Input validation with Zod
import { z } from 'zod';

const createProjectSchema = z.object({
  name: z.string().min(1).max(100),
  sourceFilename: z.string().max(255),
  sourceDuration: z.number().positive().max(7200), // Max 2 hours
  sourceResolution: z.string().regex(/^\d+x\d+$/),
});
```

---

## 7. Performance Optimization

### 7.1 Web Worker Architecture

```typescript
// src/workers/video-processor.worker.ts

import { expose } from 'comlink';

const videoProcessor = {
  async loadFFmpeg(): Promise<void> {
    // Load FFmpeg in worker thread
  },
  
  async processVideo(
    inputBuffer: ArrayBuffer,
    operations: Operation[]
  ): Promise<ArrayBuffer> {
    // Heavy processing happens here, off main thread
  },
  
  async extractFrames(
    inputBuffer: ArrayBuffer,
    timestamps: number[]
  ): Promise<ImageData[]> {
    // Frame extraction for thumbnails
  },
};

expose(videoProcessor);

// Main thread usage
// src/lib/video/worker-client.ts
import { wrap } from 'comlink';

const worker = new Worker(
  new URL('../workers/video-processor.worker.ts', import.meta.url)
);
const videoProcessor = wrap<typeof videoProcessor>(worker);

// Now calls run in worker
await videoProcessor.processVideo(buffer, operations);
```

### 7.2 Memory Management

```typescript
// Aggressive cleanup for large files
class MemoryManager {
  private allocations: Map<string, ArrayBuffer> = new Map();
  
  allocate(id: string, size: number): ArrayBuffer {
    const buffer = new ArrayBuffer(size);
    this.allocations.set(id, buffer);
    return buffer;
  }
  
  release(id: string): void {
    this.allocations.delete(id);
    // Suggest garbage collection (not guaranteed)
    if (typeof gc !== 'undefined') gc();
  }
  
  releaseAll(): void {
    this.allocations.clear();
  }
  
  getUsage(): number {
    return Array.from(this.allocations.values())
      .reduce((sum, buf) => sum + buf.byteLength, 0);
  }
}

// Monitor memory pressure
if ('memory' in performance) {
  const checkMemory = () => {
    const { usedJSHeapSize, jsHeapSizeLimit } = (performance as any).memory;
    const usage = usedJSHeapSize / jsHeapSizeLimit;
    if (usage > 0.8) {
      console.warn('High memory usage, triggering cleanup');
      memoryManager.releaseAll();
    }
  };
  setInterval(checkMemory, 5000);
}
```

### 7.3 Progressive Loading

```typescript
// Load models on demand
const modelLoader = {
  whisper: null as WhisperModel | null,
  faceDetector: null as FaceDetector | null,
  
  async getWhisper(): Promise<WhisperModel> {
    if (!this.whisper) {
      this.whisper = await loadWhisperModel('base');
    }
    return this.whisper;
  },
  
  async getFaceDetector(): Promise<FaceDetector> {
    if (!this.faceDetector) {
      this.faceDetector = await loadFaceDetector();
    }
    return this.faceDetector;
  },
};

// Preload based on user intent
function preloadForCaptions(): void {
  modelLoader.getWhisper(); // Start loading in background
}

function preloadForReframe(): void {
  modelLoader.getFaceDetector();
}
```

---

## 8. Error Handling

### 8.1 Error Types

```typescript
// src/lib/errors.ts

class ClipFlowError extends Error {
  constructor(
    message: string,
    public code: string,
    public recoverable: boolean = true,
    public userMessage?: string
  ) {
    super(message);
    this.name = 'ClipFlowError';
  }
}

// Specific error types
class VideoLoadError extends ClipFlowError {
  constructor(message: string) {
    super(message, 'VIDEO_LOAD_ERROR', true, 'Failed to load video. Please try a different file.');
  }
}

class TranscriptionError extends ClipFlowError {
  constructor(message: string) {
    super(message, 'TRANSCRIPTION_ERROR', true, 'Transcription failed. Please try again.');
  }
}

class ExportError extends ClipFlowError {
  constructor(message: string) {
    super(message, 'EXPORT_ERROR', true, 'Export failed. Please try with lower quality settings.');
  }
}

class MemoryError extends ClipFlowError {
  constructor() {
    super(
      'Out of memory',
      'MEMORY_ERROR',
      true,
      'Your browser ran out of memory. Try a shorter video or close other tabs.'
    );
  }
}

class FillerDetectionError extends ClipFlowError {
  constructor(message: string) {
    super(message, 'FILLER_DETECTION_ERROR', true, 'Filler detection encountered an issue. You can still edit manually.');
  }
}

class ClipAnalysisError extends ClipFlowError {
  constructor(message: string) {
    super(message, 'CLIP_ANALYSIS_ERROR', true, 'Clip analysis encountered an issue. You can still trim manually.');
  }
}
```

### 8.2 Error Recovery

```typescript
// Automatic retry with exponential backoff
async function withRetry<T>(
  fn: () => Promise<T>,
  options: { maxRetries: number; baseDelay: number }
): Promise<T> {
  let lastError: Error;
  
  for (let attempt = 0; attempt < options.maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      
      if (!isRetryable(error)) throw error;
      
      const delay = options.baseDelay * Math.pow(2, attempt);
      await sleep(delay);
    }
  }
  
  throw lastError!;
}

function isRetryable(error: unknown): boolean {
  if (error instanceof ClipFlowError) {
    return error.recoverable;
  }
  return false;
}
```

---

## 9. Testing Strategy

### 9.1 Unit Tests

```typescript
// src/lib/captions/__tests__/caption-renderer.test.ts

describe('CaptionRenderer', () => {
  describe('generateASS', () => {
    it('should generate valid ASS format', () => {
      const transcript: Transcript = {
        text: 'Hello world',
        segments: [{ start: 0, end: 1, text: 'Hello world' }],
        words: [
          { word: 'Hello', start: 0, end: 0.5, confidence: 0.99 },
          { word: 'world', start: 0.5, end: 1, confidence: 0.98 },
        ],
      };
      
      const ass = renderer.generateASS(transcript, defaultStyle);
      
      expect(ass).toContain('[Script Info]');
      expect(ass).toContain('[V4+ Styles]');
      expect(ass).toContain('Dialogue:');
    });
  });
});
```

### 9.2 Integration Tests

```typescript
// src/lib/video/__tests__/pipeline.integration.test.ts

describe('VideoPipeline Integration', () => {
  it('should process video with captions and reframe', async () => {
    const testVideo = await loadTestVideo('sample-720p.mp4');
    
    const pipeline = new VideoPipeline();
    await pipeline.initialize();
    
    const result = await pipeline.process({
      inputFile: testVideo,
      operations: [
        { type: 'caption', params: { style: defaultStyle } },
        { type: 'reframe', params: { targetAspect: '9:16' } },
      ],
      outputFormat: { format: 'mp4', resolution: '720p' },
    });
    
    expect(result).toBeInstanceOf(Blob);
    expect(result.type).toBe('video/mp4');
  }, 60000); // 60s timeout for video processing
});
```

### 9.3 E2E Tests

```typescript
// e2e/export-flow.spec.ts

import { test, expect } from '@playwright/test';

test('complete export flow', async ({ page }) => {
  await page.goto('/');
  
  // Upload video
  const fileInput = page.locator('input[type="file"]');
  await fileInput.setInputFiles('test-fixtures/sample.mp4');
  
  // Wait for video to load
  await expect(page.locator('[data-testid="video-preview"]')).toBeVisible();
  
  // Enable captions
  await page.click('[data-testid="captions-toggle"]');
  await expect(page.locator('[data-testid="transcribing-indicator"]')).toBeVisible();
  await expect(page.locator('[data-testid="transcript-ready"]')).toBeVisible({ timeout: 60000 });
  
  // Start export
  await page.click('[data-testid="export-button"]');
  
  // Wait for download
  const [download] = await Promise.all([
    page.waitForEvent('download'),
    page.click('[data-testid="confirm-export"]'),
  ]);
  
  expect(download.suggestedFilename()).toMatch(/\.mp4$/);
});

test('transcript editing and filler removal flow', async ({ page }) => {
  await page.goto('/');

  const fileInput = page.locator('input[type="file"]');
  await fileInput.setInputFiles('test-fixtures/sample-with-fillers.mp4');
  await expect(page.locator('[data-testid="video-preview"]')).toBeVisible();

  // Wait for transcription
  await expect(page.locator('[data-testid="transcript-ready"]')).toBeVisible({ timeout: 60000 });

  // Verify transcript editor is visible
  await expect(page.locator('[data-testid="transcript-editor"]')).toBeVisible();

  // Run filler detection
  await page.click('[data-testid="detect-fillers-button"]');
  await expect(page.locator('[data-testid="filler-count"]')).toBeVisible();

  // Remove all fillers
  await page.click('[data-testid="remove-all-fillers"]');
  await expect(page.locator('[data-testid="fillers-removed-badge"]')).toBeVisible();
});

test('smart clip suggestions flow', async ({ page }) => {
  await page.goto('/');

  const fileInput = page.locator('input[type="file"]');
  await fileInput.setInputFiles('test-fixtures/sample-long.mp4');
  await expect(page.locator('[data-testid="video-preview"]')).toBeVisible();

  // Wait for analysis
  await expect(page.locator('[data-testid="clip-suggestions"]')).toBeVisible({ timeout: 90000 });

  // Click first suggestion
  await page.click('[data-testid="suggestion-0"]');

  // Verify trim points updated
  await expect(page.locator('[data-testid="trim-start"]')).not.toHaveText('0:00');
});
```

---

## 10. Deployment

### 10.1 Environment Variables

```bash
# .env.example

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=xxx
SUPABASE_SERVICE_ROLE_KEY=xxx

# Stripe
STRIPE_SECRET_KEY=sk_xxx
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Analytics
NEXT_PUBLIC_POSTHOG_KEY=xxx
SENTRY_DSN=xxx

# Feature flags
NEXT_PUBLIC_ENABLE_SERVER_AI=false
```

### 10.2 Vercel Configuration

```json
// vercel.json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "Cross-Origin-Opener-Policy", "value": "same-origin" },
        { "key": "Cross-Origin-Embedder-Policy", "value": "require-corp" }
      ]
    }
  ],
  "functions": {
    "api/**/*.ts": {
      "maxDuration": 30
    }
  }
}
```

### 10.3 CDN Configuration for Models

```typescript
// Model files should be served from CDN with long cache
// vercel.json
{
  "headers": [
    {
      "source": "/models/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ]
}
```

---

## 11. Monitoring & Observability

### 11.1 Key Metrics to Track

```typescript
// src/lib/analytics/metrics.ts

const metrics = {
  // Performance
  'video.load.duration': 'histogram',
  'transcription.duration': 'histogram',
  'export.duration': 'histogram',
  'ffmpeg.init.duration': 'histogram',
  'filler.detection.duration': 'histogram',
  'clip.analysis.duration': 'histogram',
  
  // Success/Failure
  'export.success': 'counter',
  'export.failure': 'counter',
  'transcription.success': 'counter',
  'transcription.failure': 'counter',
  
  // Feature Adoption
  'transcript.editing.used': 'counter',
  'filler.removal.used': 'counter',
  'filler.removal.words_removed': 'counter',
  'text.overlay.added': 'counter',
  'brand.template.created': 'counter',
  'brand.template.applied': 'counter',
  'clip.suggestion.shown': 'counter',
  'clip.suggestion.accepted': 'counter',
  'clip.suggestion.rejected': 'counter',
  
  // Usage
  'video.processed.minutes': 'counter',
  'credits.used': 'counter',
  
  // Browser
  'browser.memory.usage': 'gauge',
  'browser.webgpu.available': 'gauge',
};

function trackMetric(name: string, value: number, tags?: Record<string, string>): void {
  posthog.capture(name, { value, ...tags });
}
```

### 11.2 Error Tracking

```typescript
// src/lib/analytics/sentry.ts

import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 0.1,
  beforeSend(event) {
    // Scrub sensitive data
    if (event.request?.data) {
      delete event.request.data;
    }
    return event;
  },
});

// Capture with context
function captureError(error: Error, context: Record<string, any>): void {
  Sentry.withScope((scope) => {
    scope.setExtras(context);
    Sentry.captureException(error);
  });
}
```

---

## 12. Future Considerations

### 12.1 Server-Side AI Integration (Phase 2)

```typescript
// Future: Server-side processing for paid features
interface AIService {
  // Clip detection using ML
  detectClips(videoUrl: string): Promise<ClipSuggestion[]>;
  
  // B-roll generation
  generateBRoll(prompt: string, duration: number): Promise<string>;
  
  // Voice enhancement
  enhanceAudio(audioUrl: string): Promise<string>;
}

// Implementation would use Replicate/Modal for GPU inference
```

### 12.2 Mobile Support (Phase 3)

- React Native app with shared business logic
- Native video processing for better performance
- Offline-first architecture with sync

### 12.3 Collaboration Features (Phase 3)

- Real-time collaborative editing (CRDT-based)
- Comments and annotations
- Version history

### 12.4 Community Brand Template Gallery (Phase 3)

- Public template sharing with attribution
- Featured/trending templates curated by team
- Template categories (gaming, podcast, education, business)

---

*Document maintained by: Engineering Team*  
*Last updated: February 24, 2026*
