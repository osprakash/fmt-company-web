# Product Requirements Document (PRD)
## ClipFlow.ai - The Short-Form Video Editor That Runs in Your Browser

**Version:** 4.0  
**Date:** February 28, 2026  
**Status:** Draft (Updated with realistic browser limits and hybrid cloud model)

---

## 1. Executive Summary

### 1.1 Product Vision
ClipFlow.ai is a **short-form video editor** optimized for creators who work with videos under 20 minutes. Our core differentiator is **client-side processing** using modern browser APIs (WebCodecs, FFmpeg.wasm, Whisper Web), enabling instant editing with zero upload wait times and complete privacy — your video never leaves your device.

**What ClipFlow IS:** The fastest way to edit 5-20 minute videos into polished shorts with captions, reframing, and transcript-based editing.

**What ClipFlow is NOT:** A long-form podcast/stream clipper. For 1-2 hour videos, use OpusClip or Descript. We're not competing there.

### 1.2 Problem Statement
Short-form creators (educators, explainer channels, TikTokers) need to:
1. Add captions to their 5-15 minute recordings
2. Reframe horizontal videos to vertical (9:16)
3. Trim and clean up their content (remove filler words, awkward pauses)
4. Do this quickly, without uploading to a server and waiting in a queue

Current solutions either:
- **Require upload + queue** (OpusClip, Descript) — slow for quick edits
- **Are desktop apps** (Premiere, DaVinci) — overkill for simple tasks
- **Are mobile-only** (CapCut) — limited for serious editing

### 1.3 Solution
A **browser-native editor** with a **hybrid cloud fallback**:
- **Browser Processing (Default)**: Videos under 20 min / 400MB process entirely in your browser. No upload, no wait, no privacy concerns.
- **Cloud Processing (Fallback)**: Videos over 20 min / 400MB can optionally upload to our servers for processing (uses credits).
- **"Try everything, pay to produce"**: All editing features free, but exports gated (3/month, 720p, watermarked on free tier).

**Pricing philosophy**: Gate outputs, not inputs. Browser processing costs us nothing. Every free user is a future paying customer, not a cost center.

### 1.4 Target Use Cases (What We're Optimized For)
| Use Case | Typical Video Length | Why ClipFlow Wins |
|----------|---------------------|-------------------|
| YouTube educator making Shorts from explainer | 8-15 min | Instant processing, transcript editing |
| TikTok creator editing raw recordings | 3-10 min | No upload wait, privacy |
| Course creator adding captions | 10-20 min | Batch caption styling, brand templates |
| Social media manager repurposing clips | 5-15 min | Multi-ratio export, fast turnaround |

### 1.5 Non-Target Use Cases (Use Something Else)
| Use Case | Why Not ClipFlow | Better Alternative |
|----------|-----------------|-------------------|
| Clipping 2-hour podcasts | Browser can't handle 1-2GB files reliably | OpusClip, Descript |
| Finding viral moments in livestreams | Requires server-side AI analysis | OpusClip, Eklipse |
| Professional video production | Need timeline, effects, color grading | Premiere, DaVinci, CapCut |

---

## 2. Market Analysis

### 2.1 Competitive Landscape

**We compete in the "quick video editing" space, NOT the "AI clip discovery" space.**

| Competitor | Pricing | Processing | Best For | Why We're Different |
|------------|---------|------------|----------|---------------------|
| **CapCut (Web)** | Free | Server-side | Mobile-first creators | We have transcript editing, they don't |
| **VEED** | $0-24/mo | Server-side | All-in-one editing | We're faster (no upload), they're more feature-rich |
| **Kapwing** | $0-24/mo | Server-side | Teams, memes | We focus on short-form optimization |
| **Canva Video** | $0-15/mo | Server-side | Design-first creators | We have better video-specific features |
| **Descript** | $12-24/mo | Server-side | Podcasters, long-form | We're simpler and cheaper for short videos |

**Adjacent competitors (different use case):**
| Competitor | Their Focus | Why We Don't Compete Directly |
|------------|-------------|------------------------------|
| **OpusClip** | AI clip discovery from 1-2hr videos | We can't process 1-2GB files in browser; they can't match our speed for short videos |
| **Premiere/DaVinci** | Professional editing | Overkill for our users; we're for quick edits |
| **CapCut (Desktop)** | Full-featured mobile export | We're browser-based, they require install |

