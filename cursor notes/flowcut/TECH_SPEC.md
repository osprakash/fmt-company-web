# Technical Specification
## FlowCut - Browser-Based Caption Tool

**Version:** 1.0  
**Date:** February 28, 2026  
**Status:** MVP Scope

---

## 1. System Overview

### 1.1 Architecture
FlowCut is a **browser-first** video caption tool. All processing happens client-side using WebCodecs, FFmpeg.wasm, and Whisper Web.

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT (Browser)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  React    │  │  FFmpeg   │  │  Whisper  │  │  Canvas   │    │
│  │  Frontend │  │  WASM     │  │  Web      │  │  Renderer │    │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘    │
│        │              │              │              │            │
│        └──────────────┴──────────────┴──────────────┘            │
│                              │                                    │
│                    ┌─────────▼─────────┐                         │
│                    │  Video Pipeline   │                         │
│                    └─────────┬─────────┘                         │
│                              │                                    │
│  ┌────────────┐  ┌──────────▼───────┐  ┌────────────┐           │
│  │ IndexedDB  │  │ Caption Renderer │  │ Timeline   │           │
│  │ (Cache)    │  │                  │  │ Editor     │           │
│  └────────────┘  └──────────────────┘  └────────────┘           │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                           SERVER                                  │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Next.js   │  │  Supabase   │  │   Stripe    │              │
│  │   API       │  │  (Auth/DB)  │  │  (Payments) │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Why Browser-First
- **Instant start**: No upload wait
- **Privacy**: Video never leaves device
- **Cost**: Zero server compute costs for video processing

---

## 2. Technology Stack

### 2.1 Frontend

| Layer | Technology | Purpose |
|-------|------------|---------|
| Framework | Next.js 14+ (App Router) | React with SSR |
| Language | TypeScript | Type safety |
| Styling | Tailwind CSS + shadcn/ui | UI components |
| State | Zustand | Lightweight state |
| Video Processing | FFmpeg.wasm 0.12+ | Encoding/decoding |
| Transcription | @remotion/whisper-web | Speech-to-text |
| Storage | IndexedDB (Dexie.js) | Model caching |

### 2.2 Backend

| Layer | Technology | Purpose |
|-------|------------|---------|
| API | Next.js API Routes | Serverless functions |
| Database | Supabase (PostgreSQL) | User data |
| Auth | Supabase Auth | Email + Google OAuth |
| Payments | Stripe | Subscriptions |
| Hosting | Vercel | Deployment |

---

## 3. Core Modules

### 3.1 Video Pipeline

```typescript
// src/lib/video/pipeline.ts

interface PipelineConfig {
  inputFile: File;
  transcript: Transcript;
  captionStyle: CaptionStyle;
  trimStart: number;
  trimEnd: number;
  outputQuality: '720p' | '1080p';
  addWatermark: boolean;
  onProgress: (progress: number) => void;
}

class VideoPipeline {
  private ffmpeg: FFmpeg;
  
  async initialize(): Promise<void>;
  async process(config: PipelineConfig): Promise<Blob>;
  async abort(): void;
}
```

### 3.2 FFmpeg Service

```typescript
// src/lib/video/ffmpeg-service.ts

interface FFmpegService {
  load(): Promise<void>;
  isLoaded(): boolean;
  
  // Core operations
  trim(input: Uint8Array, start: number, end: number): Promise<Uint8Array>;
  addSubtitles(input: Uint8Array, assContent: string): Promise<Uint8Array>;
  addWatermark(input: Uint8Array, text: string): Promise<Uint8Array>;
  
  // Export
  encode(input: Uint8Array, options: EncodeOptions): Promise<Uint8Array>;
  
  // Utilities
  getMetadata(input: Uint8Array): Promise<VideoMetadata>;
  extractAudio(input: Uint8Array): Promise<Uint8Array>;
}

interface EncodeOptions {
  resolution: '720p' | '1080p';
  quality: 'draft' | 'standard' | 'high';
}
```

**FFmpeg Commands**:

```bash
# Trim video
-ss {start} -to {end} -i input.mp4 -c copy output.mp4

# Add ASS subtitles (burned in)
-i input.mp4 -vf "ass=subtitles.ass" -c:a copy output.mp4

# Add watermark
-i input.mp4 -vf "drawtext=text='Made with FlowCut':fontsize=24:fontcolor=white@0.5:x=w-tw-10:y=h-th-10" output.mp4

# Export 720p
-i input.mp4 -vf scale=-1:720 -c:v libx264 -preset fast -crf 23 -c:a aac output.mp4

# Export 1080p
-i input.mp4 -vf scale=-1:1080 -c:v libx264 -preset medium -crf 20 -c:a aac output.mp4
```

