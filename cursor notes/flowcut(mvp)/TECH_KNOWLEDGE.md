# Technical Knowledge Document
## Understanding Video Captions in the Browser

**Version:** 1.0  
**Date:** February 28, 2026  
**Purpose:** Reference for understanding how FlowCut's caption features work

---

## Table of Contents

1. [Video Fundamentals](#1-video-fundamentals)
2. [Browser Video APIs](#2-browser-video-apis)
3. [FFmpeg.wasm](#3-ffmpegwasm)
4. [Speech-to-Text with Whisper](#4-speech-to-text-with-whisper)
5. [Caption Rendering](#5-caption-rendering)
6. [Video Encoding](#6-video-encoding)
7. [Browser Limitations](#7-browser-limitations)
8. [Glossary](#8-glossary)

---

## 1. Video Fundamentals

### 1.1 What is a Video File?

A video file is a **container** holding multiple streams:

```
┌─────────────────────────────────────────────────────────────┐
│                    VIDEO CONTAINER (MP4)                     │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │  VIDEO STREAM (H.264 encoded frames)                 │   │
│  │  - Resolution: 1920x1080                             │   │
│  │  - Frame rate: 30 fps                                │   │
│  │  - Bitrate: 5 Mbps                                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  AUDIO STREAM (AAC encoded)                          │   │
│  │  - Sample rate: 48000 Hz                             │   │
│  │  - Channels: Stereo                                  │   │
│  │  - Bitrate: 128 kbps                                 │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Key Terms:**

| Term | Definition |
|------|------------|
| **Container** | File format (MP4, WebM, MOV) |
| **Codec** | Compression algorithm (H.264, VP9) |
| **Bitrate** | Data per second (higher = better quality) |
| **Frame Rate** | Frames per second (24, 30, 60 fps) |
| **Resolution** | Pixel dimensions (1920x1080 = 1080p) |

### 1.2 How Video Compression Works

Video files use compression to reduce size:

- **Uncompressed 1-min 1080p**: ~10 GB
- **H.264 compressed**: ~50 MB (200x smaller)

**Compression types:**

1. **I-frames (Keyframes)**: Complete images, can seek to these
2. **P-frames**: Store differences from previous frame
3. **B-frames**: Store differences from both previous and next

```
I-frame    P-frame    P-frame    P-frame    I-frame
[Full]  →  [Diff]  →  [Diff]  →  [Diff]  →  [Full]
  ↑                                            ↑
Keyframe                                   Keyframe
```

**Why this matters:** You can only seek accurately to keyframes. Cutting at non-keyframe positions requires re-encoding.

---

## 2. Browser Video APIs

### 2.1 HTML5 Video Element

Basic video playback:

```javascript
const video = document.getElementById('player');

video.play();
video.pause();
video.currentTime = 30; // Seek to 30 seconds

console.log(video.duration);    // Total length
console.log(video.videoWidth);  // Native width
```

### 2.2 Canvas API

Draw video frames for manipulation:

```javascript
const video = document.getElementById('video');
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

function drawFrame() {
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
  requestAnimationFrame(drawFrame);
}
```

**Use cases:**
- Real-time caption preview
- Frame extraction for thumbnails
- Overlaying graphics

---

## 3. FFmpeg.wasm

### 3.1 What is FFmpeg?

FFmpeg is the standard tool for video processing. **FFmpeg.wasm** compiles it to WebAssembly for browser use.

### 3.2 Setup

```javascript
import { FFmpeg } from '@ffmpeg/ffmpeg';
import { fetchFile, toBlobURL } from '@ffmpeg/util';

const ffmpeg = new FFmpeg();

await ffmpeg.load({
  coreURL: await toBlobURL('/ffmpeg-core.js', 'text/javascript'),
  wasmURL: await toBlobURL('/ffmpeg-core.wasm', 'application/wasm'),
});

ffmpeg.on('progress', ({ progress }) => {
  console.log(`Processing: ${(progress * 100).toFixed(1)}%`);
});
```

### 3.3 Common Operations

**Trim video:**
```javascript
await ffmpeg.writeFile('input.mp4', await fetchFile(videoFile));

await ffmpeg.exec([
  '-i', 'input.mp4',
  '-ss', '10',      // Start at 10 seconds
  '-to', '40',      // End at 40 seconds
  '-c', 'copy',     // No re-encode (fast)
  'output.mp4'
]);

const data = await ffmpeg.readFile('output.mp4');
```

**Add subtitles:**
```javascript
await ffmpeg.writeFile('subs.ass', subtitleContent);

await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vf', 'ass=subs.ass',
  '-c:a', 'copy',
  'output.mp4'
]);
```

**Extract audio (for Whisper):**
```javascript
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vn',                    // No video
  '-acodec', 'pcm_f32le',   // 32-bit float
  '-ar', '16000',           // 16kHz (Whisper requirement)
  '-ac', '1',               // Mono
  'audio.pcm'
]);
```

**Export with quality settings:**
```javascript
// 720p draft
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vf', 'scale=-1:720',
  '-c:v', 'libx264',
  '-preset', 'fast',
  '-crf', '23',
  '-c:a', 'aac',
  '-b:a', '128k',
  'output.mp4'
]);

// 1080p high quality
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vf', 'scale=-1:1080',
  '-c:v', 'libx264',
  '-preset', 'medium',
  '-crf', '20',
  '-c:a', 'aac',
  '-b:a', '192k',
  'output.mp4'
]);
```

---

## 4. Speech-to-Text with Whisper

### 4.1 How Whisper Works

OpenAI's Whisper is a transformer model trained on 680,000 hours of audio:

```
Audio → Mel Spectrogram → Encoder → Decoder → Text
```

**Features:**
- Multilingual (99+ languages)
- Automatic language detection
- Word-level timestamps

### 4.2 Whisper Web Usage

```javascript
import { transcribe } from '@remotion/whisper-web';