### 2.2 Our Unique Position
1. **Instant processing**: No upload wait. Drop a 15-min video, start editing in seconds.
2. **Privacy by default**: Video never leaves your device (for browser-processed content).
3. **Transcript-first editing**: Edit video by editing text — faster than timeline scrubbing.
4. **Free tier that actually works**: All features available, just output-gated.

### 2.3 Target Users
1. **Primary**: Short-form creators (YouTube Shorts, TikTok, Reels) working with 5-20 min source videos
2. **Secondary**: Educators and course creators adding captions to explainer videos
3. **Tertiary**: Social media managers doing quick turnaround edits

### 2.4 Explicitly NOT Our Target
- Podcasters with 1-2 hour episodes (use OpusClip/Descript)
- Livestreamers clipping 4-hour streams (use Eklipse/OpusClip)
- Professional editors needing timeline precision (use Premiere/DaVinci)

---

## 3. Core Features (MVP)

### 3.1 Feature 1: Smart Captions (Browser-Based)

**Description**: Automatically transcribe video audio and generate stylized, animated captions.

**User Story**: *As a content creator, I want to add engaging captions to my videos so that viewers can watch without sound and stay engaged.*

**Functional Requirements**:
| ID | Requirement | Priority |
|----|-------------|----------|
| SC-01 | Transcribe audio using Whisper Web (client-side) | P0 |
| SC-02 | Support 25+ languages with auto-detection | P0 |
| SC-03 | Word-level timestamp accuracy for animations | P0 |
| SC-04 | 10+ caption style presets (karaoke, bounce, glow, etc.) | P0 |
| SC-05 | Custom font, color, position, and animation settings | P1 |
| SC-06 | Manual transcript editing with re-sync | P1 |
| SC-07 | Emoji auto-insertion based on sentiment | P2 |
| SC-08 | Keyword highlighting (auto-detect important terms) | P2 |

**Technical Approach**:
- Whisper Web (WebGPU/WASM) for transcription - 98% accuracy, fully client-side
- Web Animations API for smooth caption animations
- Canvas/WebGL overlay for caption rendering during export

**Success Metrics**:
- Transcription accuracy ≥95%
- Processing speed: 1 minute of video in <30 seconds
- User satisfaction: 4.5+ stars on caption quality

---

### 3.2 Feature 2: Aspect Ratio Reframing (Browser-Based)

**Description**: Convert horizontal (16:9) videos to vertical (9:16) or square (1:1) formats with intelligent subject tracking.

**User Story**: *As a content creator, I want to convert my YouTube videos to TikTok format so that the main subject stays centered and visible.*

**Functional Requirements**:
| ID | Requirement | Priority |
|----|-------------|----------|
| AR-01 | One-click conversion: 16:9 → 9:16, 1:1, 4:5 | P0 |
| AR-02 | Face detection for automatic subject centering | P0 |
| AR-03 | Manual crop position adjustment with preview | P0 |
| AR-04 | Keyframe-based pan/zoom for dynamic reframing | P1 |
| AR-05 | Multi-face detection with priority selection | P1 |
| AR-06 | Object tracking for non-face subjects | P2 |
| AR-07 | Safe zone guides for platform-specific UI overlays | P2 |

**Technical Approach**:
- MediaPipe Face Detection (TensorFlow.js) for subject tracking - runs in browser
- FFmpeg.wasm for video cropping and encoding
- Canvas-based preview with real-time crop adjustment

**Success Metrics**:
- Face detection accuracy ≥90%
- Export quality: Up to 1080p without visible artifacts
- Processing speed: 1 minute of video in <2 minutes

---

### 3.3 Feature 3: Video Trimming & Export (Browser-Based)

**Description**: Trim videos to specific segments and export in multiple formats/qualities.

**User Story**: *As a content creator, I want to quickly trim my video to the best 30-60 second segment and export it ready for upload.*

