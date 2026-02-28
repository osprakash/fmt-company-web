# Whisper Model Loading Strategy

**Purpose:** Optimize the ~150MB Whisper model download for best user experience  
**Goal:** Users should never feel blocked by the model download

---

## 1. Model Selection

### 1.1 Recommended Model: `whisper-base` (Quantized)

| Model | Original Size | Quantized (Q5) | Quality | Recommendation |
|-------|---------------|----------------|---------|----------------|
| `tiny` | 39MB | ~20MB | Low accuracy, misses words | ❌ Not recommended |
| `base` | 74MB | ~40MB | Good for short-form content | ✅ **Default choice** |
| `small` | 244MB | ~130MB | Better accuracy | 🔄 Optional upgrade |
| `medium` | 769MB | ~400MB | High accuracy | ❌ Too large for web |

**Why `base` quantized:**
- 40MB is ~73% smaller than the original 150MB estimate
- Downloads in ~8-15 seconds on average connections
- Accuracy is sufficient for short-form video (< 15 min)
- Quantization has minimal impact on transcription quality

### 1.2 Model Source

Use pre-quantized GGML models from Hugging Face:
```
https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin
```

For multi-language support:
```
https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.bin
```

---

## 2. Loading Strategy: Progressive Background Download

### 2.1 Core Principle

**Never block the user.** The app should be fully usable while the model downloads.

### 2.2 User Flow with Model States

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           USER LANDS ON SITE                            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  CHECK: Is model cached in IndexedDB?                                   │
│  ├── YES → Ready to transcribe immediately                              │
│  └── NO  → Start background download, show UI anyway                    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
        ┌───────────────────┐           ┌───────────────────┐
        │  MODEL CACHED     │           │  MODEL DOWNLOADING │
        │  ─────────────    │           │  ──────────────────│
        │  • Full UI access │           │  • Full UI access  │
        │  • Instant        │           │  • Show progress   │
        │    transcription  │           │    indicator       │
        └───────────────────┘           │  • User can browse │
                                        │    demo, explore   │
                                        └───────────────────┘
                                                  │
                                                  ▼
                                        ┌───────────────────┐
                                        │  USER DROPS VIDEO │
                                        │  ──────────────── │
                                        │  Model ready?     │
                                        │  ├── YES → Start  │
                                        │  │   transcription│
                                        │  └── NO → Show    │
                                        │      "Preparing   │
                                        │       AI..." with │
                                        │       progress    │
                                        └───────────────────┘
```

---

## 3. Implementation Details

### 3.1 Model State Machine

```typescript
type ModelState = 
  | { status: 'checking' }                    // Checking IndexedDB
  | { status: 'cached'; version: string }     // Ready to use
  | { status: 'downloading'; progress: number; bytesLoaded: number; totalBytes: number }
  | { status: 'ready' }                       // Just finished downloading
  | { status: 'error'; message: string; retryable: boolean }

interface ModelManager {
  state: ModelState;
  initialize(): Promise<void>;
  getModel(): Promise<ArrayBuffer>;
  cancelDownload(): void;
  retryDownload(): void;
}
```

### 3.2 Storage Strategy

```typescript
// Store in IndexedDB for persistence across sessions
const MODEL_DB_NAME = 'flowcut-models';
const MODEL_STORE_NAME = 'whisper';
const MODEL_VERSION_KEY = 'whisper-base-q5-v1';

// Check cache first
async function getModelFromCache(): Promise<ArrayBuffer | null> {
  const db = await openDB(MODEL_DB_NAME);
  return db.get(MODEL_STORE_NAME, MODEL_VERSION_KEY);
}