### 3.3 Whisper Transcription

```typescript
// src/lib/transcription/whisper-service.ts

interface WhisperService {
  loadModel(size: 'tiny' | 'base'): Promise<void>;
  isModelLoaded(): boolean;
  
  transcribe(
    audioData: Float32Array,
    options: TranscribeOptions
  ): Promise<Transcript>;
}

interface TranscribeOptions {
  language?: string; // Auto-detect if not specified
  wordTimestamps: boolean;
  onProgress?: (progress: number) => void;
}

interface Transcript {
  text: string;
  language: string;
  segments: TranscriptSegment[];
  words: TranscriptWord[];
}

interface TranscriptWord {
  word: string;
  start: number;
  end: number;
  confidence: number;
}
```

**Model Selection**:

| Model | Size | Speed | Accuracy | Use Case |
|-------|------|-------|----------|----------|
| tiny | ~75MB | 10x realtime | ~88% | Quick preview |
| base | ~150MB | 5x realtime | ~94% | Default |

### 3.4 Caption Renderer

```typescript
// src/lib/captions/caption-renderer.ts

interface CaptionStyle {
  preset: 'classic' | 'karaoke' | 'bounce' | 'glow' | 'minimal';
  fontFamily: string;
  fontSize: number;
  fontWeight: number;
  color: string;
  backgroundColor?: string;
  strokeColor?: string;
  strokeWidth?: number;
  position: 'top' | 'center' | 'bottom';
}

interface CaptionRenderer {
  // Generate ASS subtitle file for FFmpeg export
  generateASS(transcript: Transcript, style: CaptionStyle): string;
  
  // Render to canvas for real-time preview
  renderFrame(
    ctx: CanvasRenderingContext2D,
    transcript: Transcript,
    currentTime: number,
    style: CaptionStyle
  ): void;
}
```

**Caption Style Presets**:

```typescript
const CAPTION_PRESETS = {
  classic: {
    fontFamily: 'Arial',
    fontSize: 48,
    color: '#FFFFFF',
    backgroundColor: 'rgba(0,0,0,0.7)',
    position: 'bottom',
  },
  karaoke: {
    fontFamily: 'Montserrat',
    fontSize: 56,
    color: '#FFFFFF',
    highlightColor: '#FFFF00',
    strokeColor: '#000000',
    strokeWidth: 3,
    position: 'center',
  },
  bounce: {
    fontFamily: 'Poppins',
    fontSize: 52,
    color: '#FFFFFF',
    strokeColor: '#000000',
    strokeWidth: 2,
    position: 'bottom',
  },
  glow: {
    fontFamily: 'Inter',
    fontSize: 48,
    color: '#FFFFFF',
    glowColor: '#00FFFF',
    position: 'bottom',
  },
  minimal: {
    fontFamily: 'Helvetica',
    fontSize: 40,
    color: '#FFFFFF',
    position: 'bottom',
  },
};
```

---

## 4. Data Models

### 4.1 Database Schema (Supabase)

```sql
-- Users (managed by Supabase Auth)
CREATE TABLE user_profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id),
  tier TEXT DEFAULT 'free' CHECK (tier IN ('free', 'pro')),
  exports_this_month INTEGER DEFAULT 0,
  exports_reset_at TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Export history
CREATE TABLE exports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES user_profiles(id) ON DELETE CASCADE,
  resolution TEXT NOT NULL,
  duration_seconds FLOAT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_exports_user ON exports(user_id);
CREATE INDEX idx_exports_created ON exports(created_at);
```

### 4.2 Client State (Zustand)

```typescript
// src/store/editor-store.ts

interface EditorState {
  // Video source
  sourceFile: File | null;
  sourceMetadata: VideoMetadata | null;
  
  // Timeline
  trimStart: number;
  trimEnd: number;
  currentTime: number;
  isPlaying: boolean;
  
  // Transcript
  transcript: Transcript | null;
  isTranscribing: boolean;
  
  // Captions
  captionStyle: CaptionStyle;
  captionsEnabled: boolean;
  
  // Export
  exportProgress: number | null;
  
  // Actions
  setSourceFile: (file: File) => Promise<void>;
  setTrimRange: (start: number, end: number) => void;
  updateCaptionStyle: (style: Partial<CaptionStyle>) => void;
  startExport: (options: ExportOptions) => Promise<Blob>;
}
```