**Functional Requirements**:
| ID | Requirement | Priority |
|----|-------------|----------|
| VT-01 | Timeline-based trim with frame-accurate cutting | P0 |
| VT-02 | Waveform visualization for audio-guided trimming | P0 |
| VT-03 | Export to MP4 (H.264) up to 1080p | P0 |
| VT-04 | Multiple quality presets (draft, standard, high) | P0 |
| VT-05 | Batch export to multiple aspect ratios | P1 |
| VT-06 | Export to WebM, MOV formats | P1 |
| VT-07 | Custom bitrate and codec settings | P2 |
| VT-08 | Direct upload to social platforms (OAuth) | P2 |

**Technical Approach**:
- WebCodecs API for efficient frame-level access
- FFmpeg.wasm for encoding and muxing
- IndexedDB for caching large video files during editing

**Success Metrics**:
- Export success rate ≥99%
- No quality loss at same resolution/bitrate
- Export speed: 1 minute of 1080p video in <3 minutes

---

### 3.4 Feature 4: Transcript-Based Editing (Browser-Based)

**Description**: Edit video by editing its transcript. Delete words or sentences to remove corresponding video segments. Click any word to jump to that timestamp. The fastest way to cut a video without touching a timeline.

**User Story**: *As a content creator, I want to edit my video by reading and editing its transcript so that I can cut sections faster than scrubbing through a timeline.*

**Functional Requirements**:
| ID | Requirement | Priority |
|----|-------------|----------|
| TE-01 | Display full transcript synced with video playback (highlighted current word) | P0 |
| TE-02 | Click any word to seek video to that timestamp | P0 |
| TE-03 | Select and delete text to create cut points (removes corresponding video) | P0 |
| TE-04 | Visual diff showing deleted sections (strikethrough) with undo support | P0 |
| TE-05 | Automatic gap removal when segments are deleted | P0 |
| TE-06 | Split/merge segments by inserting or removing line breaks | P1 |
| TE-07 | Search within transcript (Ctrl+F) to find specific words/phrases | P1 |
| TE-08 | Export edited transcript as SRT/VTT file | P2 |

**Technical Approach**:
- Built on top of Whisper Web transcript (word-level timestamps from Feature 1)
- Editing operations map text ranges to time ranges via the existing `TranscriptWord[]` data
- Cut list maintained as a set of `{start, end}` time ranges to exclude
- FFmpeg.wasm concat demuxer reassembles kept segments during export

**Success Metrics**:
- 50% faster editing time vs timeline-only trimming (measured via A/B test)
- Transcript click-to-seek latency <100ms
- Zero audio glitches at cut points

---

### 3.5 Feature 5: Filler Word & Silence Detection/Removal (Browser-Based)

**Description**: Automatically detect filler words ("um," "uh," "like," "you know"), awkward pauses, and long silences. One-click removal with visual markers on the timeline.

**User Story**: *As a content creator, I want to automatically find and remove filler words and dead air from my video so that it sounds polished without hours of manual editing.*

**Functional Requirements**:
| ID | Requirement | Priority |
|----|-------------|----------|
| FW-01 | Detect common filler words from Whisper transcript (um, uh, ah, like, you know, so, basically, literally) | P0 |
| FW-02 | Detect silence gaps >0.5s using audio waveform analysis | P0 |
| FW-03 | Visual markers on timeline for all detected fillers and silences | P0 |
| FW-04 | One-click "Remove All Fillers" with preview before applying | P0 |
| FW-05 | Selective removal — toggle individual filler/silence markers on/off | P1 |
| FW-06 | Configurable sensitivity (silence threshold in dB, minimum gap duration) | P1 |
| FW-07 | Custom filler word list (add domain-specific words) | P2 |
| FW-08 | Statistics panel showing total filler count, silence duration, time saved | P2 |

**Technical Approach**:
- Filler detection: regex/keyword matching against `TranscriptWord[]` from Whisper
- Silence detection: Web Audio API `AnalyserNode` RMS energy analysis on decoded audio
- Removal: generates a cut list (same format as transcript editing) that excludes filler/silence ranges
- Combined with transcript editing — removed sections shown as strikethrough in transcript view

**Success Metrics**:
- Filler word detection recall ≥90% (catches most fillers)
- Filler word detection precision ≥85% (minimal false positives)
- Average time saved per video: 15-30 seconds of content per minute of raw footage

---

### 3.6 Feature 6: Text Overlay & Titles (Browser-Based)