const result = await transcribe({
  audioData: audioFloat32Array,
  model: 'base',
  language: 'en',  // or 'auto'
  onProgress: (progress) => {
    console.log(`Transcribing: ${(progress * 100).toFixed(1)}%`);
  },
});

console.log(result.text);    // Full transcript
console.log(result.words);   // Word-level timestamps
```

### 4.3 Word Timestamps

Word timestamps enable animated captions:

```javascript
// Result structure
{
  text: "Hello world, this is a test.",
  words: [
    { word: "Hello", start: 0.0, end: 0.32, confidence: 0.98 },
    { word: "world,", start: 0.32, end: 0.64, confidence: 0.97 },
    { word: "this", start: 0.72, end: 0.88, confidence: 0.99 },
    { word: "is", start: 0.88, end: 0.96, confidence: 0.99 },
    { word: "a", start: 0.96, end: 1.04, confidence: 0.98 },
    { word: "test.", start: 1.04, end: 1.36, confidence: 0.97 },
  ]
}
```

### 4.4 Model Sizes

| Model | Size | Speed | Accuracy | Best For |
|-------|------|-------|----------|----------|
| tiny | ~75MB | 10x realtime | ~88% | Quick preview |
| base | ~150MB | 5x realtime | ~94% | Default |
| small | ~500MB | 2x realtime | ~97% | High accuracy |

**FlowCut uses `base` by default** — good balance of speed and accuracy.

### 4.5 Preparing Audio for Whisper

Whisper expects specific format:

```javascript
// Extract audio from video
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vn',
  '-acodec', 'pcm_f32le',
  '-ar', '16000',
  '-ac', '1',
  'audio.pcm'
]);

// Read as Float32Array
const audioData = await ffmpeg.readFile('audio.pcm');
const float32Array = new Float32Array(audioData.buffer);
```

---

## 5. Caption Rendering

### 5.1 Subtitle Formats

**SRT (Simple):**
```
1
00:00:01,000 --> 00:00:04,000
Hello, this is the first subtitle.

2
00:00:05,000 --> 00:00:08,000
And this is the second one.
```

**ASS (Advanced — used by FlowCut):**
```
[Script Info]
ScriptType: v4.00+
PlayResX: 1920
PlayResY: 1080