### 4.3 IndexedDB Schema (Dexie.js)

```typescript
// src/lib/storage/db.ts

import Dexie, { Table } from 'dexie';

interface CachedModel {
  id: string;
  name: string;
  data: ArrayBuffer;
  downloadedAt: Date;
}

class FlowCutDB extends Dexie {
  models!: Table<CachedModel>;

  constructor() {
    super('flowcut');
    this.version(1).stores({
      models: 'id, name'
    });
  }
}

export const db = new FlowCutDB();
```

---

## 5. API Endpoints

```typescript
// User
GET    /api/user/profile
PATCH  /api/user/profile

// Exports (for tracking limits)
POST   /api/exports
GET    /api/exports/remaining

// Payments
POST   /api/subscription/checkout
POST   /api/subscription/portal
POST   /api/webhooks/stripe
```

---

## 6. Browser Requirements

### 6.1 Minimum Requirements
- Chrome 94+, Firefox 88+, Edge 94+
- Safari 15.2+ (limited support)
- 4GB RAM minimum, 8GB recommended
- ~150MB initial download (Whisper model)

### 6.2 Required Headers

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

### 6.3 File Limits

| Tier | Max File Size | Max Duration |
|------|---------------|--------------|
| Free | 200MB | 10 min |
| Pro | 300MB | 15 min |

---

## 7. Export Pipeline

### 7.1 Export Flow

```
1. Validate export limits
2. Generate ASS subtitle file from transcript + style
3. Run FFmpeg:
   - Trim to selected range
   - Burn in subtitles
   - Add watermark (if free tier)
   - Encode to target resolution
4. Return Blob for download
5. Track export in database
```

### 7.2 FFmpeg Filter Chain

```bash
# Free tier (720p, watermark)
-i input.mp4 \
  -ss {start} -to {end} \
  -vf "ass=subs.ass,scale=-1:720,drawtext=text='Made with FlowCut':fontsize=20:fontcolor=white@0.5:x=w-tw-10:y=h-th-10" \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k \
  output.mp4

# Pro tier (1080p, no watermark)
-i input.mp4 \
  -ss {start} -to {end} \
  -vf "ass=subs.ass,scale=-1:1080" \
  -c:v libx264 -preset medium -crf 20 \
  -c:a aac -b:a 192k \
  output.mp4
```

---

## 8. Error Handling

```typescript
// src/lib/errors.ts

class FlowCutError extends Error {
  constructor(
    message: string,
    public code: string,
    public userMessage: string
  ) {
    super(message);
  }
}

class VideoLoadError extends FlowCutError {
  constructor(message: string) {
    super(message, 'VIDEO_LOAD_ERROR', 'Failed to load video. Try a different file.');
  }
}

class TranscriptionError extends FlowCutError {
  constructor(message: string) {
    super(message, 'TRANSCRIPTION_ERROR', 'Transcription failed. Please try again.');
  }
}

class ExportError extends FlowCutError {
  constructor(message: string) {
    super(message, 'EXPORT_ERROR', 'Export failed. Try a shorter video.');
  }
}

class ExportLimitError extends FlowCutError {
  constructor() {
    super('Export limit reached', 'EXPORT_LIMIT', 'You\'ve used all 3 free exports this month. Upgrade to Pro for unlimited exports.');
  }
}
```

---

## 9. Performance Targets

| Operation | Target | Notes |
|-----------|--------|-------|
| Video load | <3s | For 200MB file |
| Transcription | <30s/min | Using Whisper base |
| Export (720p) | <2min/min | FFmpeg.wasm |
| Export (1080p) | <3min/min | FFmpeg.wasm |

---

## 10. Development Checklist

### Week 1: Core Transcription
- [ ] Project setup (Next.js + Tailwind + shadcn)
- [ ] Video upload component
- [ ] Whisper Web integration
- [ ] Basic transcript display

### Week 2: Caption Styling + Export
- [ ] Caption style presets (5 styles)
- [ ] Style customization UI
- [ ] Canvas preview renderer
- [ ] FFmpeg.wasm export with captions

### Week 3: Trimming + Timeline
- [ ] Timeline component
- [ ] Waveform visualization
- [ ] Trim handles
- [ ] Export trimmed segment

### Week 4: Payments + Polish
- [ ] Supabase auth
- [ ] Stripe integration
- [ ] Export limits
- [ ] Watermark for free tier
- [ ] Landing page
- [ ] Bug fixes

---

*Document maintained by: Engineering*  
*Last updated: February 28, 2026*