**Description**: Add customizable text overlays, titles, and lower-third graphics to videos. Position anywhere, animate, and set display duration.

**User Story**: *As a content creator, I want to add titles, callouts, and lower thirds to my clips so that I can add context and visual interest without a separate editing tool.*

**Functional Requirements**:
| ID | Requirement | Priority |
|----|-------------|----------|
| TO-01 | Add text overlay at any position on the video canvas (drag to place) | P0 |
| TO-02 | Set start time and end time for each overlay (when it appears/disappears) | P0 |
| TO-03 | Customize font, size, color, background, stroke, shadow per overlay | P0 |
| TO-04 | 5+ text animation presets (fade in/out, slide up, typewriter, pop, none) | P0 |
| TO-05 | Multiple simultaneous overlays supported | P1 |
| TO-06 | Lower-third templates (name + title bar presets) | P1 |
| TO-07 | Emoji and icon insertion in overlays | P2 |
| TO-08 | Duplicate and adjust overlays across the timeline | P2 |

**Technical Approach**:
- Same Canvas/WebGL rendering pipeline as caption renderer (shared infrastructure)
- Overlays stored as an array of `TextOverlay` objects in project state
- During export, overlays composited via FFmpeg `drawtext` filter chain or Canvas frame-by-frame rendering
- Real-time preview via Canvas overlay on the video player

**Success Metrics**:
- Overlay placement latency <50ms (drag feels instant)
- Zero visual artifacts in exported overlays
- 30% of users add at least one text overlay per project

---

### 3.7 Feature 7: Brand Templates (Browser-Based)

**Description**: Save and reuse complete style configurations — caption style, text overlay defaults, colors, fonts, logo placement — as reusable brand templates. Unlimited templates on all tiers.

**User Story**: *As a content creator who posts regularly, I want to save my brand's fonts, colors, and caption style as a template so that every clip I export looks consistent without manual setup.*

**Functional Requirements**:
| ID | Requirement | Priority |
|----|-------------|----------|
| BT-01 | Save current editor style configuration as a named template | P0 |
| BT-02 | Template includes: caption style, text overlay defaults, color palette, font selections | P0 |
| BT-03 | Apply template with one click to any new or existing project | P0 |
| BT-04 | Unlimited templates on all tiers (including free) | P0 |
| BT-05 | Import/export templates as JSON files (share with team or across devices) | P1 |
| BT-06 | Template preview thumbnail showing caption + overlay style | P1 |
| BT-07 | Logo/watermark placement saved in template (image upload, position, opacity) | P2 |
| BT-08 | Community template gallery (browse and use templates shared by others) | P2 |

**Technical Approach**:
- Templates stored as JSON in IndexedDB (free tier) and Supabase (logged-in users)
- Template schema: `{ name, captionStyle, overlayDefaults, colorPalette, fonts, logoConfig }`
- No server cost — purely client-side state persistence
- Sync across devices via Supabase for authenticated users

**Success Metrics**:
- 40% of returning users create at least one brand template
- Template apply time <200ms
- Average templates per active user: 2-3

---

### 3.8 Feature 8: Cloud Processing Fallback (Hybrid Model)

**Description**: For videos that exceed browser processing limits (>20 min or >400MB), offer optional upload to ClipFlow servers for processing. This enables the full use case for users with longer content while maintaining browser-first as the default.

**User Story**: *As a content creator with a 45-minute video, I want to still use ClipFlow's editing features even though my video is too large for browser processing.*

**Functional Requirements**:
| ID | Requirement | Priority |
|----|-------------|----------|
| CP-01 | Detect when video exceeds browser limits (duration or file size) | P0 |
| CP-02 | Show clear message: "This video is too large for browser processing. Upload to cloud?" | P0 |
| CP-03 | Display estimated credit cost before upload (2 credits/min of input video) | P0 |
| CP-04 | Secure upload with progress indicator and resume capability | P0 |
| CP-05 | Server-side processing using same feature set (captions, reframe, trim, filler removal) | P0 |
| CP-06 | Download processed video or export directly | P0 |
| CP-07 | Auto-delete uploaded files after 24 hours (privacy) | P1 |
| CP-08 | Option to keep in cloud storage (Pro/Business tiers) | P2 |

