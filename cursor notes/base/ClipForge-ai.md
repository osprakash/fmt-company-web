# Product Requirements Document (PRD) + Technical Specification
## ClipForge - "Upload a short, get back a 10x better short"

**Version:** 3.0
**Date:** March 6, 2026
**Status:** MVP Scope

---

## 1. Executive Summary

### 1.1 Product Vision
ClipForge is a **short-form video transformation engine**. Upload a raw Reel, TikTok, or YouTube Short. AI processes the actual video — cuts dead space, removes filler words, tracks and crops faces to 9:16, adds dynamic zooms at key moments, cleans up audio, burns animated captions, trims the intro preamble, optimizes the loop point, speed-ramps slow sections, and color grades the footage. Download a genuinely different, professionally edited video in under 2 minutes.

This is not a caption tool. This is not an overlay tool. This is an **automated video editor** that does what a $150/video human editor does — in 90 seconds.

### 1.2 Problem Statement
Short-form creators publish 3-7 shorts per week. The difference between 500 views and 50K views often comes down to editing quality:
- **Pacing kills most shorts**: "ums", dead air, slow intros — viewers swipe in 1-2 seconds
- **Professional editing costs $50-200 per short**: most creators can't afford this at 5+ videos/week
- **CapCut is powerful but manual**: every zoom, every cut, every caption adjustment is hand-placed. 30-60 minutes per short.
- **Existing AI tools only add captions**: Submagic, Captions.ai — they paste text on your video. The video itself doesn't change.
- **No tool transforms the actual video**: cuts, crops, zooms, audio cleanup, pacing — all still require a human editor or manual work in an NLE

### 1.3 Solution
A server-side pipeline that performs 10 real video transformations:

| # | Transformation | What Changes |
|---|---------------|-------------|
| 1 | Auto-cut silence | Dead air removed, video shortened |
| 2 | Auto-cut filler words | "um", "uh", "like" cut out |
| 3 | Face-tracking crop | 16:9 → 9:16 with face always centered |
| 4 | Auto-zoom key moments | 1.3x punch-in at punchlines, reveals, numbers |
| 5 | AI noise removal | Background hum, echo, room noise removed |
| 6 | Audio normalization + EQ | Consistent volume, voice clarity boost |
| 7 | Animated captions | Word-level, emphasis-highlighted, styled |
| 8 | Intro trim | "Hey guys so um..." preamble auto-cut |
| 9 | Loop optimization | Trailing dead air trimmed, seamless loop point |
| 10 | Color grade | Flat phone footage → polished, graded look |

**Bonus enhancements (toggleable):**
- Hook text card (pre-roll text before video)
- Speed ramp slow sections
- CTA end card
- Progress bar overlay

Upload → 1-4 minutes processing → download a video that looks and sounds like a different video.

### 1.4 Target User
- TikTok/Reels creators publishing 3-7 shorts per week
- Solo YouTubers posting Shorts alongside long-form
- Coaches, consultants, and personal brands using short-form for lead gen
- Small social media agencies managing 5-20 creator accounts

### 1.5 Business Goal
**$5K MRR within 6 months**, bootstrapped, no VC.

---

## 2. MVP Scope (4 Weeks)

### 2.1 Week 1: Upload + Transcription + Audio Pipeline

| Feature | Description | Priority |
|---------|-------------|----------|
| Video upload | Direct upload (S3 presigned URL), max 10 min / 500MB | P0 |
| Audio extraction | FFmpeg extract WAV from video | P0 |
| Transcription | Whisper large-v3 with word-level timestamps | P0 |
| AI noise removal | DeepFilterNet or Demucs for background noise cleanup | P0 |
| Audio normalization | FFmpeg loudnorm + voice EQ (presence boost, low-cut) | P0 |
| Silence detection | Identify gaps > 0.5s from Whisper timestamps | P0 |
| Filler word detection | Identify "um", "uh", "like", "you know", "so", "basically" from transcript | P0 |

### 2.2 Week 2: Video Transformations

| Feature | Description | Priority |
|---------|-------------|----------|
| Auto-cut silence + fillers | Generate FFmpeg concat filter to remove detected segments | P0 |
| Face-tracking crop | MediaPipe face detection → smooth crop path → 9:16 output | P0 |
| Auto-zoom key moments | LLM marks high-energy timestamps → FFmpeg zoompan at those points | P0 |
| Intro trim | LLM identifies where real content starts → trim preamble | P0 |
| Loop optimization | Detect natural endpoint → trim trailing dead air | P0 |
| Color grade | 3 LUT presets: Warm, Punchy, Clean | P1 |
| Speed ramp | Detect low-energy sections → 1.5x speed with pitch correction | P1 |

### 2.3 Week 3: Captions + Preview + Controls

| Feature | Description | Priority |
|---------|-------------|----------|
| Animated captions | Whisper timestamps + LLM emphasis → ASS subtitle burn | P0 |
| Caption styles | 3 presets: Bold Pop, Karaoke, Minimal | P0 |
| Hook text card | LLM-generated pre-roll text card (1.5s) prepended to video | P1 |
| CTA end card | "Follow for more" card appended | P1 |
| Progress bar | Thin animated bar at top of video | P2 |
| Preview page | Side-by-side before/after video player | P0 |
| Toggle controls | Enable/disable each transformation independently | P0 |
| Re-render | Change toggles → trigger new render with updated config | P0 |

### 2.4 Week 4: Auth + Payments + Launch

| Feature | Description | Priority |
|---------|-------------|----------|
| Auth | Email + Google OAuth (Clerk) | P0 |
| Stripe payments | $15/mo Creator, $39/mo Pro | P0 |
| Free tier | 1 video/month, watermarked, core transforms only, max 3 min | P0 |
| Processing status | Real-time progress (uploading → processing → ready) | P0 |
| Landing page | Hero with before/after video, pricing, FAQ | P0 |
| Dashboard | Video list with status + before/after scores | P0 |
| Download | Individual MP4 download with presigned URL | P0 |

### 2.5 Explicitly NOT in MVP

| Feature | Why Deferred |
|---------|--------------|
| Hook-first reorder (move best moment to start) | Context-aware editing is error-prone; defer until v2 |
| Auto background music | Licensing complexity |
| Multi-speaker reframe | Requires speaker diarization + dual face tracking |
| Custom font upload | Ship with built-in styles first |
| TikTok/IG direct publishing | API complexity |
| Batch upload | Solo creator focus for v1 |
| Mobile app | Web works on mobile browsers |
| Multi-language | English only for MVP |
| Client-side live preview of all transforms | Too complex; server renders, user sees before/after |

---

## 3. User Flow

```
1. UPLOAD              2. PROCESS              3. REVIEW + TOGGLE        4. DOWNLOAD
+-----------------+   +--------------------+   +---------------------+   +----------------+
| Upload MP4      |   | "Processing your   |   | BEFORE  |  AFTER   |   |                |
| or drag & drop  |-->| video..."          |-->|  [raw]  | [enhanced|-->| Download MP4   |
| (up to 10 min)  |   |                    |   |         |  video]  |   | (1080x1920)    |
|                 |   | ██████░░ 65%       |   |         |          |   |                |
| No account      |   |                    |   | Toggles:            |   | 60s → 44s      |
| needed for      |   | - Transcribing     |   | [x] Cut silence     |   | Noise: cleaned |
| first video     |   | - Cleaning audio   |   | [x] Cut fillers     |   | Captions: yes  |
|                 |   | - Detecting faces  |   | [x] Face crop       |   | Face-tracked   |
|                 |   | - Applying edits   |   | [x] Auto-zoom       |   |                |
|                 |   | - Rendering        |   | [x] Captions: Bold  |   | [Re-render]    |
|                 |   |                    |   | [ ] Speed ramp      |   | if toggles     |
| ~1-4 min total  |   | ~1-4 min           |   | [ ] Hook card       |   | changed        |
+-----------------+   +--------------------+   +---------------------+   +----------------+
```

### 3.1 First-Time Experience
1. Land on homepage. See side-by-side before/after video (same video, dramatic difference)
2. Upload a short video (no signup required for first video)
3. Wait ~1-4 minutes (depends on video length). Progress bar shows real phases: transcribing → cleaning audio → tracking faces → cutting → rendering
4. Review page loads:
   - Left: original video player
   - Right: enhanced video player
   - Below: toggle switches for each transformation
   - Stats: "60s → 44s | 5 filler words removed | 3.2s dead air cut | noise cleaned"
