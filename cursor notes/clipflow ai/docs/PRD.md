# Product Requirements Document (PRD)
## ClipFlow.ai - Browser-First Video Repurposing Platform

**Version:** 1.0  
**Date:** February 23, 2026  
**Status:** Draft

---

## 1. Executive Summary

### 1.1 Product Vision
ClipFlow.ai is a browser-first video repurposing SaaS that transforms long-form videos into short-form content optimized for social platforms. Our core differentiator is **client-side processing** using modern browser APIs (WebCodecs, FFmpeg.wasm, Whisper Web), dramatically reducing infrastructure costs and enabling a generous free tier that competitors cannot match.

### 1.2 Problem Statement
Content creators need to repurpose long-form content into shorts for TikTok, YouTube Shorts, and Instagram Reels. Current solutions like OpusClip charge $15-29/month with limited processing minutes because they rely on expensive server-side compute. Many creators, especially beginners, cannot justify this cost.

### 1.3 Solution
A hybrid processing model:
- **Free Tier**: Full-featured browser-based processing (no server costs = sustainable free tier)
- **Paid Tiers**: Server-side AI processing for advanced features (smart clip detection, AI enhancements)

---

## 2. Market Analysis

### 2.1 Competitive Landscape

| Competitor | Pricing | Processing | Key Strengths | Key Weaknesses |
|------------|---------|------------|---------------|----------------|
| **OpusClip** | $0-29/mo | Server-side | Virality scoring, B-roll, brand templates | Expensive, limited free tier (60 min/mo watermarked) |
| **Descript** | $12-24/mo | Server-side | Transcript-based editing, audio cleanup | Per-seat pricing, not clip-focused |
| **VEED** | $0-24/mo | Server-side | All-in-one editor, team features | Generalist tool, not optimized for clipping |
| **Reap** | $9-29/mo | Server-side | Multi-language dubbing, scheduling | Complex for simple use cases |
| **ClipFinder** | $2/hour | Server-side | Budget-friendly | Limited features |
| **Eklipse** | Free-$15/mo | Server-side | Gaming-focused | Niche audience |

### 2.2 Market Gap
No competitor offers **truly unlimited free processing** because server compute is expensive. Browser-based processing eliminates this constraint.

### 2.3 Target Users
1. **Primary**: Solo content creators (YouTubers, podcasters, educators) with 1K-100K followers
2. **Secondary**: Small agencies managing 2-5 creator accounts
3. **Tertiary**: Enterprise teams (future expansion)

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
- Brand templates and asset libraries
- Social media scheduling
- Analytics dashboard

---

## 5. Pricing Model

### 5.1 Credit System

**Credit Definition**: 1 credit = 1 minute of server-side AI processing

**Credit Costs by Operation**:
| Operation | Credits | Processing Location |
|-----------|---------|---------------------|
| Smart Captions | 0 | Browser (free) |
| Aspect Ratio Reframe | 0 | Browser (free) |
| Video Trim & Export | 0 | Browser (free) |
| AI Clip Detection | 5/min | Server |
| AI B-Roll Generation | 10/clip | Server |
| AI Voice Enhancement | 3/min | Server |
| 4K Export | 2/min | Server |

### 5.2 Pricing Tiers

| Tier | Price | Credits | Key Features |
|------|-------|---------|--------------|
| **Free** | $0/mo | 0 | Unlimited browser processing, watermark on exports |
| **Starter** | $9/mo | 100 | No watermark, priority support |
| **Pro** | $19/mo | 300 | AI features, cloud storage (10GB) |
| **Business** | $49/mo | 1000 | Team seats (3), API access, 50GB storage |

### 5.3 Credit Top-Up
- 50 credits: $5
- 150 credits: $12 (20% bonus)
- 500 credits: $35 (30% bonus)

---

## 6. User Experience