**Technical Approach**:
- Upload to Cloudflare R2 or AWS S3 (chunked, resumable via tus protocol)
- Processing via Modal/Replicate serverless GPUs (FFmpeg + Whisper)
- Same editing UI — user doesn't see difference except upload step
- Results stored temporarily, download link provided

**Success Metrics**:
- Upload success rate ≥99%
- Processing time: 1 min of video in <30 seconds (server-side)
- User satisfaction with cloud fallback ≥4/5

**Credit Cost**:
- 2 credits per minute of input video (covers GPU compute + storage)
- Business tier: cloud processing included (no credit cost)

---

## 4. Future Features (Post-MVP)

### 4.1 Phase 2: Server-Side AI Features (Paid Tiers)
| Feature | Description | Credit Cost |
|---------|-------------|-------------|
| AI Clip Detection | Automatically find viral-worthy moments | 5 credits/min |
| AI B-Roll | Generate relevant stock footage overlays | 10 credits/clip |
| Voice Enhancement | Remove background noise, enhance clarity | 3 credits/min |
| Auto-Chapters | Generate chapter markers from content | 2 credits/video |

### 4.2 Phase 3: Platform Expansion
- Team workspaces with collaboration
- Shared asset libraries (cloud-stored logos, intros, music)
- Social media scheduling and direct posting
- Analytics dashboard
- Community template gallery

---

## 5. Pricing Model

### 5.1 Pricing Philosophy: "Try Everything, Pay to Produce"

The free tier unlocks **all editing features** — users experience the full power of the editor before hitting any limit. Conversion happens at the moment of highest perceived value: export time, after the user has invested time editing.

**Why output-gating (not minute-gating or feature-gating)?**
- **Not minute-gating**: OpusClip's most hated feature. Opaque, burns unpredictably. We don't copy their worst trait.
- **Not feature-gating**: If users can't try transcript editing for free, they'll never discover it's great. Let them fall in love first.
- **Output-gating**: Transparent, predictable. "3 exports this month, upgrade for more." The user just spent 20 minutes editing and wants their clip — that's peak willingness to pay.

**Why this still works competitively**: OpusClip's free tier gives 60 min of processing with stripped features and watermark. ClipFlow's free tier gives ALL features with 3 full exports. A user who tries both will spend more time in ClipFlow and be further down the funnel.

### 5.2 Pricing Tiers

| | **Free** | **Starter** | **Pro** | **Business** |
|--|----------|-------------|---------|--------------|
| **Price** | $0/mo | $9/mo | $19/mo | $49/mo |
| | | | | |
| **Editing Features** | | | | |
| Smart Captions (all styles) | All | All | All | All |
| Transcript-Based Editing | Yes | Yes | Yes | Yes |
| Filler Word & Silence Removal | Yes | Yes | Yes | Yes |
| Aspect Ratio Reframing | Yes | Yes | Yes | Yes |
| Text Overlays & Titles | Yes | Yes | Yes | Yes |
| Timeline Trim + Waveform | Yes | Yes | Yes | Yes |
| | | | | |
| **Output Limits** | | | | |
| Exports per month | **3** | **25** | **Unlimited** | **Unlimited** |
| Watermark | Yes | **No** | No | No |
| Max export quality | **720p** | **1080p** | **1080p** | **4K** |
| Batch export (multi-ratio) | No | Yes | Yes | Yes |
| | | | | |
| **Browser Processing Limits** | | | | |
| Max input duration (browser) | **10 min** | **15 min** | **20 min** | **20 min** |
| Max file size (browser) | **200MB** | **300MB** | **400MB** | **400MB** |
| | | | | |
| **Cloud Processing (Larger Files)** | | | | |
| Cloud processing available | **No** | **Yes** | **Yes** | **Yes** |
| Max input duration (cloud) | — | **60 min** | **2 hours** | **4 hours** |
| Max file size (cloud) | — | **1GB** | **2GB** | **5GB** |
| Cloud processing cost | — | **2 credits/min** | **2 credits/min** | **Included** |
| | | | | |
| **Credits & AI Features** | | | | |
| Credits/month | 0 | **50** | **300** | **1000** |
| Cloud processing | — | Yes (uses credits) | Yes (uses credits) | Yes (included) |
| AI Clip Detection (Phase 2) | — | — | Yes | Yes |
| AI B-Roll Generation (Phase 2) | — | — | Yes | Yes |
| AI Voice Enhancement (Phase 2) | — | — | Yes | Yes |
| | | | | |
| **Brand & Storage** | | | | |
| Brand Templates | **1** | **10** | **Unlimited** | Unlimited |
| Template cloud sync | No | Yes | Yes | Yes |
| Cloud storage | — | — | 10GB | 50GB |
| | | | | |
| **Platform** | | | | |
| Team seats | — | — | — | 3 |
| API access | — | — | — | Yes |
| Priority support | — | Yes | Yes | Yes |