[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, ...
Style: Default,Arial,48,&H00FFFFFF,...

[Events]
Format: Layer, Start, End, Style, Text
Dialogue: 0,0:00:01.00,0:00:04.00,Default,Hello world
```

### 5.2 ASS Styling Tags

ASS supports inline styling:

```
{\fn Arial}           - Font name
{\fs 48}              - Font size
{\c&HFFFFFF&}         - Primary color (BGR format!)
{\3c&H000000&}        - Outline color
{\bord 2}             - Border width
{\shad 1}             - Shadow depth
{\pos(960,900)}       - Position
{\fad(200,200)}       - Fade in/out
{\k50}                - Karaoke timing (50 centiseconds)
```

### 5.3 Karaoke Effect

Word-by-word highlighting:

```javascript
function generateKaraokeASS(words, style) {
  let assText = '';
  
  words.forEach(word => {
    const duration = (word.end - word.start) * 100; // centiseconds
    assText += `{\\k${Math.round(duration)}}${word.word} `;
  });
  
  return assText;
}

// Output: {\k32}Hello {\k28}world {\k24}this {\k16}is {\k28}a {\k32}test
```

### 5.4 Canvas Preview

Real-time caption preview without re-encoding:

```javascript
class CaptionAnimator {
  render(currentTime) {
    // Find current segment
    const segment = this.transcript.segments.find(s => 
      currentTime >= s.start && currentTime < s.end
    );
    
    if (!segment) return;
    
    // Get words for this segment
    const words = this.transcript.words.filter(w =>
      w.start >= segment.start && w.end <= segment.end
    );
    
    // Render each word with highlight effect
    words.forEach(word => {
      const isActive = currentTime >= word.start && currentTime < word.end;
      this.renderWord(word.word, isActive);
    });
  }
}
```

---

## 6. Video Encoding

### 6.1 Quality Settings

**CRF (Constant Rate Factor):**
- Range: 0-51 (lower = better quality)
- 18 = visually lossless
- 23 = default, good balance
- 28 = smaller file, visible compression

**Preset (speed vs compression):**
```
ultrafast → fast → medium → slow → veryslow
   ↑                                    ↑
Fastest encoding              Best compression
```

### 6.2 Resolution Guidelines

| Resolution | Bitrate | File Size (1 min) |
|------------|---------|-------------------|
| 720p | 2.5-5 Mbps | 20-40 MB |
| 1080p | 5-10 Mbps | 40-80 MB |

### 6.3 Social Media Requirements

| Platform | Max Resolution | Max Duration | Aspect Ratio |
|----------|---------------|--------------|--------------|
| TikTok | 1080x1920 | 10 min | 9:16 |
| Instagram Reels | 1080x1920 | 90 sec | 9:16 |
| YouTube Shorts | 1080x1920 | 60 sec | 9:16 |

---

## 7. Browser Limitations

### 7.1 Memory Limits

**Browser tab memory limit: ~2GB**

This is a hard limit. Exceed it and the tab crashes.

**Memory usage for a 200MB video:**
```
Input file in MEMFS:          200 MB
Whisper model:                150 MB
Audio Float32Array:            20 MB
FFmpeg buffers:                60 MB
Output file:                  180 MB
Application:                  100 MB
────────────────────────────────────
TOTAL:                       ~710 MB (safe)
```

**Why FlowCut limits to 200-300MB:** Leaves headroom for browser variance.

### 7.2 Required Headers

SharedArrayBuffer (needed for multi-threaded WASM) requires:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

### 7.3 Browser Support

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| WebCodecs | 94+ | 130+ | 16.4+ | 94+ |
| SharedArrayBuffer | 92+ | 79+ | 15.2+ | 92+ |
| WASM SIMD | 91+ | 89+ | 16.4+ | 91+ |

### 7.4 Common Issues

**"SharedArrayBuffer is not defined"**
```javascript
if (typeof SharedArrayBuffer === 'undefined') {
  console.warn('Using single-threaded mode');
  ffmpeg.load({ singleThread: true });
}
```

**"Out of memory"**
- Enforce file size limits
- Clean up buffers after use
- Process shorter videos

---

## 8. Glossary

| Term | Definition |
|------|------------|
| **AAC** | Audio codec (Advanced Audio Coding) |
| **ASS** | Subtitle format with styling support |
| **Bitrate** | Data per second in video/audio |
| **Codec** | Compression/decompression algorithm |
| **Container** | File format (MP4, WebM) |
| **CRF** | Quality setting for encoding |
| **FFmpeg** | Video processing tool |
| **I-frame** | Keyframe (complete image) |
| **P-frame** | Predicted frame (stores differences) |
| **Resolution** | Pixel dimensions |
| **SRT** | Simple subtitle format |
| **WASM** | WebAssembly |
| **Whisper** | OpenAI's speech recognition model |

---

## Further Reading

- [FFmpeg.wasm Documentation](https://ffmpegwasm.netlify.app/)
- [Whisper Web](https://whisperweb.dev/)
- [ASS Subtitle Format](http://www.tcax.org/docs/ass-specs.htm)
- [WebCodecs API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API)

---

*Document maintained by: Engineering*  
*Last updated: February 28, 2026*
