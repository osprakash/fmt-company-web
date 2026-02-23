# Technical Specification Document
## ClipFlow.ai - Browser-First Video Repurposing Platform

**Version:** 1.0  
**Date:** February 23, 2026  
**Status:** Draft

---

## 1. System Overview

### 1.1 Architecture Philosophy
ClipFlow.ai uses a **browser-first architecture** where computationally intensive video processing happens on the client device. The server handles only lightweight operations: authentication, metadata storage, credit management, and optional AI features.

### 1.2 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Browser)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   React     │  │  FFmpeg     │  │  Whisper    │  │  MediaPipe  │        │
│  │   Frontend  │  │  WASM       │  │  Web        │  │  Face Det.  │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │                │
│         └────────────────┴────────────────┴────────────────┘                │
│                                    │                                         │
│                          ┌─────────▼─────────┐                              │
│                          │   Video Pipeline   │                              │
│                          │   Orchestrator     │                              │
│                          └─────────┬─────────┘                              │
│                                    │                                         │
│  ┌─────────────┐  ┌─────────────┐  │  ┌─────────────┐  ┌─────────────┐     │
│  │  IndexedDB  │  │  WebCodecs  │◄─┴─►│   Canvas    │  │  Web Worker │     │
│  │  (Cache)    │  │  (Decode)   │     │  (Preview)  │  │  (Process)  │     │
│  └─────────────┘  └─────────────┘     └─────────────┘  └─────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       │ HTTPS (Auth, Metadata, Credits)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SERVER (Lightweight)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Next.js   │  │  Supabase   │  │   Stripe    │  │  AI Service │        │
│  │   API       │  │  (Auth/DB)  │  │  (Payments) │  │  (Future)   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
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
  type: 'trim' | 'caption' | 'reframe';
  params: TrimParams | CaptionParams | ReframeParams;
}

interface PipelineProgress {
  stage: 'loading' | 'transcribing' | 'processing' | 'encoding' | 'complete';
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

-- Indexes
CREATE INDEX idx_projects_user ON projects(user_id);
CREATE INDEX idx_exports_user ON exports(user_id);
CREATE INDEX idx_exports_project ON exports(project_id);
CREATE INDEX idx_credit_transactions_user ON credit_transactions(user_id);
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
  
  // Timeline
  trimStart: number;
  trimEnd: number;
  currentTime: number;
  isPlaying: boolean;
  
  // Transcript
  transcript: Transcript | null;
  isTranscribing: boolean;
  transcriptEdits: Map<string, string>; // word id -> edited text
  
  // Captions
  captionStyle: CaptionStyle;
  captionsEnabled: boolean;
  
  // Reframe
  reframeEnabled: boolean;
  targetAspectRatio: AspectRatio;
  cropRegion: CropRegion | null;
  faceTracking: FaceTrack[] | null;
  
  // Export
  exportSettings: ExportSettings;
  exportProgress: ExportProgress | null;
  
  // Actions
  setSourceFile: (file: File) => Promise<void>;
  setTrimRange: (start: number, end: number) => void;
  updateCaptionStyle: (style: Partial<CaptionStyle>) => void;
  setTargetAspectRatio: (ratio: AspectRatio) => void;
  startExport: () => Promise<Blob>;
  // ... more actions
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

class ClipFlowDB extends Dexie {
  videos!: Table<CachedVideo>;
  models!: Table<CachedModel>;
  drafts!: Table<ProjectDraft>;

  constructor() {
    super('clipflow');
    this.version(1).stores({
      videos: 'id, filename, lastAccessedAt',
      models: 'id, name, version',
      drafts: 'id, projectId, savedAt'
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
// File validation
const ALLOWED_VIDEO_TYPES = ['video/mp4', 'video/webm', 'video/quicktime'];
const MAX_FILE_SIZE_FREE = 500 * 1024 * 1024; // 500MB
const MAX_FILE_SIZE_PAID = 2 * 1024 * 1024 * 1024; // 2GB

function validateVideoFile(file: File, userTier: string): ValidationResult {
  if (!ALLOWED_VIDEO_TYPES.includes(file.type)) {
    return { valid: false, error: 'Unsupported video format' };
  }
  
  const maxSize = userTier === 'free' ? MAX_FILE_SIZE_FREE : MAX_FILE_SIZE_PAID;
  if (file.size > maxSize) {
    return { valid: false, error: `File too large. Max: ${maxSize / 1024 / 1024}MB` };
  }
  
  return { valid: true };
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
  
  // Success/Failure
  'export.success': 'counter',
  'export.failure': 'counter',
  'transcription.success': 'counter',
  'transcription.failure': 'counter',
  
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

---

*Document maintained by: Engineering Team*  
*Last updated: February 23, 2026*