5. Toggle transformations on/off. Click "Re-render" if changed (~30s re-render)
6. Download enhanced MP4. First video is free (watermarked)
7. Prompt: "Remove watermark + unlimited videos — $15/mo"

### 3.2 Why Server-Side Render (Not Client-Side Preview)

The v1 PRD proposed a canvas-based live preview. That's wrong for this product because:
- **Auto-cuts change the video timeline** — you can't preview a shorter video on the original timeline
- **Face-tracking crop changes every frame** — can't simulate in real-time in browser
- **Noise removal modifies the audio waveform** — can't preview without processing
- **Speed ramps change duration** — timeline is different

Instead: render the full enhanced video server-side (~1-4 minutes depending on length), show before/after players. If user toggles something, re-render (~30-90 seconds since audio/transcript are cached). This is simpler to build, always accurate, and the render time is acceptable for short-form videos.

---

## 4. Core Features (The 10 Transformations)

### 4.1 Auto-Cut Silence

**What:** Detect silence gaps > 0.5 seconds in the audio. Remove them with tight jump cuts.

**How:**
- Whisper provides word-level timestamps with start/end times
- Gaps between words > 0.5s are marked as silence segments
- FFmpeg concat filter stitches remaining segments
- 50ms audio crossfade at each cut point to avoid clicks
- Optional: 1-frame video crossfade to soften jump cuts

**Config:**
```json
{
  "enabled": true,
  "min_silence_duration": 0.5,
  "crossfade_ms": 50,
  "segments_removed": 4,
  "time_saved_sec": 3.2
}
```

**Impact:** Typically saves 3-8 seconds on a 60-second video. Instantly tighter pacing.

---

### 4.2 Auto-Cut Filler Words

**What:** Detect filler words ("um", "uh", "like", "you know", "so", "basically", "literally", "right") from the Whisper transcript. Cut those segments.

**How:**
- Whisper transcript provides per-word timestamps
- Match against filler word list (configurable)
- Context-aware: "like" as a filler ("I was like, um") vs. "like" as content ("I like this") — LLM disambiguates
- Same FFmpeg concat approach as silence removal
- Filler cuts and silence cuts are merged into a single cut list, sorted by timestamp

**Config:**
```json
{
  "enabled": true,
  "filler_words_found": [
    { "word": "um", "count": 3, "timestamps": [4.2, 18.7, 31.5] },
    { "word": "like", "count": 4, "timestamps": [7.1, 12.3, 25.8, 40.2], "is_filler": [true, false, true, true] },
    { "word": "you know", "count": 2, "timestamps": [15.4, 35.1] }
  ],
  "total_removed": 7,
  "time_saved_sec": 2.8
}
```

**Impact:** Removes 5-12 filler words from a typical 60-second video. Combined with silence cuts, a 60-second video becomes 48-52 seconds with zero wasted words.

---

### 4.3 Face-Tracking Auto-Crop

**What:** Convert any aspect ratio to 9:16 vertical with the speaker's face always centered and properly framed.

**How:**
- MediaPipe Face Detection runs per frame (or every 3rd frame + interpolation for speed)
- Calculate bounding box center for each frame
- Smooth the crop path with exponential moving average (prevents jittery movement)
- Crop window: `source_height × 9/16` wide, full height, centered on smoothed face position
- Scale to 1080×1920 output
- If no face detected (B-roll, product shot): fall back to center crop
- If multiple faces: crop to include both (wider frame) or track the speaker (based on audio energy)

**Config:**
```json
{
  "enabled": true,
  "source_aspect": "16:9",
  "output_aspect": "9:16",
  "face_detected_pct": 94.5,
  "smoothing_factor": 0.15,
  "fallback_mode": "center_crop"
}
```

**FFmpeg approach:**
- Pre-process: run MediaPipe, output per-frame crop coordinates to a text file
- Render: FFmpeg `crop` filter with frame-by-frame coordinates via `sendcmd` or pre-computed filter script

**Impact:** Transforms unusable horizontal footage into platform-native vertical video. This alone is a $20/mo feature.

---

### 4.4 Auto-Zoom on Key Moments

**What:** Add 1.2-1.4× punch-in zooms at high-energy moments — punchlines, reveals, numbers, emotional peaks. Creates visual rhythm in static talking-head footage.

**How:**
- LLM reads the full transcript with timestamps
- Identifies 3-5 "zoom-worthy" moments per 60 seconds:
  - Surprising statements or reveals
  - Numbers or statistics
  - Emotional peaks (anger, excitement, vulnerability)
  - Punchlines or payoffs
  - Contrast moments ("but here's the thing...")
- Each zoom: ease-in over 0.3s → hold 1-2s → ease-out over 0.3s
- Zoom center: face position (from face-tracking data)
- Zooms are applied AFTER face-tracking crop, so they're always face-centered

**Config:**
```json
{
  "enabled": true,
  "zoom_points": [
    {
      "timestamp_sec": 8.3,
      "duration_sec": 1.5,
      "zoom_factor": 1.3,
      "reason": "Surprising number reveal: 'I lost $50,000'",
      "transcript_excerpt": "...and that's when I realized I lost fifty thousand dollars"
    },
    {
      "timestamp_sec": 24.1,
      "duration_sec": 1.2,
      "zoom_factor": 1.25,
      "reason": "Emotional peak: vulnerability moment",
      "transcript_excerpt": "...I had never felt so alone in my life"
    },
    {
      "timestamp_sec": 41.7,
      "duration_sec": 1.8,
      "zoom_factor": 1.35,
      "reason": "Punchline / payoff",
      "transcript_excerpt": "...and that one decision changed everything"
    }
  ]
}
```

**FFmpeg:** `zoompan` filter with calculated keyframes, or `scale` + `crop` with animated parameters.

**Impact:** The single biggest visual upgrade for talking-head shorts. Every viral short uses strategic zooms — this automates what editors charge $30-50 to do manually.

---

### 4.5 AI Noise Removal

**What:** Remove background noise — room hum, AC, traffic, echo, keyboard clicks, fan noise — using a pre-trained AI model.

**How:**
- DeepFilterNet (open-source, real-time capable, MIT license) or Meta's Demucs
- Process the extracted audio WAV through the model
- Output: clean voice-only WAV
- This runs BEFORE normalization and EQ (clean signal first, then enhance)

**Config:**
```json
{
  "enabled": true,
  "model": "deepfilternet3",
  "noise_reduction_level": "auto",
  "input_snr_estimate_db": 18,
  "output_snr_estimate_db": 42
}
```

**Impact:** Bedroom recording → studio quality. This is the "invisible upgrade" that makes everything else feel more professional. Viewers can't articulate why the video feels better, but they don't swipe away.

---

### 4.6 Audio Normalization + Voice EQ

**What:** Consistent volume across the entire clip. No quiet mumbles, no sudden loud moments. Plus a voice-optimized EQ that adds clarity and presence.

**How:**
- FFmpeg `loudnorm` filter: two-pass loudness normalization to -16 LUFS (social media standard)
- Voice EQ via FFmpeg `equalizer` filter:
  - High-pass at 80Hz (remove rumble)
  - Slight boost at 2-4kHz (presence / clarity)
  - Slight boost at 8-10kHz (air / crispness)
  - Gentle compression (dynamic range control)
- Applied to the noise-removed audio

**FFmpeg:**
```bash
ffmpeg -i clean_audio.wav \
  -af "highpass=f=80,
       equalizer=f=3000:t=q:w=1.5:g=3,
       equalizer=f=9000:t=q:w=2:g=2,
       acompressor=threshold=-20dB:ratio=3:attack=5:release=50,
       loudnorm=I=-16:TP=-1.5:LRA=11" \
  normalized_audio.wav
```

**Config:**
```json
{
  "enabled": true,
  "target_lufs": -16,
  "eq_preset": "voice_clarity",
  "compression": true
}
```

**Impact:** Combined with noise removal, this is the full "audio glow-up." The creator sounds like they're recording in a treated studio with a $500 mic, not in their bedroom with AirPods.

---

### 4.7 Animated Word-Level Captions