### 5.3 Conversion Triggers (Why Users Upgrade)

| Trigger | Free → Starter | Starter → Pro |
|---------|---------------|---------------|
| **Primary** | Hits 3 export/month limit (posts >1x/week) | Needs unlimited exports (daily posting) |
| **Secondary** | Needs 1080p (TikTok/Reels look bad at 720p) | Wants AI features (clip detection, B-roll) |
| **Tertiary** | Wants watermark removed for professional look | Needs cloud storage for team/cross-device |
| **Timeline** | Within 1-2 weeks of regular use | Within 1-2 months of Starter use |

**Key insight — the 720p gate**: TikTok and Instagram Reels are natively 1080p. A 720p export looks noticeably soft on mobile. Serious creators will upgrade for quality alone, even before hitting the export limit.

### 5.4 Revenue Projections (Revised)

**Assumptions**: 8-12% free-to-paid conversion (industry avg for freemium with strong product, vs 3-5% for weak free tiers)

| Month | Users | MAE | Starter (8%) | Pro (2%) | MRR | ARR |
|-------|-------|-----|-------------|----------|-----|-----|
| 3 | 10,000 | 3,000 | 240 x $9 | 60 x $19 | **$3,300** | $39,600 |
| 6 | 50,000 | 15,000 | 1,200 x $9 | 300 x $19 | **$16,500** | $198,000 |
| 12 | 150,000 | 50,000 | 4,000 x $9 | 1,000 x $19 | **$55,000** | $660,000 |
| 18 | 300,000 | 100,000 | 8,000 x $9 | 2,000 x $19 | **$110,000** | **$1,320,000** |

**Path to $1M ARR: Month 15-18** (with organic growth, no paid acquisition).

With Business tier and credit top-ups, actual ARR likely 20-30% higher.

### 5.5 Credit System (Phase 2 — Pro & Business Only)

**Credit Definition**: 1 credit = 1 minute of server-side AI processing

| Operation | Credits | Available On |
|-----------|---------|-------------|
| AI Clip Detection (full) | 5/min | Pro, Business |
| AI B-Roll Generation | 10/clip | Pro, Business |
| AI Voice Enhancement | 3/min | Pro, Business |
| 4K Export | 2/min | Business (included), Pro (credits) |

### 5.6 Credit Top-Up (Pro & Business)
- 50 credits: $5
- 150 credits: $12 (20% bonus)
- 500 credits: $35 (30% bonus)

---

## 6. User Experience

### 6.1 User Flow (MVP)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIPFLOW.AI                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. UPLOAD          2. ANALYZE           3. EDIT             4. EXPORT      │
│  ┌──────────┐      ┌──────────────┐     ┌──────────────┐   ┌──────────┐   │
│  │ Drag &   │ ──▶  │ • Transcribe │ ──▶ │ • Transcript │──▶│ • Choose │   │
│  │ Drop     │      │ • Detect     │     │   editing    │   │   format │   │
│  │ Video    │      │   fillers    │     │ • Trim       │   │ • Apply  │   │
│  └──────────┘      │ • Suggest    │     │ • Captions   │   │   brand  │   │
│                     │   clips     │     │ • Reframe    │   │ • Export │   │
│                     └──────────────┘     │ • Text       │   └──────────┘   │
│                                          │   overlays   │                   │
│                                          │ • Remove     │                   │
│                                          │   fillers    │                   │
│                                          └──────────────┘                   │
│                                                                              │
│  Supported: MP4, WebM, MOV (up to 500MB free, 2GB paid)                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Key UX Principles
1. **Zero friction start**: No signup required for first export (watermarked)
2. **Progressive disclosure**: Simple interface with advanced options tucked away
3. **Real-time preview**: All edits visible instantly before export
4. **Transparent processing**: Clear progress indicators showing browser activity

