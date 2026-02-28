# Technical Knowledge Document
## Understanding Video Processing in the Browser (and When to Use the Cloud)

**Version:** 3.0  
**Date:** February 28, 2026  
**Purpose:** Educational reference for understanding how video features work and why browser processing has hard limits

---

## Table of Contents

1. [Video Fundamentals](#1-video-fundamentals)
2. [Browser Video APIs](#2-browser-video-apis)
3. [FFmpeg.wasm Deep Dive](#3-ffmpegwasm-deep-dive)
4. [Speech-to-Text with Whisper](#4-speech-to-text-with-whisper)
5. [Face Detection & Tracking](#5-face-detection--tracking)
6. [Caption Rendering](#6-caption-rendering)
7. [Video Encoding & Formats](#7-video-encoding--formats)
8. [Performance Optimization](#8-performance-optimization)
9. [Browser Limitations](#9-browser-limitations)
10. [Transcript-Based Video Editing](#10-transcript-based-video-editing)
11. [Filler Word & Silence Detection](#11-filler-word--silence-detection)
12. [Text Overlays & Title Rendering](#12-text-overlays--title-rendering)
13. [Brand Templates & Style Persistence](#13-brand-templates--style-persistence)
14. [**Why Browser Processing Has Hard Limits**](#14-why-browser-processing-has-hard-limits)
15. [**Hybrid Processing: Browser + Cloud**](#15-hybrid-processing-browser--cloud)
16. [Glossary](#16-glossary)

---

## 1. Video Fundamentals

### 1.1 What is a Video File?

A video file is a **container** that holds multiple streams of data:

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
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  METADATA                                            │   │
│  │  - Duration, creation date, etc.                     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Key Concepts:**

| Term | Definition |
|------|------------|
| **Container** | File format that holds streams (MP4, WebM, MOV, MKV) |
| **Codec** | Algorithm to compress/decompress video (H.264, VP9, AV1) |
| **Bitrate** | Data per second (higher = better quality, larger file) |
| **Frame Rate** | Frames per second (24, 30, 60 fps common) |
| **Resolution** | Pixel dimensions (1920x1080 = 1080p) |

### 1.2 How Video Compression Works

Video files would be enormous without compression. A 1-minute 1080p video at 30fps:
- Uncompressed: ~10 GB (1920 × 1080 × 3 bytes × 30 fps × 60 sec)
- H.264 compressed: ~50 MB (200x smaller!)

**Compression Techniques:**

1. **Intra-frame compression (I-frames)**: Compress each frame like a JPEG image
2. **Inter-frame compression (P/B-frames)**: Store only differences between frames
3. **Motion estimation**: Describe movement instead of redrawing pixels

```
I-frame    P-frame    P-frame    P-frame    I-frame
[Full]  →  [Diff]  →  [Diff]  →  [Diff]  →  [Full]
  ↑                                            ↑
Keyframe                                   Keyframe
(can seek to)                           (can seek to)
```

**Why this matters for editing:**
- You can only seek accurately to I-frames (keyframes)
- Cutting at non-keyframe positions requires re-encoding
- This is why "smart rendering" exists (only re-encode affected sections)

### 1.3 Aspect Ratios Explained

```
16:9 (Landscape - YouTube)          9:16 (Portrait - TikTok/Reels)
┌─────────────────────┐             ┌───────┐
│                     │             │       │
│                     │             │       │
│                     │             │       │
│                     │             │       │
└─────────────────────┘             │       │
                                    │       │
                                    │       │
                                    └───────┘

1:1 (Square - Instagram)            4:5 (Portrait - Instagram Feed)
┌───────────┐                       ┌─────────┐
│           │                       │         │
│           │                       │         │
│           │                       │         │
│           │                       │         │
│           │                       │         │
└───────────┘                       └─────────┘
```

**Converting 16:9 to 9:16:**
- Option 1: Crop (lose sides) - requires smart positioning
- Option 2: Letterbox (add black bars) - wastes screen space
- Option 3: Blur fill (blurred video as background) - popular compromise

---

## 2. Browser Video APIs

### 2.1 The HTML5 Video Element

The simplest way to work with video in browsers:

```html
<video id="player" controls>
  <source src="video.mp4" type="video/mp4">
</video>
```

```javascript
const video = document.getElementById('player');

// Basic controls
video.play();
video.pause();
video.currentTime = 30; // Seek to 30 seconds

// Properties
console.log(video.duration);     // Total length in seconds
console.log(video.videoWidth);   // Native width
console.log(video.videoHeight);  // Native height

// Events
video.addEventListener('timeupdate', () => {
  console.log('Current time:', video.currentTime);
});

video.addEventListener('loadedmetadata', () => {
  console.log('Video loaded, duration:', video.duration);
});
```

**Limitations:**
- Can only play, not edit
- No access to individual frames
- No encoding capabilities

### 2.2 Canvas API for Video Manipulation

Canvas lets you draw video frames and manipulate them:

```javascript
const video = document.getElementById('video');
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

// Draw current video frame to canvas
function drawFrame() {
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
  requestAnimationFrame(drawFrame);
}

// Crop video (draw only a portion)
function drawCropped(cropX, cropY, cropW, cropH) {
  ctx.drawImage(
    video,
    cropX, cropY, cropW, cropH,  // Source rectangle
    0, 0, canvas.width, canvas.height  // Destination rectangle
  );
}

// Apply filters
ctx.filter = 'brightness(1.2) contrast(1.1)';
ctx.drawImage(video, 0, 0);

// Get pixel data for analysis
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
const pixels = imageData.data; // RGBA array
```

**Use Cases:**
- Real-time preview with effects
- Frame extraction for thumbnails
- Overlaying graphics (captions, watermarks)

### 2.3 WebCodecs API (Modern Approach)

WebCodecs provides low-level access to browser's built-in codecs:

```javascript
// Decode video frames
const decoder = new VideoDecoder({
  output: (frame) => {
    // Process each decoded frame
    console.log('Frame:', frame.timestamp, frame.codedWidth, frame.codedHeight);
    
    // Draw to canvas
    ctx.drawImage(frame, 0, 0);
    
    // IMPORTANT: Close frame to free memory
    frame.close();
  },
  error: (e) => console.error('Decoder error:', e),
});

// Configure decoder for H.264
decoder.configure({
  codec: 'avc1.42E01E', // H.264 Baseline Profile
  codedWidth: 1920,
  codedHeight: 1080,
});

// Feed encoded chunks to decoder
decoder.decode(encodedVideoChunk);

// Encode frames back to video
const encoder = new VideoEncoder({
  output: (chunk, metadata) => {
    // Collect encoded chunks
    encodedChunks.push(chunk);
  },
  error: (e) => console.error('Encoder error:', e),
});

encoder.configure({
  codec: 'avc1.42E01E',
  width: 1920,
  height: 1080,
  bitrate: 5_000_000, // 5 Mbps
  framerate: 30,
});

// Encode a frame
encoder.encode(videoFrame, { keyFrame: true });
```

**Advantages over Canvas:**
- Hardware acceleration
- Direct codec access
- Better performance for real-time processing

**Limitations:**
- No muxing (combining video + audio into container)
- Need FFmpeg.wasm for final file creation
- Browser support varies (no Safari until recently)

### 2.4 MediaRecorder API

Record canvas or video streams:

```javascript
// Capture canvas as video stream
const stream = canvas.captureStream(30); // 30 fps

// Add audio from original video
const audioCtx = new AudioContext();
const source = audioCtx.createMediaElementSource(video);
const dest = audioCtx.createMediaStreamDestination();
source.connect(dest);
stream.addTrack(dest.stream.getAudioTracks()[0]);

// Record the stream
const recorder = new MediaRecorder(stream, {
  mimeType: 'video/webm;codecs=vp9',
  videoBitsPerSecond: 5000000,
});

const chunks = [];
recorder.ondataavailable = (e) => chunks.push(e.data);
recorder.onstop = () => {
  const blob = new Blob(chunks, { type: 'video/webm' });
  const url = URL.createObjectURL(blob);
  // Download or use the video
};

recorder.start();
// ... later
recorder.stop();
```

**Use Cases:**
- Quick exports with real-time effects
- Screen recording
- Live streaming

**Limitations:**
- Limited format support (mainly WebM)
- Less control over encoding parameters
- Quality can be inconsistent

---

## 3. FFmpeg.wasm Deep Dive

### 3.1 What is FFmpeg?

FFmpeg is the Swiss Army knife of video processing. It can:
- Convert between any video formats
- Trim, crop, resize videos
- Add/remove audio tracks
- Apply filters and effects
- Burn in subtitles
- Extract frames

**FFmpeg.wasm** compiles FFmpeg to WebAssembly, running it entirely in the browser.

### 3.2 Setting Up FFmpeg.wasm

```javascript
import { FFmpeg } from '@ffmpeg/ffmpeg';
import { fetchFile, toBlobURL } from '@ffmpeg/util';

// Create FFmpeg instance
const ffmpeg = new FFmpeg();

// Load FFmpeg (downloads ~30MB of WASM files)
await ffmpeg.load({
  coreURL: await toBlobURL('/ffmpeg-core.js', 'text/javascript'),
  wasmURL: await toBlobURL('/ffmpeg-core.wasm', 'application/wasm'),
});

// Progress callback
ffmpeg.on('progress', ({ progress, time }) => {
  console.log(`Processing: ${(progress * 100).toFixed(1)}%`);
});

// Log output
ffmpeg.on('log', ({ message }) => {
  console.log('FFmpeg:', message);
});
```

### 3.3 Common FFmpeg Operations

#### Trimming Video

```javascript
// Trim video from 10s to 40s
await ffmpeg.writeFile('input.mp4', await fetchFile(videoFile));

await ffmpeg.exec([
  '-i', 'input.mp4',
  '-ss', '10',        // Start time
  '-to', '40',        // End time
  '-c', 'copy',       // Copy streams without re-encoding (fast!)
  'output.mp4'
]);

const data = await ffmpeg.readFile('output.mp4');
const blob = new Blob([data], { type: 'video/mp4' });
```

**Note:** `-c copy` is fast but can only cut at keyframes. For frame-accurate cuts:

```javascript
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-ss', '10.5',      // Precise start
  '-to', '40.3',      // Precise end
  '-c:v', 'libx264',  // Re-encode video
  '-c:a', 'aac',      // Re-encode audio
  'output.mp4'
]);
```

#### Cropping/Reframing

```javascript
// Crop to center 9:16 from 16:9
// Original: 1920x1080, Target: 607x1080 (9:16)
const cropWidth = 607;
const cropX = (1920 - cropWidth) / 2; // Center horizontally

await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vf', `crop=${cropWidth}:1080:${cropX}:0`,
  '-c:a', 'copy',
  'output.mp4'
]);
```

#### Adding Subtitles

```javascript
// Write subtitle file
await ffmpeg.writeFile('subs.ass', subtitleContent);

// Burn subtitles into video
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vf', 'ass=subs.ass',
  '-c:a', 'copy',
  'output.mp4'
]);
```

#### Extracting Audio

```javascript
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vn',              // No video
  '-acodec', 'pcm_s16le',  // Uncompressed audio
  '-ar', '16000',     // 16kHz sample rate (good for speech recognition)
  '-ac', '1',         // Mono
  'audio.wav'
]);
```

#### Quality Presets

```javascript
// High quality (slower, larger file)
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-c:v', 'libx264',
  '-preset', 'slow',
  '-crf', '18',       // Lower = better quality (18-28 typical)
  '-c:a', 'aac',
  '-b:a', '192k',
  'output_hq.mp4'
]);

// Fast/draft quality
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-c:v', 'libx264',
  '-preset', 'ultrafast',
  '-crf', '28',
  '-c:a', 'aac',
  '-b:a', '128k',
  'output_draft.mp4'
]);
```

### 3.4 FFmpeg Filter Graph

Complex operations use filter graphs:

```javascript
// Multiple filters: crop + scale + add text
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vf', [
    'crop=607:1080:656:0',           // Crop to 9:16
    'scale=1080:1920',               // Scale up
    'drawtext=text=\'Hello\':fontsize=48:fontcolor=white:x=50:y=50'
  ].join(','),
  '-c:a', 'copy',
  'output.mp4'
]);
```

### 3.5 Memory Management in FFmpeg.wasm

FFmpeg.wasm runs in a virtual filesystem in memory:

```javascript
// Write file to FFmpeg's virtual filesystem
await ffmpeg.writeFile('input.mp4', videoData);

// After processing, clean up
await ffmpeg.deleteFile('input.mp4');
await ffmpeg.deleteFile('output.mp4');

// For large files, process in chunks or use streaming (advanced)
```

---

## 4. Speech-to-Text with Whisper

### 4.1 How Whisper Works

OpenAI's Whisper is a transformer-based model trained on 680,000 hours of audio:

```
Audio Input → Mel Spectrogram → Encoder → Decoder → Text Output
     ↓              ↓              ↓          ↓
  [Waveform]   [Visual repr.   [Understand  [Generate
               of frequencies]  context]     words]
```

**Key Features:**
- Multilingual (99+ languages)
- Automatic language detection
- Punctuation and capitalization
- Word-level timestamps

### 4.2 Whisper Web Implementation

```javascript
import { transcribe } from '@remotion/whisper-web';

// Transcribe audio
const result = await transcribe({
  audioData: audioFloat32Array,  // Audio as Float32Array
  model: 'base',                 // 'tiny', 'base', 'small'
  language: 'en',                // or 'auto' for detection
  onProgress: (progress) => {
    console.log(`Transcribing: ${(progress * 100).toFixed(1)}%`);
  },
});

console.log(result.text);        // Full transcript
console.log(result.segments);    // Segments with timestamps
console.log(result.words);       // Word-level timestamps
```

### 4.3 Preparing Audio for Whisper

Whisper expects specific audio format:

```javascript
// Extract audio from video using FFmpeg
await ffmpeg.exec([
  '-i', 'input.mp4',
  '-vn',                    // No video
  '-acodec', 'pcm_f32le',   // 32-bit float PCM
  '-ar', '16000',           // 16kHz sample rate
  '-ac', '1',               // Mono
  'audio.pcm'
]);

// Read as Float32Array
const audioData = await ffmpeg.readFile('audio.pcm');
const float32Array = new Float32Array(audioData.buffer);
```

### 4.4 Word-Level Timestamps

Word timestamps enable animated captions:

```javascript
const result = await transcribe({
  audioData,
  model: 'base',
  wordTimestamps: true,  // Enable word-level timing
});

// Result structure:
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

### 4.5 Model Size Trade-offs

| Model | Size | Speed | Accuracy | Best For |
|-------|------|-------|----------|----------|
| tiny | ~75MB | 32x realtime | ~88% | Quick previews, testing |
| base | ~150MB | 16x realtime | ~94% | Default, good balance |
| small | ~500MB | 6x realtime | ~97% | High accuracy needs |
| medium | ~1.5GB | 2x realtime | ~98% | Professional use |
| large | ~3GB | 1x realtime | ~99% | Maximum accuracy |

**Note:** Browser implementations typically support up to "small" due to memory constraints.

---

## 5. Face Detection & Tracking

### 5.1 How Face Detection Works

Modern face detection uses neural networks:

```
Image → CNN Feature Extraction → Bounding Box Regression → Face Locations
                                         ↓
                               [x, y, width, height, confidence]
```

### 5.2 MediaPipe Face Detection

```javascript
import { FaceDetector, FilesetResolver } from '@mediapipe/tasks-vision';

// Initialize
const vision = await FilesetResolver.forVisionTasks(
  'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision/wasm'
);

const faceDetector = await FaceDetector.createFromOptions(vision, {
  baseOptions: {
    modelAssetPath: 'face_detection_short_range.tflite',
    delegate: 'GPU',  // Use WebGL acceleration
  },
  runningMode: 'VIDEO',
  minDetectionConfidence: 0.5,
});

// Detect faces in video frame
function detectFaces(videoElement, timestamp) {
  const detections = faceDetector.detectForVideo(videoElement, timestamp);
  
  return detections.detections.map(detection => ({
    boundingBox: detection.boundingBox,
    keypoints: detection.keypoints,  // Eyes, nose, mouth, ears
    confidence: detection.categories[0].score,
  }));
}
```

### 5.3 Face Tracking Across Frames

For smooth reframing, track faces across the entire video:

```javascript
async function trackFacesInVideo(video) {
  const tracks = [];
  const fps = 5;  // Sample 5 frames per second
  const interval = 1000 / fps;
  
  video.currentTime = 0;
  
  while (video.currentTime < video.duration) {
    await new Promise(resolve => {
      video.onseeked = resolve;
      video.currentTime += interval / 1000;
    });
    
    const faces = detectFaces(video, video.currentTime * 1000);
    
    faces.forEach(face => {
      // Match to existing track or create new one
      const track = findOrCreateTrack(tracks, face);
      track.frames.push({
        timestamp: video.currentTime,
        boundingBox: face.boundingBox,
      });
    });
  }
  
  return tracks;
}

function findOrCreateTrack(tracks, face) {
  // Simple IoU (Intersection over Union) matching
  for (const track of tracks) {
    const lastFrame = track.frames[track.frames.length - 1];
    if (calculateIoU(lastFrame.boundingBox, face.boundingBox) > 0.5) {
      return track;
    }
  }
  
  // New track
  const newTrack = { id: generateId(), frames: [] };
  tracks.push(newTrack);
  return newTrack;
}
```

### 5.4 Smart Crop Calculation

```javascript
function calculateSmartCrop(faceTracks, sourceSize, targetAspect) {
  // Find primary subject (most screen time)
  const primaryTrack = faceTracks.reduce((a, b) => 
    a.frames.length > b.frames.length ? a : b
  );
  
  // Calculate average position
  const avgX = primaryTrack.frames.reduce((sum, f) => 
    sum + f.boundingBox.x + f.boundingBox.width / 2, 0
  ) / primaryTrack.frames.length;
  
  // Calculate crop dimensions
  const targetWidth = sourceSize.height * targetAspect;
  const targetHeight = sourceSize.height;
  
  // Center crop on face with bounds checking
  let cropX = avgX - targetWidth / 2;
  cropX = Math.max(0, Math.min(cropX, sourceSize.width - targetWidth));
  
  return {
    x: Math.round(cropX),
    y: 0,
    width: Math.round(targetWidth),
    height: targetHeight,
  };
}
```

### 5.5 Keyframe-Based Dynamic Reframing

For videos where the subject moves significantly:

```javascript
function generateCropKeyframes(faceTracks, sourceSize, targetAspect) {
  const keyframes = [];
  const primaryTrack = getPrimaryTrack(faceTracks);
  
  // Sample every N frames for keyframes
  const keyframeInterval = 30; // Every 30 frames
  
  for (let i = 0; i < primaryTrack.frames.length; i += keyframeInterval) {
    const frame = primaryTrack.frames[i];
    const crop = calculateCropForFrame(frame, sourceSize, targetAspect);
    
    keyframes.push({
      timestamp: frame.timestamp,
      crop: crop,
    });
  }
  
  return keyframes;
}

// FFmpeg command with keyframe-based cropping (complex filter)
function generateDynamicCropFilter(keyframes) {
  // This requires FFmpeg's zoompan or crop with expressions
  // Simplified example:
  return keyframes.map((kf, i) => {
    const next = keyframes[i + 1];
    if (!next) return null;
    
    return `crop=${kf.crop.width}:${kf.crop.height}:${kf.crop.x}:${kf.crop.y}:enable='between(t,${kf.timestamp},${next.timestamp})'`;
  }).filter(Boolean).join(',');
}
```

---

## 6. Caption Rendering

### 6.1 Subtitle Formats

**SRT (SubRip)** - Simple, widely supported:
```
1
00:00:01,000 --> 00:00:04,000
Hello, this is the first subtitle.

2
00:00:05,000 --> 00:00:08,000
And this is the second one.
```

**WebVTT** - Web standard with basic styling:
```
WEBVTT

00:00:01.000 --> 00:00:04.000
Hello, this is the first subtitle.

00:00:05.000 --> 00:00:08.000 align:center
And this is <b>styled</b> text.
```

**ASS (Advanced SubStation Alpha)** - Full styling and animation:
```
[Script Info]
ScriptType: v4.00+
PlayResX: 1920
PlayResY: 1080

[V4+ Styles]
Format: Name, Fontname, Fontsize, PrimaryColour, ...
Style: Default,Arial,48,&H00FFFFFF,&H000000FF,...

[Events]
Format: Layer, Start, End, Style, Text
Dialogue: 0,0:00:01.00,0:00:04.00,Default,Hello world
```

### 6.2 ASS Styling Deep Dive

ASS supports inline styling tags:

```
{\fn Arial}           - Font name
{\fs 48}              - Font size
{\c&HFFFFFF&}         - Primary color (BGR format!)
{\3c&H000000&}        - Outline color
{\bord 2}             - Border width
{\shad 1}             - Shadow depth
{\pos(960,900)}       - Position
{\fad(200,200)}       - Fade in/out
{\t(\fs72)}           - Animate to size 72
{\k50}                - Karaoke timing (50 centiseconds)
```

### 6.3 Karaoke-Style Captions

Word-by-word highlighting using ASS:

```javascript
function generateKaraokeASS(words, style) {
  let assText = '';
  
  words.forEach(word => {
    const duration = (word.end - word.start) * 100; // centiseconds
    // \k = karaoke tag, fills text over duration
    assText += `{\\k${Math.round(duration)}}${word.word} `;
  });
  
  return assText;
}

// Output example:
// {\k32}Hello {\k28}world {\k24}this {\k16}is {\k12}a {\k28}test
```

### 6.4 Animated Captions with Canvas

For real-time preview without re-encoding:

```javascript
class CaptionAnimator {
  constructor(canvas, transcript, style) {
    this.ctx = canvas.getContext('2d');
    this.transcript = transcript;
    this.style = style;
  }
  
  render(currentTime) {
    // Find current segment
    const segment = this.transcript.segments.find(s => 
      currentTime >= s.start && currentTime < s.end
    );
    
    if (!segment) return;
    
    // Clear previous
    this.ctx.clearRect(0, 0, this.ctx.canvas.width, this.ctx.canvas.height);
    
    // Get words for this segment
    const words = this.transcript.words.filter(w =>
      w.start >= segment.start && w.end <= segment.end
    );
    
    // Render each word
    let x = this.style.startX;
    words.forEach(word => {
      const isActive = currentTime >= word.start && currentTime < word.end;
      const progress = isActive 
        ? (currentTime - word.start) / (word.end - word.start)
        : currentTime >= word.end ? 1 : 0;
      
      this.renderWord(word.word, x, this.style.y, progress);
      x += this.measureWord(word.word);
    });
  }
  
  renderWord(text, x, y, progress) {
    // Karaoke effect: fill from left to right
    this.ctx.save();
    
    // Background (unhighlighted)
    this.ctx.fillStyle = this.style.color;
    this.ctx.fillText(text, x, y);
    
    // Foreground (highlighted portion)
    if (progress > 0) {
      const width = this.ctx.measureText(text).width * progress;
      this.ctx.beginPath();
      this.ctx.rect(x, y - this.style.fontSize, width, this.style.fontSize * 1.5);
      this.ctx.clip();
      this.ctx.fillStyle = this.style.highlightColor;
      this.ctx.fillText(text, x, y);
    }
    
    this.ctx.restore();
  }
}
```

### 6.5 Caption Style Presets

```javascript
const captionPresets = {
  classic: {
    fontFamily: 'Arial',
    fontSize: 48,
    color: '#FFFFFF',
    backgroundColor: 'rgba(0,0,0,0.7)',
    position: 'bottom',
    animation: { type: 'none' },
  },
  
  karaoke: {
    fontFamily: 'Montserrat',
    fontSize: 56,
    color: '#FFFFFF',
    highlightColor: '#FFFF00',
    strokeColor: '#000000',
    strokeWidth: 3,
    position: 'center',
    animation: { type: 'karaoke' },
  },
  
  bounce: {
    fontFamily: 'Poppins',
    fontSize: 52,
    color: '#FFFFFF',
    strokeColor: '#000000',
    strokeWidth: 2,
    position: 'bottom',
    animation: { 
      type: 'bounce',
      intensity: 0.3,
      duration: 200,
    },
  },
  
  glow: {
    fontFamily: 'Inter',
    fontSize: 48,
    color: '#FFFFFF',
    glowColor: '#00FFFF',
    glowIntensity: 20,
    position: 'bottom',
    animation: { type: 'glow' },
  },
};
```

---

## 7. Video Encoding & Formats

### 7.1 Container vs Codec

```
┌──────────────────────────────────────────────────────────────┐
│                        CONTAINER                              │
│  (MP4, WebM, MOV, MKV)                                       │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                    VIDEO CODEC                          │  │
│  │  (H.264, H.265/HEVC, VP9, AV1)                         │  │
│  └────────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                    AUDIO CODEC                          │  │
│  │  (AAC, MP3, Opus, FLAC)                                │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 7.2 Common Format Combinations

| Use Case | Container | Video Codec | Audio Codec |
|----------|-----------|-------------|-------------|
| Universal playback | MP4 | H.264 | AAC |
| Web optimized | WebM | VP9 | Opus |
| Apple devices | MOV | H.264/HEVC | AAC |
| High quality archive | MKV | H.265 | FLAC |
| Social media | MP4 | H.264 | AAC |

### 7.3 Quality Settings Explained

**CRF (Constant Rate Factor):**
- Range: 0-51 (lower = better quality, larger file)
- Recommended: 18-23 for good quality
- 18 = visually lossless
- 23 = default, good balance
- 28 = smaller file, visible compression

```javascript
// High quality
'-crf', '18'

// Balanced
'-crf', '23'

// Small file
'-crf', '28'
```

**Preset (encoding speed vs compression):**
```
ultrafast → superfast → veryfast → faster → fast → medium → slow → slower → veryslow
   ↑                                                                           ↑
Fastest encoding                                                    Best compression
Largest file                                                        Smallest file
```

**Bitrate vs CRF:**
- CRF: Variable bitrate, consistent quality
- Bitrate: Fixed bitrate, variable quality

```javascript
// CRF mode (recommended for local files)
'-crf', '23'

// Bitrate mode (for streaming/specific file size)
'-b:v', '5M'  // 5 Mbps
```

### 7.4 Resolution and Bitrate Guidelines

| Resolution | Recommended Bitrate | File Size (1 min) |
|------------|---------------------|-------------------|
| 720p | 2.5-5 Mbps | 20-40 MB |
| 1080p | 5-10 Mbps | 40-80 MB |
| 1440p | 10-20 Mbps | 80-150 MB |
| 4K | 20-50 Mbps | 150-400 MB |

### 7.5 Social Media Requirements

| Platform | Max Resolution | Max Duration | Max File Size | Aspect Ratio |
|----------|---------------|--------------|---------------|--------------|
| TikTok | 1080x1920 | 10 min | 287 MB | 9:16 |
| Instagram Reels | 1080x1920 | 90 sec | 250 MB | 9:16 |
| YouTube Shorts | 1080x1920 | 60 sec | - | 9:16 |
| Twitter/X | 1920x1200 | 2:20 | 512 MB | 16:9, 1:1 |
| LinkedIn | 1920x1080 | 10 min | 5 GB | 16:9, 1:1, 9:16 |

---

## 8. Performance Optimization

### 8.1 Web Workers for Heavy Processing

```javascript
// video-worker.js
import { FFmpeg } from '@ffmpeg/ffmpeg';

let ffmpeg = null;

self.onmessage = async (e) => {
  const { type, payload } = e.data;
  
  switch (type) {
    case 'INIT':
      ffmpeg = new FFmpeg();
      await ffmpeg.load();
      self.postMessage({ type: 'READY' });
      break;
      
    case 'PROCESS':
      const result = await processVideo(payload);
      self.postMessage({ type: 'COMPLETE', payload: result });
      break;
  }
};

// Main thread
const worker = new Worker('video-worker.js');

worker.onmessage = (e) => {
  if (e.data.type === 'COMPLETE') {
    handleResult(e.data.payload);
  }
};

worker.postMessage({ type: 'INIT' });
```

### 8.2 Chunked Processing

For large files, process in chunks to avoid memory issues:

```javascript
async function processLargeVideo(file, chunkDuration = 30) {
  const metadata = await getVideoMetadata(file);
  const chunks = Math.ceil(metadata.duration / chunkDuration);
  const processedChunks = [];
  
  for (let i = 0; i < chunks; i++) {
    const start = i * chunkDuration;
    const end = Math.min((i + 1) * chunkDuration, metadata.duration);
    
    // Extract chunk
    const chunk = await extractChunk(file, start, end);
    
    // Process chunk
    const processed = await processChunk(chunk);
    processedChunks.push(processed);
    
    // Report progress
    onProgress((i + 1) / chunks);
    
    // Allow GC between chunks
    await sleep(100);
  }
  
  // Concatenate all chunks
  return await concatenateChunks(processedChunks);
}
```

### 8.3 Caching Strategies

```javascript
// Cache models in IndexedDB
async function loadModelWithCache(modelName) {
  const cached = await db.models.get(modelName);
  
  if (cached) {
    console.log('Loading model from cache');
    return cached.data;
  }
  
  console.log('Downloading model...');
  const response = await fetch(`/models/${modelName}`);
  const data = await response.arrayBuffer();
  
  // Cache for future use
  await db.models.put({
    id: modelName,
    name: modelName,
    data: data,
    downloadedAt: new Date(),
  });
  
  return data;
}
```

### 8.4 Memory Monitoring

```javascript
function monitorMemory() {
  if (!('memory' in performance)) {
    console.warn('Memory API not available');
    return;
  }
  
  const { usedJSHeapSize, jsHeapSizeLimit } = performance.memory;
  const usagePercent = (usedJSHeapSize / jsHeapSizeLimit) * 100;
  
  console.log(`Memory: ${(usedJSHeapSize / 1024 / 1024).toFixed(1)} MB / ${(jsHeapSizeLimit / 1024 / 1024).toFixed(1)} MB (${usagePercent.toFixed(1)}%)`);
  
  if (usagePercent > 80) {
    console.warn('High memory usage! Consider cleanup.');
    return 'high';
  }
  
  return 'normal';
}
```

### 8.5 WebGPU Acceleration

For Whisper and face detection:

```javascript
async function checkWebGPUSupport() {
  if (!navigator.gpu) {
    return { supported: false, reason: 'WebGPU not available' };
  }
  
  try {
    const adapter = await navigator.gpu.requestAdapter();
    if (!adapter) {
      return { supported: false, reason: 'No GPU adapter found' };
    }
    
    const device = await adapter.requestDevice();
    return { 
      supported: true, 
      device,
      limits: adapter.limits,
    };
  } catch (e) {
    return { supported: false, reason: e.message };
  }
}

// Use WebGPU for Whisper if available
const gpuSupport = await checkWebGPUSupport();
const whisperConfig = {
  model: 'base',
  device: gpuSupport.supported ? 'webgpu' : 'wasm',
};
```

---

## 9. Browser Limitations

### 9.1 Cross-Origin Isolation

SharedArrayBuffer (required for multi-threaded WASM) needs these headers:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

**Impact:**
- Cannot embed cross-origin iframes without `crossorigin` attribute
- Cannot load cross-origin images without CORS
- Some third-party scripts may break

### 9.2 Browser Support Matrix

| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| WebCodecs | 94+ | 130+ | 16.4+ | 94+ |
| SharedArrayBuffer | 92+ | 79+ | 15.2+ | 92+ |
| WebGPU | 113+ | Nightly | 17+ | 113+ |
| OffscreenCanvas | 69+ | 105+ | 16.4+ | 79+ |
| WASM SIMD | 91+ | 89+ | 16.4+ | 91+ |

### 9.3 File Size Limitations

| Browser | Max Blob Size | Max ArrayBuffer |
|---------|---------------|-----------------|
| Chrome | ~2 GB | ~2 GB |
| Firefox | ~2 GB | ~2 GB |
| Safari | ~1 GB | ~1 GB |

### 9.4 Common Issues and Solutions

**Issue: "SharedArrayBuffer is not defined"**
```javascript
// Solution: Add COOP/COEP headers
// Or use single-threaded fallback
if (typeof SharedArrayBuffer === 'undefined') {
  console.warn('Using single-threaded mode');
  ffmpeg.load({ singleThread: true });
}
```

**Issue: "Out of memory" on large files**
```javascript
// Solution: Process in chunks, monitor memory
const MAX_SAFE_SIZE = 500 * 1024 * 1024; // 500MB

if (file.size > MAX_SAFE_SIZE) {
  return processInChunks(file);
}
```

**Issue: Safari WebCodecs limitations**
```javascript
// Solution: Fallback to FFmpeg.wasm only
const useWebCodecs = 'VideoDecoder' in window && 
                     !navigator.userAgent.includes('Safari');
```

---

## 10. Transcript-Based Video Editing

### 10.1 The Core Idea

Transcript-based editing flips the traditional video editing paradigm: instead of scrubbing a timeline to find content, you **read text and edit it like a document**. Every word in the transcript maps to a precise time range in the video, so deleting a sentence removes that video segment.

```
Traditional editing:                    Transcript editing:

[Video timeline ████████████████]      "Hello everyone, um, today we're
 ↑ scrub, play, pause, find moment      going to talk about, you know,
                                         the basics of machine learning."
                                         
                                        → Select "um," → Delete → Video cut
                                        → Select "you know," → Delete → Video cut
```

### 10.2 How Word Timestamps Enable This

Whisper provides word-level timestamps — each word has a start and end time:

```javascript
// Whisper output (simplified)
const words = [
  { word: "Hello",     start: 0.00,  end: 0.32 },
  { word: "everyone,", start: 0.33,  end: 0.78 },
  { word: "um,",       start: 0.80,  end: 1.10 },  // ← filler
  { word: "today",     start: 1.20,  end: 1.50 },
  { word: "we're",     start: 1.51,  end: 1.70 },
  // ...
];
```

When a user selects and deletes "um," from the transcript, the system:
1. Looks up the word's time range: `{start: 0.80, end: 1.10}`
2. Adds it to a **cut list**: `[{start: 0.80, end: 1.10}]`
3. During export, FFmpeg excludes those time ranges

### 10.3 The Cut List Model

A cut list is an ordered array of time ranges to **exclude** from the final video:

```javascript
// User deletes two sections via transcript
const cutList = [
  { start: 0.80, end: 1.10 },   // "um,"
  { start: 5.20, end: 6.80 },   // "you know, like,"
];

// The kept segments are the inverse:
// [0.00 → 0.80], [1.10 → 5.20], [6.80 → end]
```

**Why a cut list instead of a kept list?** Because users make small deletions. Tracking what they removed is simpler and more undo-friendly than tracking everything they kept.

### 10.4 FFmpeg Concat Demuxer for Reassembly

The most reliable way to reassemble non-contiguous segments:

```javascript
// Step 1: Extract each kept segment as a separate file
const keptSegments = invertCutList(cutList, totalDuration);

for (const [i, seg] of keptSegments.entries()) {
  await ffmpeg.exec([
    '-i', 'input.mp4',
    '-ss', String(seg.start),
    '-to', String(seg.end),
    '-c', 'copy',              // No re-encode = fast
    '-avoid_negative_ts', '1',
    `segment_${i}.mp4`
  ]);
}

// Step 2: Create a concat list file
const concatList = keptSegments
  .map((_, i) => `file 'segment_${i}.mp4'`)
  .join('\n');
await ffmpeg.writeFile('segments.txt', concatList);

// Step 3: Concatenate
await ffmpeg.exec([
  '-f', 'concat',
  '-safe', '0',
  '-i', 'segments.txt',
  '-c', 'copy',
  'output.mp4'
]);
```

**Caveat — keyframe alignment:** `-c copy` only cuts at keyframes. For frame-accurate cuts at word boundaries, you may need to re-encode the segments that start/end mid-GOP:

```javascript
// Re-encode only the first/last 2 seconds of each segment for accuracy
// while keeping the middle as a stream copy (hybrid approach)
'-ss', String(seg.start),
'-to', String(seg.end),
'-c:v', 'libx264', '-preset', 'fast', '-crf', '18',
'-c:a', 'aac'
```

### 10.5 Audio Smoothing at Cut Points

Hard cuts can produce audible pops or clicks. A 10-20ms crossfade at each cut point eliminates this:

```javascript
// Using FFmpeg's afade filter
'-af', 'afade=t=out:st=4.98:d=0.02,afade=t=in:d=0.02'
//                    ↑ fade out last 20ms      ↑ fade in first 20ms
```

Alternatively, in the browser, apply a gain ramp on the raw audio samples:

```javascript
function applyCrossfade(audioBuffer, cutPoints, fadeDurationSamples = 320) {
  // 320 samples at 16kHz = 20ms
  for (const cut of cutPoints) {
    const sampleIdx = Math.floor(cut.time * audioBuffer.sampleRate);
    
    // Fade out before cut
    for (let i = 0; i < fadeDurationSamples; i++) {
      const gain = 1 - (i / fadeDurationSamples);
      audioBuffer.data[sampleIdx - fadeDurationSamples + i] *= gain;
    }
    
    // Fade in after cut
    for (let i = 0; i < fadeDurationSamples; i++) {
      const gain = i / fadeDurationSamples;
      audioBuffer.data[sampleIdx + i] *= gain;
    }
  }
}
```

---

## 11. Filler Word & Silence Detection

### 11.1 Why Filler Removal Matters

Studies show that removing filler words improves perceived speaker competence by 20-30% and increases viewer retention. This is one of the most requested features in any video editing tool — and competitors charge $15-29/month for it.

### 11.2 Two Detection Methods

ClipFlow uses two complementary techniques:

```
METHOD 1: TRANSCRIPT-BASED (Filler Words)
Whisper transcript → Word matching → Filler detections

METHOD 2: AUDIO-BASED (Silence Gaps)
Raw audio → RMS energy analysis → Silence detections
```

### 11.3 Filler Word Detection from Transcript

Simple but effective — match known filler words against the Whisper transcript:

```javascript
const FILLER_PATTERNS = [
  // Exact matches (case-insensitive)
  /^(um|uh|ah|er|erm|hmm)[\.,!?]?$/i,
  
  // Phrase-level fillers (require checking word sequences)
  /^you know[\.,]?$/i,
  /^i mean[\.,]?$/i,
  /^sort of[\.,]?$/i,
  /^kind of[\.,]?$/i,
];

// Context-aware detection to reduce false positives
function isFillerInContext(word, prevWord, nextWord) {
  const normalized = word.word.toLowerCase().replace(/[.,!?]/g, '');
  
  // "like" is only a filler when used as a verbal tic, not as a comparison
  if (normalized === 'like') {
    // Filler: "I was, like, going to..."
    // Not filler: "It looks like a cat"
    const prevNorm = prevWord?.word.toLowerCase() ?? '';
    return prevNorm.endsWith(',') || prevWord?.confidence < 0.7;
  }
  
  // "so" is a filler at sentence starts, not as a conjunction
  if (normalized === 'so') {
    return !prevWord || prevWord.word.endsWith('.');
  }
  
  return FILLER_PATTERNS.some(pattern => pattern.test(word.word));
}
```

**Key challenge: "like" and "so" have legitimate uses.** The context-aware check reduces false positives. Users can always undo individual detections.

### 11.4 Silence Detection from Audio

Uses Root Mean Square (RMS) energy analysis on the raw audio waveform:

```
Audio waveform:    ▃▅▇█▇▅▃▁▁▁▁▁▁▁▃▅▇█▇▅▃
                                  ↑ silence gap
                                  detected here
```

```javascript
function detectSilence(audioData, sampleRate, config) {
  const windowSize = Math.floor(sampleRate * 0.05); // 50ms analysis windows
  const threshold = dbToLinear(config.silenceThresholdDb); // -40dB default
  
  let silenceStart = null;
  const detections = [];
  
  for (let i = 0; i < audioData.length; i += windowSize) {
    const window = audioData.slice(i, i + windowSize);
    
    // Calculate RMS energy
    const rms = Math.sqrt(
      window.reduce((sum, sample) => sum + sample * sample, 0) / window.length
    );
    
    const time = i / sampleRate;
    
    if (rms < threshold) {
      if (silenceStart === null) silenceStart = time;
    } else {
      if (silenceStart !== null) {
        const duration = time - silenceStart;
        if (duration >= config.minSilenceDuration) {
          detections.push({ type: 'silence', start: silenceStart, end: time });
        }
        silenceStart = null;
      }
    }
  }
  
  return detections;
}

function dbToLinear(db) {
  return Math.pow(10, db / 20);
}
```

### 11.5 RMS Energy Explained

**RMS (Root Mean Square)** is the standard way to measure audio loudness:

```
Given audio samples: [0.3, -0.5, 0.4, -0.2, 0.1]

1. Square each:    [0.09, 0.25, 0.16, 0.04, 0.01]
2. Mean:           (0.09 + 0.25 + 0.16 + 0.04 + 0.01) / 5 = 0.11
3. Square root:    √0.11 = 0.332

RMS = 0.332 (on a 0-1 scale)
```

**Decibel conversion:**
- 0 dB = maximum digital level (RMS = 1.0)
- -20 dB = moderate audio
- -40 dB = very quiet (typical silence threshold)
- -60 dB = near silence

```javascript
const rmsToDb = (rms) => 20 * Math.log10(rms);
// rms=0.01 → -40 dB
// rms=0.001 → -60 dB
```

### 11.6 Combining Results

Filler word detections and silence detections are merged into a single cut list:

```javascript
function mergeDetections(fillerDetections, silenceDetections) {
  const all = [...fillerDetections, ...silenceDetections]
    .sort((a, b) => a.start - b.start);
  
  // Merge overlapping ranges
  const merged = [];
  for (const det of all) {
    const last = merged[merged.length - 1];
    if (last && det.start <= last.end + 0.05) { // 50ms tolerance
      last.end = Math.max(last.end, det.end);
    } else {
      merged.push({ ...det });
    }
  }
  
  return merged;
}
```

---

## 12. Text Overlays & Title Rendering

### 12.1 What Are Text Overlays?

Text overlays are visual elements placed on top of video — titles, callouts, lower thirds, CTAs, etc. Unlike captions (which follow speech timing), overlays are placed manually at specific times and positions.

```
┌─────────────────────────────────┐
│                                 │
│   ┌─────────────────────┐      │ ← Title overlay at top
│   │  "5 Tips for Better  │      │
│   │   Video Content"     │      │
│   └─────────────────────┘      │
│                                 │
│         [Video content]         │
│                                 │
│   ┌─────────────────────────┐  │ ← Lower third overlay
│   │ John Smith              │  │
│   │ Content Creator         │  │
│   └─────────────────────────┘  │
└─────────────────────────────────┘
```

### 12.2 Canvas-Based Rendering

Text overlays use the same Canvas rendering pipeline as captions, making them nearly free to implement:

```javascript
function renderTextOverlay(ctx, overlay, currentTime) {
  // Check if overlay should be visible
  if (currentTime < overlay.startTime || currentTime > overlay.endTime) return;
  
  const { style, position, text } = overlay;
  
  ctx.save();
  
  // Apply font settings
  ctx.font = `${style.fontWeight} ${style.fontSize}px ${style.fontFamily}`;
  ctx.textAlign = style.textAlign;
  ctx.textBaseline = 'top';
  
  // Calculate position (normalized 0-1 → pixel coordinates)
  const x = position.x * ctx.canvas.width;
  const y = position.y * ctx.canvas.height;
  
  // Draw background box
  if (style.backgroundColor) {
    const metrics = ctx.measureText(text);
    const padding = style.backgroundPadding || 8;
    ctx.fillStyle = style.backgroundColor;
    
    // Rounded rectangle
    roundRect(ctx,
      x - padding,
      y - padding,
      metrics.width + padding * 2,
      style.fontSize + padding * 2,
      4 // border radius
    );
    ctx.fill();
  }
  
  // Draw stroke/outline
  if (style.strokeColor && style.strokeWidth) {
    ctx.strokeStyle = style.strokeColor;
    ctx.lineWidth = style.strokeWidth;
    ctx.lineJoin = 'round';
    ctx.strokeText(text, x, y);
  }
  
  // Draw shadow
  if (style.shadowColor) {
    ctx.shadowColor = style.shadowColor;
    ctx.shadowBlur = style.shadowBlur || 4;
    ctx.shadowOffsetX = style.shadowOffsetX || 2;
    ctx.shadowOffsetY = style.shadowOffsetY || 2;
  }
  
  // Draw text
  ctx.fillStyle = style.color;
  ctx.fillText(text, x, y);
  
  ctx.restore();
}
```

### 12.3 Animations for Text Overlays

Five built-in animation types with smooth easing:

```javascript
const EASING = {
  easeOutCubic: (t) => 1 - Math.pow(1 - t, 3),
  easeOutBounce: (t) => {
    if (t < 1/2.75) return 7.5625 * t * t;
    if (t < 2/2.75) return 7.5625 * (t -= 1.5/2.75) * t + 0.75;
    if (t < 2.5/2.75) return 7.5625 * (t -= 2.25/2.75) * t + 0.9375;
    return 7.5625 * (t -= 2.625/2.75) * t + 0.984375;
  },
  easeInOutQuad: (t) => t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2,
};

function getAnimationTransform(animation, elapsed, duration) {
  const progress = Math.min(1, elapsed / (animation.duration || 0.3));
  
  switch (animation.type) {
    case 'fade':
      return { opacity: EASING.easeOutCubic(progress), translateY: 0, scale: 1 };
      
    case 'slide-up':
      const eased = EASING.easeOutCubic(progress);
      return { opacity: eased, translateY: (1 - eased) * 40, scale: 1 };
      
    case 'typewriter':
      // Reveal characters one at a time
      const charsToShow = Math.floor(progress * text.length);
      return { opacity: 1, translateY: 0, scale: 1, visibleChars: charsToShow };
      
    case 'pop':
      const popEased = EASING.easeOutBounce(progress);
      return { opacity: 1, translateY: 0, scale: popEased };
      
    default:
      return { opacity: 1, translateY: 0, scale: 1 };
  }
}
```

### 12.4 FFmpeg Export for Text Overlays

During export, text overlays are burned into the video using FFmpeg's `drawtext` filter:

```javascript
function generateDrawtextFilter(overlays) {
  return overlays.map(overlay => {
    const { style, position, text, startTime, endTime } = overlay;
    
    // FFmpeg drawtext parameters
    const params = [
      `text='${escapeFFmpegText(text)}'`,
      `fontfile=/tmp/fonts/${style.fontFamily}.ttf`,
      `fontsize=${style.fontSize}`,
      `fontcolor=${style.color}`,
      `x=${Math.round(position.x * 1920)}`,
      `y=${Math.round(position.y * 1080)}`,
      `enable='between(t,${startTime},${endTime})'`,
    ];
    
    if (style.strokeColor) {
      params.push(`borderw=${style.strokeWidth}`, `bordercolor=${style.strokeColor}`);
    }
    
    if (style.backgroundColor) {
      params.push(`box=1`, `boxcolor=${style.backgroundColor}`, `boxborderw=${style.backgroundPadding || 8}`);
    }
    
    if (style.shadowColor) {
      params.push(`shadowcolor=${style.shadowColor}`, `shadowx=${style.shadowOffsetX || 2}`, `shadowy=${style.shadowOffsetY || 2}`);
    }
    
    return `drawtext=${params.join(':')}`;
  }).join(',');
}

// Usage in FFmpeg command:
// -vf "drawtext=...,drawtext=..." (one per overlay)
```

### 12.5 Lower Third Templates

A lower third is a pre-configured text overlay combination:

```javascript
const lowerThirdPresets = {
  minimal: {
    nameStyle: { fontSize: 28, fontWeight: 700, color: '#FFFFFF' },
    titleStyle: { fontSize: 20, fontWeight: 400, color: '#CCCCCC' },
    barColor: 'rgba(0,0,0,0.6)',
    position: { x: 0.05, y: 0.82 },
  },
  
  bold: {
    nameStyle: { fontSize: 32, fontWeight: 800, color: '#FFFFFF' },
    titleStyle: { fontSize: 22, fontWeight: 500, color: '#FFDD00' },
    barColor: '#1a1a2e',
    position: { x: 0.05, y: 0.80 },
  },
  
  gradient: {
    nameStyle: { fontSize: 30, fontWeight: 700, color: '#FFFFFF' },
    titleStyle: { fontSize: 20, fontWeight: 400, color: '#E0E0E0' },
    barColor: 'linear-gradient(90deg, rgba(99,102,241,0.8), rgba(168,85,247,0.8))',
    position: { x: 0.05, y: 0.82 },
  },
};
```

---

## 13. Brand Templates & Style Persistence

### 13.1 What Brand Templates Solve

Creators who post regularly need **visual consistency** across all their clips. Without templates, they manually set caption style, colors, fonts, and overlay positions for every video. Templates save this configuration once and apply it with one click.

```
Without templates:                    With templates:
Video 1 → set fonts ✏️                Video 1 → set fonts → SAVE as "My Brand"
Video 2 → set fonts ✏️ (again)         Video 2 → one click "Apply My Brand" ✅
Video 3 → set fonts ✏️ (again)         Video 3 → one click "Apply My Brand" ✅
Video 4 → set fonts ✏️ (again)         Video 4 → one click "Apply My Brand" ✅
```

### 13.2 Template Data Structure

A brand template is a JSON object that captures the complete visual configuration:

```javascript
const exampleTemplate = {
  id: "tmpl_abc123",
  name: "My YouTube Brand",
  createdAt: "2026-02-24T10:00:00Z",
  updatedAt: "2026-02-24T10:00:00Z",
  
  captionStyle: {
    fontFamily: "Montserrat",
    fontSize: 52,
    fontWeight: 700,
    color: "#FFFFFF",
    strokeColor: "#000000",
    strokeWidth: 3,
    position: "bottom",
    animation: { type: "karaoke", highlightColor: "#FFD700" },
  },
  
  overlayDefaults: {
    fontFamily: "Inter",
    fontSize: 24,
    color: "#FFFFFF",
    backgroundColor: "rgba(0,0,0,0.6)",
  },
  
  colorPalette: {
    primary: "#6366F1",    // Indigo
    secondary: "#A855F7",  // Purple
    accent: "#FFD700",     // Gold
    text: "#FFFFFF",
    background: "#1a1a2e",
  },
  
  fonts: {
    heading: "Montserrat",
    body: "Inter",
    caption: "Montserrat",
  },
  
  logoConfig: {
    imageDataUrl: "data:image/png;base64,...",
    position: { x: 0.05, y: 0.05 },
    size: { width: 80, height: 80 },
    opacity: 0.8,
  },
};
```

### 13.3 Storage: IndexedDB + Supabase Sync

**Anonymous/free users**: templates stored entirely in IndexedDB (browser-local).

**Authenticated users**: templates sync to Supabase for cross-device access.

```javascript
// IndexedDB storage (always available)
await db.brandTemplates.put({
  id: template.id,
  name: template.name,
  config: template,
  updatedAt: new Date(),
});

// Supabase sync (if logged in)
if (user) {
  await supabase.from('brand_templates').upsert({
    id: template.id,
    user_id: user.id,
    name: template.name,
    config: template, // Stored as JSONB
    updated_at: new Date().toISOString(),
  });
}
```

**Conflict resolution:** last-write-wins with `updatedAt` comparison. If the server version is newer, it overwrites local. If local is newer, it pushes to server.

### 13.4 Applying a Template

Applying a template updates the editor state non-destructively — it changes styles but doesn't modify content:

```javascript
function applyTemplate(template, editorState) {
  return {
    ...editorState,
    captionStyle: template.captionStyle,
    textOverlays: editorState.textOverlays.map(overlay => ({
      ...overlay,
      style: { ...overlay.style, ...template.overlayDefaults },
    })),
    activeBrandTemplate: template,
  };
}
```

### 13.5 Import/Export for Sharing

Templates export as portable JSON files:

```javascript
function exportTemplate(template) {
  const exportable = { ...template };
  // Reduce logo size for sharing (compress base64)
  if (exportable.logoConfig?.imageDataUrl) {
    exportable.logoConfig.imageDataUrl = compressImage(
      exportable.logoConfig.imageDataUrl, 200 // max 200px wide
    );
  }
  return JSON.stringify(exportable, null, 2);
}

function importTemplate(jsonString) {
  const template = JSON.parse(jsonString);
  template.id = generateId(); // new ID to avoid conflicts
  template.createdAt = new Date();
  return template;
}
```

---

## 14. Why Browser Processing Has Hard Limits

### 14.1 The Fundamental Constraint: Browser Memory

**A browser tab is not a server.** This is the single most important technical reality for browser-based video processing.

```
┌─────────────────────────────────────────────────────────────────┐
│                    BROWSER TAB MEMORY LIMIT                      │
│                                                                   │
│  Chrome:     ~2,048 MB (2GB) per tab                             │
│  Firefox:    ~2,048 MB (varies by system RAM)                    │
│  Safari:     ~1,024 MB (1GB) — more restrictive                  │
│                                                                   │
│  This is a HARD LIMIT. Exceed it and the tab crashes.           │
│  No error message. No recovery. Just a dead tab.                │
└─────────────────────────────────────────────────────────────────┘
```

### 14.2 Where Memory Goes During Video Processing

Let's trace memory usage for processing a **400MB video file**:

```
Step 1: Load video into FFmpeg.wasm virtual filesystem
        Input file in MEMFS:              400 MB

Step 2: Load Whisper model for transcription
        Whisper 'base' model:             150 MB

Step 3: Extract audio for Whisper
        Audio as Float32Array:             40 MB (10 min @ 16kHz mono)

Step 4: Load MediaPipe for face detection
        MediaPipe model:                   10 MB

Step 5: FFmpeg processing buffers
        Internal buffers:                 120 MB (~30% of input)

Step 6: Output file in MEMFS
        Output video:                     350 MB

Step 7: React app, JS heap, browser overhead
        Application:                      100 MB

────────────────────────────────────────────────────
TOTAL:                                  ~1,170 MB

Browser limit:                           2,048 MB
Remaining headroom:                        878 MB
```

**At 400MB input, we're at ~57% of the limit.** This is safe.

### 14.3 What Happens at 1GB Input?

```
Input file in MEMFS:                    1,000 MB
Whisper model:                            150 MB
Audio Float32Array:                       100 MB (25 min)
MediaPipe:                                 10 MB
FFmpeg buffers:                           300 MB
Output file:                              900 MB
Application:                              100 MB
────────────────────────────────────────────────────
TOTAL:                                  ~2,560 MB

Browser limit:                           2,048 MB
OVERFLOW:                                  512 MB  ← TAB CRASHES
```

**At 1GB input, the browser tab WILL crash.** There is no workaround. This is why we have hard limits.

### 14.4 Why "Chunked Processing" Doesn't Solve This

A common suggestion: "Just process the video in chunks!"

**Why it doesn't work for our use case:**

1. **Subtitle burn-in requires full video access**
   ```bash
   # FFmpeg needs the entire video to apply subtitles
   ffmpeg -i full_video.mp4 -vf "ass=subtitles.ass" output.mp4
   ```
   You can't burn subtitles into chunk 3 without knowing where chunk 3 starts in the timeline.

2. **Transcript editing creates non-contiguous segments**
   ```
   User cuts: [0:00-2:30], [3:15-5:00], [7:20-10:00]
   
   To export, FFmpeg needs to:
   1. Write segment 1 to MEMFS
   2. Write segment 2 to MEMFS  
   3. Write segment 3 to MEMFS
   4. Concatenate all three
   
   All segments exist in memory simultaneously.
   ```

3. **Face tracking needs the full video**
   ```javascript
   // To calculate average face position for smart cropping,
   // we need to analyze the entire video
   for (let t = 0; t < video.duration; t += 0.2) {
     const faces = detectFaces(video, t);
     // Accumulate face positions
   }
   ```

### 14.5 The Honest Limits

Based on real-world testing across different hardware:

| Input Size | Input Duration | Success Rate | Notes |
|------------|---------------|--------------|-------|
| <200MB | <10 min | 99%+ | Safe on any modern browser |
| 200-400MB | 10-20 min | 95%+ | Safe on 8GB+ RAM machines |
| 400-700MB | 20-35 min | 70-80% | Risky on laptops, works on desktops |
| 700MB-1GB | 35-50 min | 30-50% | Frequent crashes |
| >1GB | >50 min | <10% | Almost always crashes |

**Our tier limits are set at the 95%+ success rate threshold:**
- Free: 200MB / 10 min
- Starter: 300MB / 15 min  
- Pro/Business: 400MB / 20 min

### 14.6 Why Not Just Increase the Limit?

"But my machine has 32GB RAM!"

**Browser memory limits are per-tab, not per-system.** Even on a 64GB machine:
- Chrome still limits each tab to ~2GB
- This is a browser security/stability feature
- It cannot be changed by the user or the website

**The only way to process larger files is to move processing to a server**, where you control the memory allocation.

---

## 15. Hybrid Processing: Browser + Cloud

### 15.1 The Hybrid Model

ClipFlow uses a **hybrid architecture** that plays to each environment's strengths:

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER DROPS VIDEO                         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │  Check file size and  │
                    │  duration             │
                    └───────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
            <400MB, <20min           >400MB or >20min
                    │                       │
                    ▼                       ▼
        ┌───────────────────┐   ┌───────────────────┐
        │  BROWSER MODE     │   │  CLOUD MODE       │
        │                   │   │                   │
        │  • Instant start  │   │  • Upload first   │
        │  • No upload      │   │  • Uses credits   │
        │  • Private        │   │  • Any file size  │
        │  • Free           │   │  • Fast processing│
        └───────────────────┘   └───────────────────┘
```

### 15.2 When to Use Each Mode

| Scenario | Mode | Why |
|----------|------|-----|
| 8-min YouTube explainer | Browser | Fast, free, private |
| 15-min tutorial | Browser | Within limits |
| 45-min webinar | Cloud | Too large for browser |
| 2-hour podcast | Cloud | Way too large for browser |
| Sensitive content | Browser | Never leaves device |
| Quick iteration | Browser | No upload wait |
| Batch processing | Cloud | More reliable for multiple large files |

### 15.3 Cloud Processing Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLOUD PROCESSING                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. UPLOAD (tus protocol — chunked, resumable)                   │
│     User → Cloudflare R2 / AWS S3                                │
│     Progress: 0% ────────────────────────────────── 100%         │
│                                                                   │
│  2. QUEUE                                                        │
│     Job submitted to processing queue (Redis/SQS)                │
│                                                                   │
│  3. PROCESS (Modal / Replicate serverless GPU)                   │
│     ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │
│     │  Download   │→ │  FFmpeg     │→ │  Whisper    │           │
│     │  from S3    │  │  (native)   │  │  (GPU)      │           │
│     └─────────────┘  └─────────────┘  └─────────────┘           │
│                                                                   │
│     Native FFmpeg: 1 min video in ~3-5 seconds                   │
│     (vs 60-90 seconds in browser WASM)                           │
│                                                                   │
│  4. UPLOAD RESULT                                                │
│     Processed video → S3 → signed download URL                   │
│                                                                   │
│  5. CLEANUP                                                      │
│     Auto-delete after 24 hours (privacy)                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 15.4 Cost Comparison: Browser vs Cloud

| Aspect | Browser Processing | Cloud Processing |
|--------|-------------------|------------------|
| **Our cost** | $0 | ~$0.002/min (GPU) |
| **User cost** | Free | 2 credits/min ($0.20/min) |
| **Speed** | 3-5x realtime | 0.3-0.5x realtime |
| **Reliability** | 95%+ under limits | 99.9%+ |
| **Privacy** | Video never leaves device | Video uploaded to servers |
| **Max file size** | 400MB | 5GB |
| **Max duration** | 20 min | 4 hours |

### 15.5 Why This Hybrid Model Works

1. **80% of short-form creators work with <20 min source videos**
   - YouTube educators, TikTok creators, course makers
   - They get instant, free, private processing

2. **20% need longer content occasionally**
   - Webinar recordings, podcast episodes
   - They pay credits for cloud processing
   - Still cheaper than OpusClip's subscription

3. **We're honest about what browsers can do**
   - No crashes, no broken promises
   - Clear messaging: "This video needs cloud processing"
   - User chooses: wait for upload, or use a different tool

### 15.6 The User Experience

**Video under limits (browser mode):**
```
1. Drop video                           [0 seconds]
2. Video loads, preview ready           [2-3 seconds]
3. Transcription runs                   [30-60 seconds for 10 min]
4. Edit with transcript, add captions   [user time]
5. Export                               [2-3 minutes for 10 min]
6. Download                             [instant]

Total wait time: ~4 minutes for a 10-min video
```

**Video over limits (cloud mode):**
```
1. Drop video                           [0 seconds]
2. "This video is too large for browser. Upload to cloud?"
3. User confirms, upload starts         [2-5 min for 1GB on fast connection]
4. Server processes                     [30-60 seconds]
5. Edit with transcript, add captions   [user time]
6. Export                               [30-60 seconds on server]
7. Download                             [1-2 min for result]

Total wait time: ~5-8 minutes for a 1GB video
```

**Compared to OpusClip (always server):**
```
1. Drop video                           [0 seconds]
2. Upload                               [2-5 min for 1GB]
3. Wait in queue                        [0-10 min depending on load]
4. Server processes                     [2-5 min]
5. Edit                                 [user time]
6. Export                               [1-2 min]
7. Download                             [1-2 min]

Total wait time: ~10-20 minutes for a 1GB video
```

**ClipFlow is faster for short videos (browser) and competitive for long videos (cloud).**

---

## 16. Glossary

| Term | Definition |
|------|------------|
| **AAC** | Advanced Audio Coding - common audio codec |
| **ASS** | Advanced SubStation Alpha - subtitle format with styling |
| **Bitrate** | Data per second in video/audio stream |
| **Codec** | Coder-decoder algorithm for compression |
| **Concat Demuxer** | FFmpeg feature that joins multiple video files sequentially |
| **Container** | File format holding video/audio streams |
| **CRF** | Constant Rate Factor - quality setting for encoding |
| **Crossfade** | Short audio fade at edit points to prevent pops/clicks |
| **Cut List** | Ordered set of time ranges to exclude from the final export |
| **dB (Decibel)** | Logarithmic unit for measuring audio loudness |
| **FFmpeg** | Multimedia framework for video processing |
| **Filler Word** | Verbal tic (um, uh, like, you know) that adds no meaning |
| **FPS** | Frames per second |
| **H.264** | Most common video codec (AVC) |
| **H.265** | Newer, more efficient codec (HEVC) |
| **I-frame** | Keyframe - complete image, not dependent on other frames |
| **Keyframe** | Frame that can be decoded independently |
| **Lower Third** | Text overlay at the bottom of the screen showing name/title |
| **MediaPipe** | Google's ML framework for vision tasks |
| **Muxing** | Combining video and audio into container |
| **P-frame** | Predicted frame - stores differences from previous |
| **Resolution** | Pixel dimensions (width × height) |
| **RMS** | Root Mean Square - standard measure of audio signal energy |
| **SRT** | SubRip - simple subtitle format |
| **Stopwords** | Common words (the, a, is, etc.) filtered out in text analysis |
| **TF-IDF** | Term Frequency–Inverse Document Frequency - text relevance scoring |
| **Transcoding** | Converting from one codec to another |
| **VP9** | Google's open video codec |
| **WASM** | WebAssembly - binary format for browser |
| **Web Audio API** | Browser API for audio analysis, processing, and synthesis |
| **WebCodecs** | Browser API for codec access |
| **WebVTT** | Web Video Text Tracks - web subtitle format |
| **Whisper** | OpenAI's speech recognition model |
| **WPM** | Words Per Minute - measure of speech pace |

---

## Further Reading

### Official Documentation
- [WebCodecs API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API)
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
- [FFmpeg.wasm](https://ffmpegwasm.netlify.app/)
- [MediaPipe Face Detection](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector)
- [Whisper Web](https://whisperweb.dev/)

### Tutorials
- [Build a Video Editor with FFmpeg.wasm](https://www.blog.brightcoding.dev/2026/01/09/build-a-viral-video-editor-in-your-browser-next-js-ffmpeg-wasm-complete-guide-2026/)
- [Video Processing with WebCodecs (Chrome)](https://developer.chrome.com/docs/web-platform/best-practices/webcodecs)

### Libraries
- [@ffmpeg/ffmpeg](https://www.npmjs.com/package/@ffmpeg/ffmpeg)
- [@remotion/whisper-web](https://www.npmjs.com/package/@remotion/whisper-web)
- [@mediapipe/tasks-vision](https://www.npmjs.com/package/@mediapipe/tasks-vision)

### New Feature References
- [Web Audio API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
- [AnalyserNode (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/AnalyserNode)
- [FFmpeg Concat Demuxer](https://ffmpeg.org/ffmpeg-formats.html#concat)
- [FFmpeg drawtext filter](https://ffmpeg.org/ffmpeg-filters.html#drawtext)
- [TF-IDF Explained](https://en.wikipedia.org/wiki/Tf%E2%80%93idf)
- [IndexedDB API (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
- [Dexie.js](https://dexie.org/) — IndexedDB wrapper used for brand template storage

---

*Document maintained by: Engineering Team*  
*Last updated: February 24, 2026*