**What:** Styled, animated captions with per-word timing and emphasis highlighting on key words.

**How:**
- Whisper provides word-level timestamps
- LLM marks emphasis words: numbers, emotional triggers, surprising words, key nouns, action verbs
- Generate ASS (Advanced SubStation Alpha) subtitle file with per-word events
- Emphasis words get: larger font size, different color, slight scale animation
- Three style presets (see below)
- Captions are timed to the POST-CUT timeline (after silence/filler removal)

**Timestamp remapping:** After auto-cuts, the timeline changes. Caption timestamps must be remapped:
```
Original: word at 8.5s → but 2.1s of cuts happened before 8.5s → new timestamp: 6.4s
```
The cut list is used to remap all word timestamps to the new shortened timeline.

**Caption Styles:**

| Style | Font | Behavior | Look |
|-------|------|----------|------|
| Bold Pop | Montserrat Bold, 62px | Words appear 2-3 at a time, emphasis words scale up 1.2x + change to accent color | White text, gold emphasis, black outline, dark shadow |
| Karaoke | Arial Bold, 56px | Full phrase visible, current word highlights as spoken | White text, current word turns cyan, dark background bar |
| Minimal | Inter, 44px | Small bottom-positioned, simple fade in/out per phrase | White text, subtle shadow, no background |

**Config:**
```json
{
  "enabled": true,
  "style": "bold_pop",
  "words": [
    { "word": "I", "start": 0.0, "end": 0.18, "emphasis": false },
    { "word": "lost", "start": 0.18, "end": 0.42, "emphasis": true },
    { "word": "fifty", "start": 0.42, "end": 0.68, "emphasis": true },
    { "word": "thousand", "start": 0.68, "end": 1.05, "emphasis": true },
    { "word": "dollars", "start": 1.05, "end": 1.45, "emphasis": true }
  ]
}
```

**Impact:** 93% of social video is watched on mute. No captions = invisible. But the real value is emphasis styling — it guides the viewer's attention and adds visual energy.

---

### 4.8 Intro Trim

**What:** Auto-detect and cut the "hey guys, so, um, today I wanted to..." preamble. Start the video at the actual content or hook.

**How:**
- LLM receives the full transcript with timestamps
- Identifies the "real start" — where the hook or substantive content begins
- Common patterns to cut:
  - Greetings: "Hey", "What's up", "Hi everyone"
  - Filler preambles: "So today I want to talk about..."
  - Throat clearing, sighs, "okay so..."
- The cut point becomes the new video start (combined with the silence/filler cut list)
- If the original hook IS strong (no preamble), LLM returns "no trim needed"

**Config:**
```json
{
  "enabled": true,
  "original_start_text": "Hey guys, so um, today I wanted to share something really interesting that happened to me",
  "trimmed_start_text": "Something really interesting happened to me last Tuesday",
  "trim_duration_sec": 3.8,
  "reason": "Removed greeting + filler preamble. Content starts at 'Something really interesting'"
}
```

**Impact:** The TikTok algorithm measures 1-second and 3-second retention. Cutting 2-4 seconds of preamble means the first thing viewers hear is the hook. This directly impacts whether the algorithm pushes or buries the video.

---

### 4.9 Loop Optimization

**What:** Trim the trailing dead space at the end so the video loops seamlessly back to the start. Optimize for replay behavior.

**How:**
- Detect the last complete sentence endpoint from the transcript
- Trim everything after: trailing silence, "so yeah...", camera-reaching-to-stop moments
- Optional: 200ms audio fade-out to avoid abrupt ending
- LLM evaluates if the ending naturally connects to the opening (for a satisfying loop)
- If it does: flag it and ensure the trim creates a clean loop point

**Config:**
```json
{
  "enabled": true,
  "original_end_sec": 58.3,
  "trimmed_end_sec": 55.8,
  "trim_saved_sec": 2.5,
  "last_sentence": "...and that one decision changed everything",
  "loop_quality": "good",
  "loop_note": "Ending 'changed everything' connects to opening 'Something interesting happened' — natural curiosity loop"
}
```

**Impact:** A 45-second video watched 2.3 times (because it loops well) = 103 seconds of watch time. A 48-second video with an awkward ending watched 1.1 times = 53 seconds. The algorithm sees 2× the signal. Pure algorithmic leverage.

---

### 4.10 Color Grade

**What:** Apply a subtle color grade to phone footage — contrast, saturation, skin tone warmth. Makes flat footage look "produced."

**How:**
- FFmpeg `lut3d` filter with pre-built LUT files
- 3 presets, auto-suggested based on scene analysis (brightness, color temperature):
  - **Warm**: Slight orange shift in midtones, warm shadows, boosted skin tones. Best for talking-head, personal content.
  - **Punchy**: High contrast, boosted saturation, slightly crushed blacks. Best for energetic, bold content.
  - **Clean**: Neutral, slightly bright, soft contrast. Best for educational, minimal aesthetic.
- Auto-exposure correction: if footage is under/over-exposed, FFmpeg `curves` filter corrects before LUT

**Config:**
```json
{
  "enabled": true,
  "preset": "warm",
  "auto_exposure_correction": true,
  "brightness_adjustment": 0.05
}
```

**FFmpeg:**
```bash
ffmpeg -i input.mp4 \
  -vf "curves=preset=lighter:master='0/0 0.1/0.15 0.5/0.55 1/1',
       lut3d=file=warm.cube" \
  output.mp4
```

**Impact:** Subtle but cumulative. Combined with all other transforms, this is the final polish that makes the output feel like a different tier of content. Viewers subconsciously associate color-graded footage with "creator worth following."

---

## 5. Bonus Enhancements (Toggleable, Lower Priority)

### 5.1 Hook Text Card (Pre-Roll)

A 1-1.5 second text card prepended before the video. Bold statement on a dark background (or blurred first frame). Generated by LLM from the transcript.

```json
{
  "enabled": false,
  "text": "I lost $50K learning this lesson",
  "duration_sec": 1.5,
  "style": "bold_white_on_dark",
  "alternatives": [
    "The mistake that cost me everything",
    "Nobody tells you this about entrepreneurship"
  ]
}
```

### 5.2 Speed Ramp Slow Sections

Detect low-energy sections (exposition, setup) and speed them to 1.5× with pitch-corrected audio.

```json
{
  "enabled": false,
  "ramp_sections": [
    { "start": 18.2, "end": 24.5, "speed": 1.5, "reason": "Expository setup, low vocal energy" }
  ],
  "time_saved_sec": 2.1
}
```

### 5.3 CTA End Card

Append a 2-second CTA card after the video. LLM generates contextual CTA text.

```json
{
  "enabled": false,
  "text": "Follow for Part 2",
  "style": "soft",
  "duration_sec": 2.0,
  "alternatives": ["Save this for later", "Comment 'GUIDE' for the full breakdown"]
}
```

### 5.4 Progress Bar

Thin animated bar at the top or bottom of the video showing playback progress. Retention hack — viewers stay to see the bar complete.

```json
{
  "enabled": false,
  "position": "top",
  "color": "#FFFFFF",
  "height_px": 4,
  "opacity": 0.6
}
```

---

## 6. Pricing

### 6.1 Model: Transform-Gated Freemium

| | **Free** | **Creator ($15/mo)** | **Pro ($39/mo)** |
|--|----------|---------------------|-----------------|
| Videos/month | 1 | 30 | Unlimited |
| Max input length | 3 min | 3 min | 10 min |
| Auto-cut silence + fillers | Yes | Yes | Yes |
| Face-tracking crop | Yes | Yes | Yes |
| Noise removal + audio EQ | Yes | Yes | Yes |
| Animated captions | 1 style (Minimal) | All 3 styles | All + custom colors |
| Auto-zoom key moments | No | Yes | Yes |
| Intro trim | No | Yes | Yes |
| Loop optimization | No | Yes | Yes |
| Color grade | No | Yes (3 presets) | Yes + manual adjust |
| Speed ramp | No | No | Yes |
| Hook text card | No | No | Yes |
| CTA end card | No | No | Yes |
| Progress bar | No | No | Yes |
| Watermark | Yes | No | No |
| Export quality | 720p | 1080p | 1080p |
| Priority rendering | No | No | Yes |
| Brand kit (colors) | No | No | Yes |