### 6.1 User Flow (MVP)

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIPFLOW.AI                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. UPLOAD          2. EDIT              3. EXPORT              │
│  ┌──────────┐      ┌──────────────┐     ┌──────────────┐       │
│  │ Drag &   │ ──▶  │ • Trim       │ ──▶ │ • Choose     │       │
│  │ Drop     │      │ • Captions   │     │   format     │       │
│  │ Video    │      │ • Reframe    │     │ • Download   │       │
│  └──────────┘      └──────────────┘     └──────────────┘       │
│                                                                  │
│  Supported: MP4, WebM, MOV (up to 500MB free, 2GB paid)        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Key UX Principles
1. **Zero friction start**: No signup required for first export (watermarked)
2. **Progressive disclosure**: Simple interface with advanced options tucked away
3. **Real-time preview**: All edits visible instantly before export
4. **Transparent processing**: Clear progress indicators showing browser activity

### 6.3 Onboarding Flow
1. Land on homepage → See demo video
2. Upload first video (no account needed)
3. Apply captions + reframe
4. Preview result
5. Export (watermarked) or signup to remove watermark

---

## 7. Technical Constraints

### 7.1 Browser Requirements
- Chrome 90+, Firefox 88+, Safari 15+, Edge 90+
- WebGPU support recommended for faster transcription
- Minimum 4GB RAM, 8GB recommended
- Stable internet for initial model downloads (~100MB)

### 7.2 Video Limitations (Browser Processing)
| Constraint | Free Tier | Paid Tiers |
|------------|-----------|------------|
| Max file size | 500MB | 2GB |
| Max duration | 30 minutes | 2 hours |
| Max resolution | 1080p | 4K (server) |
| Concurrent exports | 1 | 3 |

### 7.3 Known Limitations
- Safari has limited WebCodecs support (fallback to FFmpeg.wasm only)
- Mobile browsers not supported in MVP (desktop-first)
- Very long videos (>1hr) may cause memory issues on low-RAM devices

---

## 8. Success Metrics

### 8.1 North Star Metric
**Monthly Active Exporters (MAE)**: Users who successfully export at least one video per month

### 8.2 Key Performance Indicators

| Metric | Target (Month 3) | Target (Month 6) |
|--------|------------------|------------------|
| Registered users | 10,000 | 50,000 |
| Monthly Active Exporters | 3,000 | 15,000 |
| Free → Paid conversion | 3% | 5% |
| Export success rate | 95% | 99% |
| Average exports/user/month | 4 | 8 |
| NPS Score | 40 | 50 |

### 8.3 Tracking Plan
- Mixpanel/Amplitude for user analytics
- Sentry for error tracking
- Custom metrics for processing performance

---

## 9. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Browser API compatibility issues | High | Medium | Feature detection + graceful fallbacks |
| Memory crashes on large files | High | Medium | Chunked processing, clear file limits |
| Whisper model download failures | Medium | Low | CDN redundancy, offline caching |
| Competition copies browser approach | Medium | Medium | First-mover advantage, rapid iteration |
| Users expect server-quality from browser | Medium | High | Clear messaging about processing location |

---

## 10. Launch Plan

### 10.1 MVP Scope
- Smart Captions (5 style presets)
- Aspect Ratio Reframe (face detection)
- Video Trim & Export (1080p MP4)
- User accounts (email + Google OAuth)
- Basic analytics dashboard

### 10.2 Launch Phases
1. **Alpha** (Internal): Core team testing
2. **Beta** (Invite-only): 500 creators from waitlist
3. **Public Launch**: Product Hunt + social media campaign

### 10.3 Go-to-Market
- Target: YouTube/TikTok creator communities
- Messaging: "Unlimited free video editing - no catch"
- Channels: Product Hunt, Twitter/X, YouTube tutorials, Reddit

---

## 11. Appendix

### 11.1 Glossary
- **WebCodecs**: Browser API for low-level video encoding/decoding
- **FFmpeg.wasm**: FFmpeg compiled to WebAssembly for browser use
- **Whisper Web**: OpenAI's Whisper model running in browser via WebGPU
- **MediaPipe**: Google's ML framework for face/object detection

### 11.2 References
- [OpusClip](https://www.opus.pro/) - Primary competitor
- [WebCodecs API](https://developer.mozilla.org/en-US/docs/Web/API/WebCodecs_API)
- [FFmpeg.wasm](https://ffmpegwasm.netlify.app/)
- [Whisper Web](https://whisperweb.dev/)
- [MediaPipe Face Detection](https://ai.google.dev/edge/mediapipe/solutions/vision/face_detector/web_js)

---

*Document maintained by: Product Team*  
*Last updated: February 23, 2026*