// Save after download
async function cacheModel(data: ArrayBuffer): Promise<void> {
  const db = await openDB(MODEL_DB_NAME);
  await db.put(MODEL_STORE_NAME, data, MODEL_VERSION_KEY);
}
```

### 3.3 Download with Progress and Resume

```typescript
async function downloadModel(
  onProgress: (loaded: number, total: number) => void
): Promise<ArrayBuffer> {
  const MODEL_URL = 'https://cdn.flowcut.app/models/ggml-base.bin';
  
  // Check for partial download in IndexedDB
  const partial = await getPartialDownload();
  const startByte = partial?.byteLength ?? 0;
  
  const response = await fetch(MODEL_URL, {
    headers: startByte > 0 ? { 'Range': `bytes=${startByte}-` } : {}
  });
  
  const contentLength = parseInt(response.headers.get('content-length') || '0');
  const totalSize = startByte + contentLength;
  
  const reader = response.body!.getReader();
  const chunks: Uint8Array[] = partial ? [new Uint8Array(partial)] : [];
  let loadedBytes = startByte;
  
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    
    chunks.push(value);
    loadedBytes += value.length;
    onProgress(loadedBytes, totalSize);
    
    // Save progress every 1MB for resume capability
    if (loadedBytes % (1024 * 1024) < value.length) {
      await savePartialDownload(concatenateChunks(chunks));
    }
  }
  
  const fullModel = concatenateChunks(chunks);
  await clearPartialDownload();
  await cacheModel(fullModel);
  
  return fullModel;
}
```

---

## 4. UI Components

### 4.1 Global Model Status Indicator

Show a subtle, non-blocking indicator in the header/toolbar:

```
┌────────────────────────────────────────────────────────────────┐
│  FlowCut                              [AI: Downloading 67%] 🔄 │
└────────────────────────────────────────────────────────────────┘

After complete:
┌────────────────────────────────────────────────────────────────┐
│  FlowCut                                        [AI Ready] ✓   │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 First-Time Welcome Modal (Optional)

Only show if model isn't cached AND user hasn't dismissed before:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                           [×]   │
│                     🎙️ Setting up AI Transcription              │
│                                                                 │
│     FlowCut uses on-device AI for instant transcription.        │
│     We're downloading the speech model (~40MB) now.             │
│                                                                 │
│     ████████████████░░░░░░░░░░░░░░  45%                        │
│     18 MB / 40 MB • ~12 seconds remaining                       │
│                                                                 │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  ✓ Your videos never leave your device              │    │
│     │  ✓ Works offline after this download                │    │
│     │  ✓ No account needed to try                         │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│                    [ Explore while downloading ]                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Video Drop Zone States

**State: Model Ready**
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│                         📁                                      │
│                                                                 │
│              Drop your video here to add captions               │
│                   MP4, WebM, MOV up to 200MB                    │
│                                                                 │
│                      [ Browse files ]                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**State: Model Downloading (User drops video)**
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│                         🎙️                                      │
│                                                                 │
│                  Preparing AI transcription...                  │
│                                                                 │
│              ████████████████░░░░░░░░  67% (27/40 MB)          │
│                     ~8 seconds remaining                        │
│                                                                 │
│              Your video is ready. Transcription will            │
│              start automatically when AI is loaded.             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 Error States

**Download Failed (Retryable)**
```
┌─────────────────────────────────────────────────────────────────┐
│                         ⚠️                                       │
│                                                                 │
│              AI model download interrupted                      │
│              Check your internet connection                     │
│                                                                 │
│              Progress saved: 23 MB / 40 MB                      │
│                                                                 │
│                    [ Retry Download ]                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Browser Not Supported**
```
┌─────────────────────────────────────────────────────────────────┐
│                         🌐                                       │
│                                                                 │
│              Your browser doesn't support AI transcription      │
│                                                                 │
│              Please use one of these browsers:                  │
│              • Chrome 94+                                       │
│              • Firefox 88+                                      │
│              • Edge 94+                                         │
│                                                                 │
│                    [ Copy link to open in Chrome ]              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Performance Optimizations

### 5.1 CDN Configuration

Host model on a CDN with:
- **Compression**: Serve with `Content-Encoding: gzip` (reduces ~40MB to ~35MB)
- **Caching**: `Cache-Control: public, max-age=31536000, immutable`
- **Range requests**: Enable for resume support
- **Multiple regions**: Use Cloudflare/Vercel Edge for global distribution

### 5.2 Preloading Hints

Add to HTML `<head>`:
```html
<!-- Preconnect to CDN -->
<link rel="preconnect" href="https://cdn.flowcut.app" crossorigin>

<!-- Optional: Preload for returning users (browser will use cache) -->
<link rel="prefetch" href="https://cdn.flowcut.app/models/ggml-base.bin" as="fetch" crossorigin>
```