**Free tier strategy:** 1 video/month with the 3 most impactful transforms (auto-cut, face-crop, audio cleanup) so the output is genuinely impressive. One video is enough to create the "wow" moment that drives conversion — but not enough to actually run a content workflow. Paid tiers add volume + polish layers (zoom, color, captions style, loop, intro trim).

### 6.2 Unit Economics

| Cost component | Per video (60 sec) | Per video (3 min) | Per video (10 min) |
|----------------|--------------------|--------------------|---------------------|
| Whisper transcription (Modal GPU) | $0.02 | $0.06 | $0.20 |
| LLM analysis — Claude Haiku | $0.01 | $0.02 | $0.05 |
| DeepFilterNet noise removal (Modal GPU) | $0.03 | $0.10 | $0.30 |
| MediaPipe face tracking (Fargate Spot) | $0.003 | $0.012 | $0.04 |
| FFmpeg full render (Fargate Spot) | $0.006 | $0.018 | $0.05 |
| S3 storage (source + rendered, 30 day lifecycle) | $0.002 | $0.005 | $0.012 |
| CloudFront egress (before + after preview + download, ~3x file size) | $0.013 | $0.038 | $0.102 |
| **Total variable cost** | **$0.08** | **$0.25** | **$0.73** |

*Egress assumes: user streams "before" video (1x), streams "after" video (1x), downloads final (1x) = 3x file size. File sizes: 60s ~50MB, 3min ~150MB, 10min ~400MB. CloudFront first 10TB: $0.085/GB. Re-renders add another ~1x egress per re-render.*

| Scenario | Revenue | Cost | Gross Margin |
|----------|---------|------|-------------|
| Creator user, 10 videos/mo (avg 2 min each) | $15 | $1.60 | 89% |
| Creator user, 30 videos/mo (avg 2 min each) | $15 | $4.80 | 68% |
| Pro user, 30 videos/mo (avg 3 min each) | $39 | $6.30 | 84% |
| Pro user, 10 videos/mo (avg 8 min each) | $39 | $5.30 | 86% |
| Free user, 1 video/mo (max 3 min) | $0 | $0.21 | -$0.21 |

### 6.3 Revenue Targets

| Milestone | Paying Users | MRR |
|-----------|-------------|-----|
| Month 1 | 20 | $400 |
| Month 3 | 100 | $2,000 |
| Month 6 | 200 | $5,200 |

Assumes blended ARPU of ~$26 (mix of Creator + Pro). 6% free-to-paid conversion.

### 6.4 Monthly Infrastructure Costs (Fixed)

| Service | Cost |
|---------|------|
| ECS Fargate Spot (processing) | ~$10-25 (scales to zero, Spot saves ~70% vs on-demand) |
| RDS PostgreSQL (db.t4g.micro) | ~$15 |
| S3 + CloudFront | ~$5-20 |
| Modal (GPU — Whisper + DeepFilterNet) | Pay-per-use |
| Vercel (frontend) | $0 (hobby) or $20 (pro) |
| Clerk (auth) | $0 (free tier) |
| **Total fixed** | **~$30-80/mo** |

Breakeven at ~2-3 paying users.

---

## 7. Technical Architecture

### 7.1 System Overview

```
                         +--------------------+
                         |   Vercel (CDN)     |
                         |   Next.js App      |
                         |   React Frontend   |
                         +--------+-----------+
                                  |
                                  | API Routes / tRPC
                                  |
                    +-------------+-------------+
                    |     AWS Region (us-east-1) |
                    |                            |
     +--------------+    +----------+    +-------+-----------+
     |  API Gateway  |    |          |    |                   |
     |  + Lambda     +--->+  SQS     +--->+  ECS Fargate      |
     |  (REST/tRPC)  |    |  Queue   |    |  (Worker Tasks)   |
     +---------+------+    +----------+    +---+-----+---------+
               |                               |     |
        +------+------+              +--------+     +--------+
        |  RDS Postgres|              |  Modal GPU   |  S3    |
        |  (metadata)  |              |  - Whisper   | (files)|
        +--------------+              |  - DeepFilter|--------+
                                      +--------------+
```

### 7.2 Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | Next.js 15 + React + TypeScript | SSR, API routes, Vercel deploy |
| Styling | Tailwind CSS | Fast iteration |
| API | tRPC or Next.js API Routes | Type-safe, co-located |
| Auth | Clerk | Free tier, Google OAuth |
| Database | PostgreSQL on RDS (db.t4g.micro) | $15/mo, reliable |
| ORM | Drizzle ORM | Lightweight, type-safe |
| Job Queue | AWS SQS + ECS Fargate Spot Tasks | Scales to zero, ~70% cheaper than on-demand |
| File Storage | S3 + presigned URLs | Direct browser upload |
| Transcription | Whisper large-v3 on Modal (GPU) | Pay-per-second, word-level timestamps |
| Noise Removal | DeepFilterNet on Modal (GPU) | Open-source, real-time capable, MIT license |
| Face Tracking | MediaPipe Face Detection (CPU) | Runs in ECS worker, no GPU needed |
| LLM | Claude Haiku 4.5 | Cheap, fast, structured output |
| Video Processing | FFmpeg 7 on ECS Fargate (CPU) | All transforms: crop, zoom, cut, caption, grade, encode |
| Color Grading | FFmpeg LUT3D + curves | Pre-built .cube LUT files |
| Payments | Stripe Checkout + Webhooks | Subscription billing |
| Monitoring | CloudWatch + Sentry free tier | Errors + logs |
| CDN | CloudFront | Low latency downloads |

### 7.3 Key Architecture Decisions

- **All transforms in a single FFmpeg command:** After pre-processing (face coordinates, cut list, zoom keyframes, ASS captions, clean audio), the final render is ONE FFmpeg command with a complex filtergraph. This avoids re-encoding multiple times and keeps quality high.
- **GPU only for Whisper + DeepFilterNet:** Everything else (face tracking, rendering) runs on CPU. This keeps GPU costs minimal.
- **Modal for GPU tasks:** Pay-per-second. A 60-second short needs ~3 seconds of Whisper + ~5 seconds of DeepFilterNet = ~8 seconds of GPU time total. No idle GPU cost.
- **ECS Fargate Spot for CPU tasks:** Face tracking, LLM call, FFmpeg render all run in ECS tasks using Spot capacity (~70% cheaper). Tasks are short (1-4 min) and idempotent — if Spot reclaims, SQS redelivers the message and a new task picks it up. Scales to zero.
- **No client-side preview:** Server renders the full output. Before/after comparison in browser. Re-render on toggle changes. Simpler, always accurate, acceptable latency for short videos.

---

## 8. Data Model