### 6.3 Onboarding Flow (Optimized for Conversion)
1. Land on homepage → See demo video showing transcript editing + captions
2. Upload first video (**no account needed** — zero friction)
3. Auto-transcribe → See smart clip suggestions (top 3) + filler word highlights
4. Pick a suggested clip or trim manually, apply captions + reframe + text overlays
5. Preview result in full quality
6. **Export gate**: First export is free (720p, watermarked). Show side-by-side preview of watermarked vs clean.
7. Prompt to create account to use remaining 2 free exports this month
8. After 3 exports: upgrade modal showing Starter at $9/mo with "Remove watermark + 1080p + 25 exports"
9. Prompt to save brand template for future use (1 template free, more on Starter)

**Conversion psychology**: The user has already invested 10-20 minutes editing. The video looks great in preview. The only thing between them and a professional-looking clip is $9/mo. This is the peak of willingness to pay.

---

## 7. Technical Constraints

### 7.1 Browser Requirements
- Chrome 94+, Firefox 88+, Safari 15.2+, Edge 94+
- WebGPU support recommended for faster transcription
- Minimum 4GB RAM, 8GB recommended
- Stable internet for initial model downloads (~150MB including Whisper + face detection models)

### 7.2 Video Limitations — Browser vs Cloud Processing

**Browser Processing (Default — instant, private):**
| Constraint | Free | Starter | Pro | Business |
|------------|------|---------|-----|----------|
| Max file size | 200MB | 300MB | 400MB | 400MB |
| Max input duration | 10 min | 15 min | 20 min | 20 min |
| Max export resolution | 720p | 1080p | 1080p | 1080p |
| Exports per month | 3 | 25 | Unlimited | Unlimited |
| Processing location | Browser | Browser | Browser | Browser |

**Cloud Processing (Fallback — for larger files, uses credits):**
| Constraint | Free | Starter | Pro | Business |
|------------|------|---------|-----|----------|
| Max file size | N/A | 1GB | 2GB | 5GB |
| Max input duration | N/A | 60 min | 2 hours | 4 hours |
| Max export resolution | N/A | 1080p | 4K | 4K |
| Credit cost | N/A | 2/min | 2/min | Included |
| Processing location | N/A | Server | Server | Server |

**Why these limits?**
- Browser tabs have ~2GB memory limit
- A 400MB video + Whisper model + FFmpeg buffers + output = ~1.5GB (safe margin)
- Videos over 400MB risk browser crashes on average hardware
- Cloud fallback enables longer videos without compromising browser reliability

### 7.3 Known Limitations
- Safari has limited WebCodecs support (fallback to FFmpeg.wasm only)
- Mobile browsers not supported in MVP (desktop-first)
- Browser processing requires 8GB+ system RAM for best experience
- Videos over 20 min / 400MB require cloud processing (upload + credits)
- Cloud processing adds 2-5 min upload time depending on connection

---

## 8. Success Metrics

### 8.1 North Star Metric
**Monthly Active Exporters (MAE)**: Users who successfully export at least one video per month

### 8.2 Key Performance Indicators

| Metric | Target (Month 3) | Target (Month 6) | Target (Month 12) |
|--------|------------------|------------------|--------------------|
| Registered users | 10,000 | 50,000 | 150,000 |
| Monthly Active Exporters | 3,000 | 15,000 | 50,000 |
| Free → Paid conversion | 5% | 8% | 10% |
| Starter subscribers | 150 | 1,200 | 4,000 |
| Pro subscribers | 40 | 300 | 1,000 |
| MRR | $2,100 | $16,500 | $55,000 |
| Export success rate | 95% | 99% | 99.5% |
| Avg exports/free user/month | 2.5 | 2.8 | 3.0 (at limit) |
| Avg exports/paid user/month | 12 | 18 | 22 |
| NPS Score | 40 | 50 | 55 |
| Export limit hit rate (free) | 30% | 45% | 55% |