### 5.3 Service Worker Caching

```typescript
// sw.js
const MODEL_CACHE = 'flowcut-model-v1';
const MODEL_URL = 'https://cdn.flowcut.app/models/ggml-base.bin';

self.addEventListener('fetch', (event) => {
  if (event.request.url === MODEL_URL) {
    event.respondWith(
      caches.match(event.request).then((cached) => {
        return cached || fetch(event.request).then((response) => {
          const clone = response.clone();
          caches.open(MODEL_CACHE).then((cache) => cache.put(event.request, clone));
          return response;
        });
      })
    );
  }
});
```

---

## 6. Offline Support

### 6.1 Behavior After Initial Download

Once the model is cached:
- ✅ Transcription works fully offline
- ✅ No network requests needed for AI
- ✅ App can be used on airplane mode

### 6.2 Offline Indicator

```
┌────────────────────────────────────────────────────────────────┐
│  FlowCut                    [Offline Mode - AI Ready] ✓        │
└────────────────────────────────────────────────────────────────┘
```

---

## 7. Analytics Events

Track these events to monitor model loading health:

| Event | Properties | Purpose |
|-------|------------|---------|
| `model_check_start` | `cached: boolean` | Track cache hit rate |
| `model_download_start` | `resumed: boolean, startByte: number` | Track fresh vs resumed |
| `model_download_progress` | `percent: number` | Monitor drop-off points |
| `model_download_complete` | `duration_ms: number, size_bytes: number` | Track download times |
| `model_download_error` | `error: string, bytes_loaded: number` | Debug failures |
| `model_load_complete` | `load_time_ms: number` | Track WASM initialization |

### 7.1 Success Targets

| Metric | Target |
|--------|--------|
| Cache hit rate (returning users) | > 95% |
| Download completion rate | > 90% |
| Average download time | < 20 seconds |
| Model load time (from cache) | < 2 seconds |

---

## 8. Testing Checklist

### 8.1 Happy Path
- [ ] First visit: Model downloads in background
- [ ] User can explore UI during download
- [ ] Video dropped during download queues correctly
- [ ] Transcription starts immediately after model ready
- [ ] Second visit: Model loads from cache instantly

### 8.2 Edge Cases
- [ ] Network disconnects mid-download → Shows retry option
- [ ] User closes tab mid-download → Resumes on next visit
- [ ] Very slow connection (< 1 Mbps) → Shows realistic time estimate
- [ ] IndexedDB full → Falls back to memory (with warning)
- [ ] Private/incognito mode → Works but re-downloads each session

### 8.3 Browser Compatibility
- [ ] Chrome 94+ → Full support
- [ ] Firefox 88+ → Full support
- [ ] Edge 94+ → Full support
- [ ] Safari 15.2+ → Test WebAssembly performance
- [ ] Mobile Chrome → Test memory constraints

---

## 9. Future Improvements (Post-MVP)

| Improvement | Benefit | Complexity |
|-------------|---------|------------|
| Model quality selector | Let users choose tiny/base/small | Low |
| Background sync API | Download continues even if tab closed | Medium |
| Delta updates | Only download changed model parts | High |
| WebGPU acceleration | Faster transcription on supported browsers | Medium |
| Streaming transcription | Show words as they're recognized | High |

---

## 10. Summary

### Key Decisions

1. **Use `whisper-base` quantized** (~40MB instead of 150MB)
2. **Never block the UI** — download happens in background
3. **Cache in IndexedDB** — instant load on return visits
4. **Support resume** — partial downloads are saved
5. **Clear progress UI** — users always know what's happening

### Expected User Experience

| Scenario | Experience |
|----------|------------|
| First visit | See demo, explore UI while 40MB downloads (~15 sec) |
| Drop video (model ready) | Transcription starts in < 1 second |
| Drop video (model loading) | "Preparing AI..." then auto-starts |
| Return visit | Instant — model loads from cache in < 2 sec |
| Offline (after first use) | Full transcription works offline |

---

*The best download is the one users don't notice.*