```sql
-- Users (synced from Clerk via webhook)
CREATE TABLE users (
  id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clerk_id               TEXT UNIQUE NOT NULL,
  email                  TEXT NOT NULL,
  plan                   TEXT NOT NULL DEFAULT 'free',  -- 'free' | 'creator' | 'pro'
  stripe_customer_id     TEXT,
  stripe_subscription_id TEXT,
  videos_this_month      INT NOT NULL DEFAULT 0,
  billing_cycle_start    TIMESTAMPTZ,
  created_at             TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Videos (source upload + processing state)
CREATE TABLE videos (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id),
  title           TEXT,
  s3_key_source   TEXT NOT NULL,
  duration_sec    FLOAT,
  width           INT,
  height          INT,
  fps             FLOAT,
  status          TEXT NOT NULL DEFAULT 'uploaded',
    -- 'uploaded' | 'processing' | 'ready' | 'failed'
  error_message   TEXT,

  -- Transcription output (cached, reused across renders)
  transcript      JSONB,           -- Whisper output: { segments, words }

  -- Analysis output (cached, reused across renders)
  analysis        JSONB,           -- LLM output: zoom points, intro trim, filler disambiguation, emphasis words, hook card, CTA

  -- Face tracking data (cached)
  face_data       JSONB,           -- Per-frame face coordinates: [{ frame, x, y, w, h }]

  -- Audio processing (cached S3 keys)
  s3_key_clean_audio TEXT,         -- Noise-removed + normalized audio WAV

  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Render configs (what the user toggled on/off)
CREATE TABLE renders (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
  status          TEXT NOT NULL DEFAULT 'queued',
    -- 'queued' | 'rendering' | 'done' | 'failed'

  -- Toggle states
  cut_silence     BOOLEAN NOT NULL DEFAULT true,
  cut_fillers     BOOLEAN NOT NULL DEFAULT true,
  face_crop       BOOLEAN NOT NULL DEFAULT true,
  auto_zoom       BOOLEAN NOT NULL DEFAULT true,
  noise_removal   BOOLEAN NOT NULL DEFAULT true,
  audio_eq        BOOLEAN NOT NULL DEFAULT true,
  captions        BOOLEAN NOT NULL DEFAULT true,
  caption_style   TEXT NOT NULL DEFAULT 'bold_pop',  -- 'bold_pop' | 'karaoke' | 'minimal'
  intro_trim      BOOLEAN NOT NULL DEFAULT true,
  loop_optimize   BOOLEAN NOT NULL DEFAULT true,
  color_grade     BOOLEAN NOT NULL DEFAULT false,
  color_preset    TEXT DEFAULT 'warm',               -- 'warm' | 'punchy' | 'clean'
  speed_ramp      BOOLEAN NOT NULL DEFAULT false,
  hook_card       BOOLEAN NOT NULL DEFAULT false,
  hook_card_text  TEXT,
  cta_card        BOOLEAN NOT NULL DEFAULT false,
  cta_card_text   TEXT,
  progress_bar    BOOLEAN NOT NULL DEFAULT false,

  -- Output
  s3_key_output   TEXT,
  output_duration_sec FLOAT,
  render_duration_ms INT,

  -- Stats
  stats           JSONB,           -- { time_saved, fillers_removed, silence_cut, noise_reduced, ... }

  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes
CREATE INDEX idx_videos_user ON videos(user_id, created_at DESC);
CREATE INDEX idx_renders_video ON renders(video_id, created_at DESC);
CREATE INDEX idx_videos_status ON videos(status) WHERE status NOT IN ('ready', 'failed');
CREATE INDEX idx_renders_status ON renders(status) WHERE status NOT IN ('done', 'failed');
```

**Key design choice:** Videos store cached intermediate data (transcript, face data, clean audio, LLM analysis). Renders store toggle states + output. This means re-renders with different toggles are fast — no need to re-transcribe, re-track faces, or re-clean audio.

---

## 9. API Design

### 9.1 Endpoints

```
POST   /api/videos/upload-url       -- Get S3 presigned upload URL
POST   /api/videos                  -- Create video record + trigger processing
GET    /api/videos                  -- List user's videos
GET    /api/videos/:id              -- Get video + latest render + status
GET    /api/videos/:id/analysis     -- Get full analysis (zoom points, filler words, etc.)

POST   /api/videos/:id/render      -- Trigger render with specific toggle config
GET    /api/renders/:id             -- Get render status + download URL
GET    /api/renders/:id/download    -- Get presigned download URL

POST   /api/stripe/checkout         -- Create Stripe Checkout session
POST   /api/stripe/webhook          -- Stripe webhook handler
POST   /api/stripe/portal           -- Customer billing portal

GET    /api/user                    -- Current user profile + usage
```

### 9.2 Key Request/Response Examples

**Create video + start processing:**
```json
POST /api/videos
{ "s3_key": "uploads/usr_abc/vid_123/source.mp4", "title": "productivity-hack.mp4" }

Response 201:
{ "id": "vid_123", "status": "processing" }
```

**Get video after processing completes:**
```json
GET /api/videos/vid_123

Response 200:
{
  "id": "vid_123",
  "status": "ready",
  "duration_sec": 58.3,
  "title": "productivity-hack.mp4",
  "analysis_summary": {
    "silence_segments": 4,
    "silence_total_sec": 3.2,
    "filler_words": 7,
    "filler_total_sec": 2.8,
    "intro_trim_sec": 3.8,
    "loop_trim_sec": 2.5,
    "zoom_points": 3,
    "face_detected_pct": 94.5,
    "estimated_output_duration_sec": 44.2
  },
  "latest_render": {
    "id": "rnd_456",
    "status": "done",
    "output_duration_sec": 44.2,
    "download_url": "https://cdn.clipforge.com/...",
    "stats": {
      "time_saved_sec": 14.1,
      "fillers_removed": 7,
      "silence_segments_cut": 4,
      "zoom_points_applied": 3
    },
    "toggles": {
      "cut_silence": true,
      "cut_fillers": true,
      "face_crop": true,
      "auto_zoom": true,
      "noise_removal": true,
      "audio_eq": true,
      "captions": true,
      "caption_style": "bold_pop",
      "intro_trim": true,
      "loop_optimize": true,
      "color_grade": true,
      "color_preset": "warm",
      "speed_ramp": false,
      "hook_card": false,
      "cta_card": false,
      "progress_bar": false
    }
  }
}
```

**Trigger re-render with different toggles:**
```json
POST /api/videos/vid_123/render
{
  "cut_silence": true,
  "cut_fillers": true,
  "face_crop": true,
  "auto_zoom": false,
  "noise_removal": true,
  "audio_eq": true,
  "captions": true,
  "caption_style": "karaoke",
  "intro_trim": true,
  "loop_optimize": true,
  "color_grade": false,
  "speed_ramp": false,
  "hook_card": false,
  "cta_card": false,
  "progress_bar": false
}

Response 201:
{ "render_id": "rnd_789", "status": "queued", "estimated_seconds": 30 }
```

---

## 10. Processing Pipeline (Detail)

### 10.1 Pipeline Overview

The pipeline has two phases: **Process** (once per video, ~60 sec) and **Render** (per toggle config, ~30 sec).

```
PHASE 1: PROCESS (once per video upload)
=========================================

Video uploaded to S3
    |
    v
[Step 1: INGEST + VALIDATE]  ~2 sec
    |
    +-- FFmpeg probe: duration, dimensions, fps, codec
    +-- Reject if > 10min or > 500MB or non-video
    +-- FFmpeg: extract audio as WAV (16kHz mono)
    +-- Upload audio.wav to S3
    |
    v
[Step 2: TRANSCRIBE]  ~3 sec (Modal GPU)
    |
    +-- Whisper large-v3 with word_timestamps=True
    +-- Output: { segments, words: [{ word, start, end }] }
    +-- Save to DB: videos.transcript
    |
    v
[Step 3: NOISE REMOVAL]  ~5 sec (Modal GPU)
    |
    +-- DeepFilterNet process audio.wav → clean_audio.wav
    +-- Upload clean_audio.wav to S3
    +-- Save S3 key to DB
    |
    v
[Step 4: AUDIO NORMALIZATION + EQ]  ~2 sec (ECS CPU)
    |
    +-- FFmpeg loudnorm + EQ + compression on clean_audio.wav
    +-- Output: final_audio.wav
    +-- Upload to S3, update DB
    |
    v
[Step 5: FACE TRACKING]  ~15 sec (ECS CPU)
    |
    +-- Download source video
    +-- MediaPipe face detection: every 3rd frame
    +-- Interpolate intermediate frames
    +-- Smooth crop path (exponential moving average)
    +-- Output: [{ frame, crop_x, crop_y, crop_w, crop_h }]
    +-- Save to DB: videos.face_data
    |
    v
[Step 6: LLM ANALYSIS]  ~3 sec (API call)
    |
    +-- Send transcript + video metadata to Claude Haiku
    +-- Prompt returns (see 10.2):
    |     - Filler word disambiguation (is "like" filler or content?)
    |     - Zoom-worthy moments (timestamps + reasons)
    |     - Intro trim point
    |     - Loop endpoint
    |     - Emphasis words for captions
    |     - Hook card text suggestions
    |     - CTA suggestions
    |     - Speed ramp sections
    |     - Color grade suggestion
    +-- Save to DB: videos.analysis
    |
    v
[PROCESSING COMPLETE]
    +-- Update DB: status = 'ready'
    +-- Trigger DEFAULT RENDER (all plan-allowed features enabled)
    +-- Notify user


PHASE 2: RENDER (per toggle config, ~30 sec)
=============================================

Render requested (default or user-triggered)
    |
    v
[Step 7: BUILD CUT LIST]  ~1 sec
    |
    +-- From transcript + analysis, build ordered cut list:
    |     - Silence segments to remove
    |     - Filler word segments to remove (only LLM-confirmed fillers)
    |     - Intro preamble to remove
    |     - Trailing dead air to remove
    +-- Only include cuts for ENABLED toggles
    +-- Calculate timestamp remapping table (old → new timeline)
    |
    v
[Step 8: BUILD FFMPEG FILTERGRAPH]  ~1 sec
    |
    +-- From cut list + face data + analysis + toggles, generate:
    |
    |   VIDEO FILTERS (in order):
    |   1. trim + concat (apply cuts — silence, fillers, intro, outro)
    |   2. crop (face-tracking coordinates, per-frame)
    |   3. scale to 1080x1920
    |   4. zoompan (auto-zoom keyframes at LLM-identified moments)
    |   5. lut3d (color grade LUT, if enabled)
    |   6. drawtext (progress bar, if enabled)
    |   7. ass (burn captions, with remapped timestamps)
    |   8. drawtext (hook card, if enabled — prepended)
    |   9. drawtext (CTA card, if enabled — appended)
    |
    |   AUDIO FILTERS:
    |   1. Use pre-processed final_audio.wav (already clean + normalized)
    |   2. trim + concat (same cut list as video)
    |   3. atempo (speed ramp sections, if enabled)
    |
    +-- Generate ASS subtitle file (remapped timestamps, emphasis styling)
    |
    v
[Step 9: FFMPEG RENDER]  ~20-30 sec
    |
    +-- Single FFmpeg command with full filtergraph
    +-- Input: source video + final_audio.wav + captions.ass + LUT file
    +-- Output: H.264 + AAC, 1080x1920, CRF 20
    +-- Add watermark if free tier
    +-- movflags +faststart for streaming
    +-- Upload to S3
    |
    v
[RENDER COMPLETE]
    +-- Update renders table: status = 'done', s3_key, stats
    +-- Notify user
```