### 8.3 Tracking Plan
- Mixpanel/Amplitude for user analytics
- Sentry for error tracking
- Custom metrics for processing performance

---

## 9. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Browser API compatibility issues | High | Medium | Feature detection + graceful fallbacks |
| Memory crashes on large files | High | Low | **Strict file limits (400MB max browser), cloud fallback for larger** |
| Users try to process videos too large for browser | Medium | High | **Clear messaging + automatic cloud fallback offer** |
| Whisper model download failures | Medium | Low | CDN redundancy, offline caching |
| Competition copies browser approach | Medium | Medium | First-mover advantage, rapid iteration |
| Users expect long-form clipping (like OpusClip) | Medium | High | **Clear positioning: "for 5-20 min videos" in all marketing** |
| Filler detection false positives (e.g., "like" used correctly) | Low | Medium | Confidence threshold + easy undo; show preview before applying |
| Cloud processing costs exceed projections | Medium | Low | Monitor GPU costs, adjust credit pricing if needed |
| Users frustrated by browser limits | Medium | Medium | **Cloud fallback available on all paid tiers** |

---

## 10. Launch Plan

### 10.1 MVP Scope
**Core Editing (Browser-Based):**
- Smart Captions (10+ style presets, 25+ languages)
- Aspect Ratio Reframe (face detection, 9:16 / 1:1 / 4:5)
- Video Trim & Export (1080p MP4, waveform visualization)
- Transcript-Based Editing (click-to-seek, delete text to cut video)
- Filler Word & Silence Removal (auto-detect, one-click remove)
- Brand Templates (save/load style configurations)

**Hybrid Cloud (For Larger Videos):**
- Cloud processing fallback for videos >20 min / >400MB
- Credit-based pricing for cloud processing (2 credits/min)
- Secure upload with progress indicator

**Platform:**
- User accounts (email + Google OAuth)
- Credit balance and usage tracking
- Basic analytics dashboard

**NOT in MVP (Deferred):**
- Text Overlays & Titles (deferred to Phase 1.5 — export pipeline complexity)
- Smart Clip Suggestions (deferred to Phase 2 — requires server-side ML for quality)

### 10.2 Launch Phases
1. **Alpha** (Internal): Core team testing
2. **Beta** (Invite-only): 500 creators from waitlist
3. **Public Launch**: Product Hunt + social media campaign

### 10.3 Go-to-Market
- **Target**: Short-form creators (YouTube Shorts, TikTok, Reels) working with 5-20 min source videos
- **Messaging**: "Edit your videos instantly — no upload, no wait. Captions, transcript editing, and reframing that runs in your browser."
- **Channels**: Product Hunt, Twitter/X, YouTube tutorials, Reddit (r/NewTubers, r/TikTokCreators)
- **Competitive angle**: "Unlike tools that make you upload and wait, ClipFlow processes your video instantly on your device. Your video never leaves your browser."
- **Upgrade nudge**: "Your clip looks amazing. Remove the watermark and export in 1080p for $9/mo."
- **Positioning vs OpusClip**: "OpusClip is great for 2-hour podcasts. ClipFlow is great for 15-minute videos you need edited NOW."

---

## 11. Appendix

### 11.1 Glossary
- **WebCodecs**: Browser API for low-level video encoding/decoding
- **FFmpeg.wasm**: FFmpeg compiled to WebAssembly for browser use
- **Whisper Web**: OpenAI's Whisper model running in browser via WebGPU
- **MediaPipe**: Google's ML framework for face/object detection
- **Web Audio API**: Browser API for audio analysis and manipulation
- **RMS Energy**: Root Mean Square energy — a measure of audio loudness per time window
- **TF-IDF**: Term Frequency–Inverse Document Frequency — text scoring algorithm used in smart clip suggestions
- **Cut List**: Ordered set of time ranges to exclude from the final export

### 11.2 References
- [OpusClip](https://www.opus.pro/) - Primary competitor
- [WebCodecs API](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API)
- [FFmpeg.wasm](https://ffmpegwasm.netlify.app/)
- [Whisper Web](https://whisperweb.dev/)
- [MediaPipe Face Detection](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js)

---

*Document maintained by: Product Team*  
*Last updated: February 24, 2026*