### 10.2 LLM Analysis Prompt

```
You are a short-form video optimization engine. Analyze this transcript and
generate a complete enhancement config.

VIDEO INFO:
- Duration: {duration_sec} seconds
- Dimensions: {width}x{height}
- Platform: TikTok / Instagram Reels / YouTube Shorts

TRANSCRIPT (word-level timestamps):
---
{formatted_transcript}
---

Return a JSON object with ALL of the following:

1. FILLER_DISAMBIGUATION:
   For each instance of "like", "so", "right", "basically", "literally"
   in the transcript, determine if it's a filler word or content.
   Return: [{ word, timestamp, is_filler: boolean, reason }]

2. ZOOM_POINTS:
   Identify 3-5 moments that deserve a visual zoom punch-in.
   Criteria: surprising reveal, number, emotional peak, punchline, contrast.
   Return: [{ timestamp_sec, duration_sec, zoom_factor (1.2-1.4),
              reason, transcript_excerpt }]

3. INTRO_TRIM:
   Identify where the actual content/hook begins (skip greetings, fillers,
   preamble). If the opening is already strong, return null.
   Return: { trim_to_sec, original_text, trimmed_text, reason } or null

4. LOOP_ENDPOINT:
   Identify the natural endpoint — last complete thought/sentence.
   Evaluate if ending connects to opening for a loop.
   Return: { end_at_sec, last_sentence, loop_quality: "good"|"fair"|"poor",
             loop_note }

5. CAPTION_EMPHASIS:
   For every word, mark emphasis: true/false.
   Emphasize: numbers, emotional words, surprising words, key nouns, action verbs.
   Do NOT emphasize: fillers, articles, prepositions, conjunctions.
   Return: [{ word, start, end, emphasis: boolean }]

6. HOOK_CARD:
   Generate 3 text options for a pre-roll hook card (max 10 words each).
   Styles: personal_story, contrarian, curiosity_gap.
   Return: [{ text, style }]

7. CTA:
   Generate 3 contextual CTA options.
   Return: [{ text, style: "soft"|"utility"|"engagement" }]

8. SPEED_RAMP:
   Identify 0-2 sections that are low-energy (exposition, repetition)
   where 1.5x speed would improve pacing without losing content.
   Return: [{ start_sec, end_sec, reason }] or []

9. COLOR_SUGGESTION:
   Based on the content tone, suggest: "warm", "punchy", or "clean".
   Return: { preset, reason }

Return valid JSON only. No markdown. No explanation.
```

### 10.3 FFmpeg Master Render Command (Simplified Example)

```bash
# Full pipeline in one command (simplified — actual filtergraph is generated dynamically)

ffmpeg \
  -i source.mp4 \
  -i final_audio.wav \
  -filter_complex "
    [0:v] trim=3.8:55.8, setpts=PTS-STARTPTS,
    select='not(between(t,4.4,5.1)+between(t,8.2,9.5)+between(t,18.7,19.2))',
    setpts=N/FRAME_RATE/TB,
    crop=w_expr:h_expr:x_expr:y_expr,
    scale=1080:1920,
    zoompan=z='if(between(in_time,8.3,9.8),1.3,1)':d=1:x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)',
    lut3d=warm.cube,
    ass=captions.ass
    [vout];
    [1:a] atrim=3.8:55.8, asetpts=PTS-STARTPTS,
    aselect='not(between(t,4.4,5.1)+between(t,8.2,9.5)+between(t,18.7,19.2))',
    asetpts=N/SR/TB
    [aout]
  " \
  -map "[vout]" -map "[aout]" \
  -c:v libx264 -preset medium -crf 20 \
  -c:a aac -b:a 192k \
  -movflags +faststart \
  output.mp4
```

**Note:** The actual filtergraph is dynamically generated in Python based on the toggle config, cut list, face coordinates, zoom keyframes, and caption file. The worker builds the filter string programmatically.

### 10.4 Timestamp Remapping for Captions

After cuts (silence, fillers, intro, outro), the timeline changes. Captions must be remapped.

```python
def remap_timestamp(original_ts: float, cut_list: list[tuple[float, float]]) -> float:
    """Remap an original timestamp to the post-cut timeline."""
    offset = 0.0
    for cut_start, cut_end in sorted(cut_list):
        if original_ts <= cut_start:
            break
        if original_ts >= cut_end:
            offset += (cut_end - cut_start)
        else:
            # Timestamp falls inside a cut — this word was removed
            return -1  # Signal: word was cut
    return original_ts - offset
```

---

## 11. AWS Infrastructure

### 11.1 Resource Map

```
AWS Account
|
+-- S3
|   +-- clipforge-uploads      (source videos, lifecycle: 7 days)
|   +-- clipforge-intermediate  (audio files, face data, lifecycle: 3 days)
|   +-- clipforge-renders       (output videos, lifecycle: 30 days)
|
+-- SQS
|   +-- clipforge-process       (video processing jobs — Phase 1)
|   +-- clipforge-render        (render jobs — Phase 2)
|   +-- clipforge-dlq           (dead letter queue)
|
+-- ECS Fargate (Spot + On-Demand fallback)
|   +-- clipforge-processor     (Phase 1: ingest, face track, LLM, audio prep)
|   |   CPU: 2048, Memory: 4096
|   |   Capacity: 80% FARGATE_SPOT / 20% FARGATE fallback
|   |
|   +-- clipforge-renderer      (Phase 2: FFmpeg render only)
|   |   CPU: 2048, Memory: 4096
|   |   Capacity: 80% FARGATE_SPOT / 20% FARGATE fallback
|   |   (Pro plan users: 100% FARGATE on-demand for priority rendering)
|   |
|   +-- ECR images:
|       - clipforge-processor   (Python 3.12 + FFmpeg 7 + MediaPipe + boto3)
|       - clipforge-renderer    (Python 3.12 + FFmpeg 7 + fonts + LUTs)
|
+-- RDS
|   +-- clipforge-db            (db.t4g.micro, PostgreSQL 16, 20GB gp3)
|
+-- CloudFront
|   +-- Distribution for clipforge-renders (signed URLs, 24hr cache)
|
+-- Secrets Manager
|   +-- anthropic-api-key
|   +-- modal-api-key
|   +-- stripe-secret-key
|   +-- clerk-secret-key
|
+-- CloudWatch
    +-- Alarms: DLQ depth > 0, task failures, render time > 180s
    +-- Log groups: /ecs/clipforge-processor, /ecs/clipforge-renderer
```

### 11.2 ECS Task Definitions

**Processor (Phase 1):** Heavier — runs face tracking + downloads video.
```json
{
  "family": "clipforge-processor",
  "cpu": "2048",
  "memory": "4096",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"]
}
```

**Renderer (Phase 2):** Also needs CPU for FFmpeg.
```json
{
  "family": "clipforge-renderer",
  "cpu": "2048",
  "memory": "4096",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"]
}
```

**Capacity Provider Strategy (both services):**
```json
{
  "capacityProviderStrategy": [
    {
      "capacityProvider": "FARGATE_SPOT",
      "weight": 4,
      "base": 0
    },
    {
      "capacityProvider": "FARGATE",
      "weight": 1,
      "base": 0
    }
  ]
}
```

This routes ~80% of tasks to Spot and ~20% to on-demand as fallback. If Spot capacity is unavailable, tasks automatically fall back to on-demand so no job gets stuck.

**Spot interruption handling:**
- Worker registers a SIGTERM handler on startup
- On 2-minute Spot warning: stop processing, do NOT delete the SQS message
- SQS visibility timeout (5 min) expires → message redelivers → new task picks it up
- All intermediate results are in S3/DB, so the retry resumes from the last completed step (e.g., if transcription was done, skip to face tracking)

**Priority rendering (Pro plan):**
Pro users get `capacityProvider: "FARGATE"` (on-demand only) — guaranteed no Spot interruption, slightly faster cold start. This is a real differentiator for the Pro tier.

### 11.3 Auto-Scaling

```yaml
# Process queue scaling
ProcessScaling:
  MinCapacity: 0
  MaxCapacity: 10
  TargetValue: 1  # 1 task per message
  ScaleOutCooldown: 30
  ScaleInCooldown: 120

# Render queue scaling
RenderScaling:
  MinCapacity: 0
  MaxCapacity: 15
  TargetValue: 1
  ScaleOutCooldown: 15   # Renders are fast, scale aggressively
  ScaleInCooldown: 60
```

### 11.4 S3 Lifecycle Rules

| Bucket | Expiry |
|--------|--------|
| clipforge-uploads | 7 days |
| clipforge-intermediate | 3 days |
| clipforge-renders | 30 days |

---

## 12. Worker Docker Images

### 12.1 Processor Image

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    libgl1-mesa-glx \
    libglib2.0-0 \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements-processor.txt .
RUN pip install --no-cache-dir -r requirements-processor.txt

COPY worker/processor/ /app/worker/processor/
COPY worker/shared/ /app/worker/shared/
WORKDIR /app

CMD ["python", "-m", "worker.processor.main"]
```

```
# requirements-processor.txt
boto3>=1.35
psycopg[binary]>=3.2
anthropic>=0.40
modal>=0.70
mediapipe>=0.10
pydantic>=2.9
numpy>=1.26
```

### 12.2 Renderer Image

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    fonts-montserrat \
    fonts-inter \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements-renderer.txt .
RUN pip install --no-cache-dir -r requirements-renderer.txt

COPY worker/renderer/ /app/worker/renderer/
COPY worker/shared/ /app/worker/shared/
COPY assets/fonts/ /app/assets/fonts/
COPY assets/luts/ /app/assets/luts/
WORKDIR /app

CMD ["python", "-m", "worker.renderer.main"]
```

```
# requirements-renderer.txt
boto3>=1.35
psycopg[binary]>=3.2
pydantic>=2.9
```

---

## 13. Frontend Pages

| Route | Description |
|-------|-------------|
| `/` | Landing page: before/after video hero, feature list, pricing, FAQ |
| `/dashboard` | Video list with status, stats, before/after scores |
| `/video/[id]` | Video detail: before/after players, toggles, stats, re-render, download |
| `/pricing` | Pricing page with Stripe checkout |
| `/sign-in` | Clerk sign-in |
| `/sign-up` | Clerk sign-up |

### 13.1 Dashboard

```
+--------------------------------------------------------------+
| ClipForge                                  [Creator] [Profile]  |
+--------------------------------------------------------------+
|                                                               |
| Your Videos                                  [+ Upload New]   |
|                                                               |
| +-----------------------------------------------------------+|
| | productivity-hack.mp4                                      ||
| | 58s → 44s | 7 fillers cut | noise cleaned | [Done]        ||
| | [View Before/After]  [Download]  [Re-render]               ||
| +-----------------------------------------------------------+|
| | morning-routine.mp4                                        ||
| | Processing... ████████░░ 78% — Tracking faces              ||
| +-----------------------------------------------------------+|
| | cooking-tip.mp4                                            ||
| | 45s → 38s | 3 fillers cut | color graded | [Done]         ||
| | [View Before/After]  [Download]  [Re-render]               ||
| +-----------------------------------------------------------+|
|                                                               |
| 2 of 3 free videos used this month  [Upgrade to Creator $15] |
+--------------------------------------------------------------+
```

### 13.2 Video Detail — Before/After + Controls

```
+------------------------------------------------------------------------+
| <- Back    productivity-hack.mp4                       [Download MP4]   |
+------------------------------------------------------------------------+
|                                                                         |
|  +---------------------------+    +---------------------------+         |
|  |        BEFORE             |    |        AFTER              |         |
|  |  +-------------------+   |    |  +-------------------+   |         |
|  |  |                   |   |    |  |                   |   |         |
|  |  |   Original video  |   |    |  |  Enhanced video   |   |         |
|  |  |   58 seconds      |   |    |  |  44 seconds       |   |         |
|  |  |   16:9            |   |    |  |  9:16, face-      |   |         |
|  |  |   noisy audio     |   |    |  |  tracked, zooms,  |   |         |
|  |  |   no captions     |   |    |  |  clean audio,     |   |         |
|  |  |                   |   |    |  |  captions          |   |         |
|  |  +-------------------+   |    |  +-------------------+   |         |
|  |  [Play/Pause]  0:24/0:58|    |  [Play/Pause]  0:18/0:44|         |
|  +---------------------------+    +---------------------------+         |
|                                                                         |
|  STATS: 14.1s saved | 7 fillers removed | 4 silence gaps cut           |
|         3 zoom points | noise cleaned | intro trimmed 3.8s             |
|                                                                         |
+------------------------------------------------------------------------+
|  TRANSFORMATIONS                                           [Re-render]  |
|                                                                         |
|  Core Edits:                        Audio:                              |
|  [x] Cut silence (3.2s saved)       [x] Noise removal                  |
|  [x] Cut filler words (7 found)     [x] Audio normalize + EQ           |
|  [x] Intro trim (3.8s cut)                                             |
|  [x] Loop optimize (2.5s trimmed)   Visuals:                           |
|                                      [x] Face-tracking crop             |
|  Captions:                           [x] Auto-zoom (3 points)           |
|  [x] Enabled                         [x] Color grade: [Warm v]          |
|  Style: [Bold Pop v]                 [ ] Speed ramp                     |
|                                                                         |
|  Extras:                                                                |
|  [ ] Hook text card: "I lost $50K learning this lesson"                |
|  [ ] CTA end card: "Follow for Part 2"                                 |
|  [ ] Progress bar                                                       |
|                                                                         |
+------------------------------------------------------------------------+
```

---

## 14. Monitoring + Alerts

| What | Tool | Alert Threshold |
|------|------|----------------|
| Processing failures | CloudWatch + SQS DLQ | Any message in DLQ |
| Worker task crashes | ECS + CloudWatch | Task exit code != 0 |
| API errors | Sentry (free tier) | Any 5xx error |
| Process queue depth | CloudWatch | > 15 messages for > 5 min |
| Render queue depth | CloudWatch | > 25 messages for > 5 min |
| Processing time | CloudWatch custom metric | > 180 sec for Phase 1 |
| Render time | CloudWatch custom metric | > 120 sec for Phase 2 |
| Face tracking failures | Custom metric | Detection < 50% of frames |
| Whisper failures | Modal dashboard | Any failure |
| Monthly AWS bill | AWS Budgets | > $150/mo |
| Stripe revenue | Stripe Dashboard | Daily check |

---

## 15. Security

| Concern | Mitigation |
|---------|-----------|
| Video file uploads | S3 presigned URLs (5 min expiry), max 500MB, content-type validation |
| API authentication | Clerk JWT verification on every request |
| Database access | RDS in private subnet, VPC-only access |
| Secrets | AWS Secrets Manager, never in env vars or code |
| Rendered video access | CloudFront signed URLs (24hr expiry) |
| Rate limiting | API Gateway: 10 req/sec per user |
| Input validation | Reject videos > 10 min, > 500MB, non-video MIME types |
| LLM prompt injection | Transcript is user content — sanitize, use system prompt separation |
| Face data privacy | Face coordinates only (no images stored), deleted with video |
| Video content moderation | Out of scope for MVP; add content scanning in v2 if abuse occurs |

---

## 16. Development Schedule

### Week 1: Upload + Audio + Transcription Pipeline
- [ ] Next.js project setup (App Router + Tailwind + Clerk + Drizzle)
- [ ] S3 presigned upload + video creation API
- [ ] Video validation (duration, size, codec, FFprobe)
- [ ] Audio extraction (FFmpeg WAV)
- [ ] Whisper transcription on Modal (word-level timestamps)
- [ ] DeepFilterNet noise removal on Modal
- [ ] Audio normalization + EQ (FFmpeg loudnorm + filters)
- [ ] SQS queue + ECS processor skeleton
- [ ] DB schema + Drizzle migrations
- [ ] Basic dashboard UI (upload + video list + status polling)

### Week 2: Video Transformations
- [ ] Silence detection from Whisper timestamps
- [ ] Filler word detection + LLM disambiguation
- [ ] MediaPipe face tracking (per-frame coordinates + smoothing)
- [ ] LLM analysis prompt (zoom points, intro trim, loop endpoint, emphasis)
- [ ] Cut list builder (silence + fillers + intro + outro → sorted cut list)
- [ ] FFmpeg filtergraph generator (dynamic, from toggle config)
- [ ] Face-tracking crop filter generation
- [ ] Auto-zoom keyframe generation
- [ ] Timestamp remapping for captions
- [ ] End-to-end processing pipeline test

### Week 3: Captions + Rendering + UI
- [ ] ASS caption generator (3 styles, emphasis highlighting, remapped timestamps)
- [ ] Color grade LUT integration (3 presets)
- [ ] Speed ramp filter generation
- [ ] Hook text card generation (prepend)
- [ ] CTA end card generation (append)
- [ ] Progress bar overlay
- [ ] Watermark for free tier
- [ ] ECS renderer worker (SQS → build filtergraph → FFmpeg → S3)
- [ ] Video detail page (before/after players + toggle controls)
- [ ] Re-render flow (toggle → POST /render → poll → show new video)
- [ ] Download with presigned URLs

### Week 4: Auth + Payments + Launch
- [ ] Stripe Checkout integration (Creator + Pro plans)
- [ ] Webhook handler (subscription lifecycle)
- [ ] Usage enforcement (video limits, feature gating per plan)
- [ ] Free tier watermark + feature restrictions
- [ ] Landing page (before/after hero, feature grid, pricing, FAQ)
- [ ] Error handling + retry logic for failed jobs
- [ ] CloudWatch alarms + Sentry setup
- [ ] Load testing (10 concurrent videos)
- [ ] Deploy to production
- [ ] Product Hunt / Twitter launch prep

---

## 17. Success Metrics

### North Star
**Weekly Active Processors (WAP):** Users who process + download at least 1 video per week.

### Key Metrics

| Metric | Month 1 | Month 3 | Month 6 |
|--------|---------|---------|---------|
| Signups | 300 | 2,000 | 6,000 |
| Videos processed | 500 | 5,000 | 25,000 |
| WAP | 60 | 400 | 1,500 |
| Free → Paid | 4% | 6% | 8% |
| Paid subscribers | 12 | 100 | 200 |
| MRR | $300 | $2,000 | $5,200 |
| Avg processing time | < 120 sec | < 90 sec | < 75 sec |
| Avg time saved per video | 10 sec | 12 sec | 14 sec |
| Re-render rate | 30% | 20% | 15% |
| Download completion | 75% | 85% | 90% |

---

## 18. Risks

| Risk | Mitigation |
|------|-----------|
| **Face tracking jittery on fast movement** | Exponential smoothing + interpolation. If face lost > 20% of frames, fall back to center crop and warn user |
| **Auto-zoom feels unnatural** | Conservative zoom (1.2-1.3x max), smooth ease-in/out. Let users disable per zoom point in v2 |
| **Filler word removal cuts content words** | LLM disambiguation step. "like" as content is kept. If confidence < 80%, keep the word (false negatives > false positives) |
| **Whisper quality on noisy audio** | DeepFilterNet runs BEFORE transcription would help (re-order in v2). For MVP, show transcript for manual correction |
| **FFmpeg filtergraph complexity** | Build filtergraph programmatically with unit tests. Each transform is a composable filter module |
| **Processing time > 2 min** | Parallelize: Whisper + DeepFilterNet run concurrently on Modal. Face tracking starts while waiting for LLM |
| **DeepFilterNet removes wanted audio** | Offer "noise removal intensity" slider (low/medium/high). Default to medium |
| **Color grading looks bad on some footage** | Default to OFF. Auto-suggest preset but let user choose. LUT preview thumbnails in v2 |
| **Re-render latency frustrating** | Cache intermediate data. Re-render only runs FFmpeg (~30 sec), skips all analysis |
| **Free tier abuse** | 1 video/month hard cap. Watermark. 720p. Max 3 min. Rate limit by IP for anonymous first video |

---

## 19. Post-MVP Roadmap (Build Only If Validated)

| Feature | Trigger |
|---------|---------|
| Per-zoom-point toggle (enable/disable individual zooms) | Users complain about specific zoom placements |
| Hook-first reorder (move best moment to start) | 30+ users request it |
| Transcript editing before processing | Users with heavy accents or niche terminology |
| Multi-speaker reframe (interview format) | Podcasters request it |
| Auto background music (royalty-free, auto-ducked) | Creators request it, licensing sorted |
| Batch upload (5-10 videos at once) | Agencies / power users |
| Direct post to TikTok / YouTube / IG | 50+ users request native posting |
| Custom caption fonts / brand kit | Pro users / agencies |
| More color grade presets (10+) | Users want variety |
| Analytics integration (connect TikTok/IG, see actual view data) | Users want to close the feedback loop |
| API access | B2B / developer interest |
| Multi-language support | Non-English creator growth |
| Mobile-optimized upload + review | Mobile traffic > 40% |
| Client-side preview (approximate, for toggle exploration) | Re-render latency complaints |

---

## 20. Competitive Positioning

| Tool | What It Does | ClipForge Difference |
|------|-------------|-------------------|
| **CapCut** | Manual video editor with auto-captions | Manual. Every zoom, cut, and effect is hand-placed. 30-60 min per short. ClipForge does it in 90 seconds, zero manual editing. |
| **Opus Clip** | Long-form → short clips | Different problem. Extracts clips from podcasts. ClipForge transforms existing shorts. |
| **Captions.ai** | Auto-captions + eye contact fix | Captions + 1 gimmick. Doesn't cut silence, doesn't remove fillers, doesn't auto-zoom, doesn't clean audio, doesn't face-track crop. |
| **Submagic** | Auto-captions with emojis | Captions-only. Adds text on top of video. The video itself is unchanged. |
| **Descript** | Transcript-based editor | Powerful but complex. Full NLE with steep learning curve. $24/mo. ClipForge is one-upload, zero-editing. |
| **Veed.io** | Online video editor + captions | General-purpose editor. Still requires manual trimming, manual effects. Not automated. |
| **Klap / Vizard** | Long-form → short clips | Same as Opus Clip — different input, different problem. |

**ClipForge's positioning:**

> "CapCut is Photoshop. ClipForge is a photo enhancer. You don't need to learn an editor. Upload your video. Get back a better video. That's it."

**The moat:** 10 real video transformations in a single pipeline. No other tool does auto-cut + face-track + zoom + noise removal + captions + intro trim + loop optimize + color grade in one upload. To replicate this, a competitor needs to build ALL of them and make them work together (timestamp remapping, filtergraph composition, cached intermediate data). That's months of engineering.

---

*Upload. Transform. Download. Post. Repeat 5x this week.*
