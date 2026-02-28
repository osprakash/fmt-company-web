# Product Requirements Document (PRD)
## FlowCut - Add Captions to Your Videos in Seconds

**Version:** 1.0  
**Date:** February 28, 2026  
**Status:** MVP Scope Locked

---

## 1. Executive Summary

### 1.1 Product Vision
FlowCut is a **browser-based caption tool** for short-form video creators. Drop a video, get styled captions, export. No upload, no wait, no account required.

### 1.2 Problem Statement
Creators need to add captions to their videos. Current options:
- **Server-based tools** (VEED, Kapwing): Upload, wait 5-10 minutes, download
- **Desktop apps** (Premiere, CapCut Desktop): Overkill for just adding captions
- **Mobile apps** (CapCut): Awkward for longer editing sessions

### 1.3 Solution
A browser-native caption editor:
- **Instant processing**: Video never uploads. Transcription happens on your device.
- **Styled captions**: 5+ animation presets (karaoke, bounce, glow, etc.)
- **Simple trimming**: Cut your video to the best segment
- **One-click export**: MP4 ready for TikTok, Reels, Shorts

### 1.4 Target User
Short-form creators who post 3+ times per week and need fast caption turnaround.

### 1.5 Business Goal
**$5K MRR within 12 months** through a simple freemium model.

---

## 2. MVP Scope (4 Weeks)

### 2.1 Week 1-2: Captions

| Feature | Description | Priority |
|---------|-------------|----------|
| Video upload | Drag & drop MP4/WebM/MOV (max 200MB) | P0 |
| Transcription | Whisper Web (client-side), auto language detection | P0 |
| Caption preview | Real-time preview with word highlighting | P0 |
| Caption styles | 5 presets: Classic, Karaoke, Bounce, Glow, Minimal | P0 |
| Style customization | Font, size, color, position, background | P0 |
| Transcript editing | Fix transcription errors manually | P0 |
| Export with captions | Burn captions into video, export MP4 | P0 |

### 2.2 Week 3-4: Trimming + Polish

| Feature | Description | Priority |
|---------|-------------|----------|
| Timeline trim | Set start/end points with visual timeline | P0 |
| Waveform display | Audio waveform for easier trimming | P1 |
| Export quality | 720p (free), 1080p (paid) | P0 |
| Watermark | "Made with FlowCut" on free exports | P0 |
| User accounts | Email + Google OAuth | P0 |
| Payments | Stripe integration, monthly subscription | P0 |

### 2.3 Explicitly NOT in MVP

| Feature | Why Deferred |
|---------|--------------|
| Aspect ratio reframing | Adds 2+ weeks of complexity |
| Filler word detection | Nice-to-have, not core value |
| Transcript-based editing | Can add later if users request |
| Text overlays/titles | Scope creep |
| Brand templates | Premature optimization |
| Cloud processing | Only needed for large files |
| Multi-language UI | English only for MVP |

---

## 3. Technical Architecture

### 3.1 Stack

| Layer | Technology |
|-------|------------|
| Frontend | React + TypeScript |
| Styling | Tailwind CSS |
| Video processing | FFmpeg.wasm |
| Transcription | Whisper Web (WebGPU/WASM) |
| Caption rendering | Canvas API |
| Storage | IndexedDB (local), PostgreSQL (accounts) |
| Auth | NextAuth.js |
| Payments | Stripe |
| Hosting | Vercel |

### 3.2 Browser Requirements
- Chrome 94+, Firefox 88+, Edge 94+
- Safari 15.2+ (limited WebCodecs support)
- Minimum 4GB RAM, 8GB recommended
- Initial model download: ~150MB (Whisper model)

### 3.3 Video Limitations

| Constraint | Free | Paid |
|------------|------|------|
| Max file size | 200MB | 300MB |
| Max duration | 10 min | 15 min |
| Export quality | 720p | 1080p |
| Watermark | Yes | No |

---

## 4. User Flow

```
1. LAND           2. DROP           3. STYLE          4. EXPORT
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ See demo │ ──▶ │ Drop     │ ──▶ │ Pick     │ ──▶ │ Trim     │
│ video    │     │ video    │     │ caption  │     │ (optional)│
│          │     │          │     │ style    │     │          │
│ No signup│     │ Auto-    │     │ Edit     │     │ Export   │
│ needed   │     │ transcribe│    │ text     │     │ MP4      │
└──────────┘     └──────────┘     └──────────┘     └──────────┘

Time: 0 sec       Time: 30 sec      Time: 2 min       Time: 3 min
```

### 4.1 First-Time User Experience
1. Land on homepage → See 30-second demo of captions being added
2. Drop video (no account needed)
3. Wait for transcription (~30 sec for 5 min video)
4. Pick caption style, adjust if needed
5. Preview result
6. Export → First export free (720p, watermarked)
7. Prompt: "Remove watermark + 1080p for $9/mo"

---

## 5. Pricing

### 5.1 Model: Output-Gated Freemium

All features free to use. Pay to export without limitations.

| | **Free** | **Pro** |
|--|----------|---------|
| **Price** | $0 | $9/mo |
| Exports/month | 3 | Unlimited |
| Export quality | 720p | 1080p |
| Watermark | Yes | No |
| Max file size | 200MB | 300MB |
| Max duration | 10 min | 15 min |

### 5.2 Why This Pricing
- **$9/mo is impulse-buy territory** for creators
- **720p looks noticeably soft** on TikTok/Reels (1080p native) — quality alone drives upgrades
- **3 exports/month** lets users validate the tool but not sustain a posting schedule
- **Watermark** is free advertising + conversion pressure

### 5.3 Revenue Target

| Milestone | Subscribers | MRR |
|-----------|-------------|-----|
| Month 3 | 100 | $900 |
| Month 6 | 300 | $2,700 |
| Month 12 | 600 | $5,400 |

Conservative estimates. No viral growth assumed.

---

## 6. Go-to-Market

### 6.1 Positioning
**"Add captions to your videos in 60 seconds. No upload. No wait."**

### 6.2 Launch Channels
1. **Product Hunt** — Target top 5 of the day
2. **Reddit** — r/NewTubers, r/TikTokCreators, r/videography
3. **Twitter/X** — Build in public, share progress
4. **YouTube** — Short tutorial: "How I add captions to my videos for free"

### 6.3 Competitive Angle
| vs Competitor | Our Advantage |
|---------------|---------------|
| VEED, Kapwing | No upload wait — instant processing |
| CapCut Desktop | No install — works in browser |
| CapCut Mobile | Better for longer editing sessions |
| Descript | Simpler, cheaper, faster for just captions |

---

## 7. Success Metrics

### 7.1 North Star
**Monthly Active Exporters (MAE)**: Users who export at least one video per month.

### 7.2 Key Metrics

| Metric | Month 1 | Month 3 | Month 6 |
|--------|---------|---------|---------|
| Signups | 500 | 3,000 | 10,000 |
| MAE | 150 | 900 | 3,000 |
| Free → Paid | 3% | 5% | 6% |
| Paid subscribers | 5 | 100 | 300 |
| MRR | $45 | $900 | $2,700 |
| Export success rate | 90% | 95% | 98% |

### 7.3 Tracking
- **Analytics**: Mixpanel (free tier)
- **Error tracking**: Sentry (free tier)
- **Payments**: Stripe Dashboard

---

## 8. Risks

| Risk | Mitigation |
|------|------------|
| Whisper model fails to load | Show clear error, suggest Chrome, retry button |
| Export crashes on large files | Enforce file size limits strictly |
| Safari compatibility issues | Show browser recommendation banner |
| Users expect more features | Clear positioning: "captions + trim, that's it" |
| Low conversion rate | A/B test watermark prominence, export limit |

---

## 9. Development Schedule

### Week 1: Core Transcription
- [ ] Project setup (React + Vite + Tailwind)
- [ ] Video upload component (drag & drop)
- [ ] Whisper Web integration
- [ ] Basic transcript display

### Week 2: Caption Styling + Export
- [ ] Caption style presets (5 styles)
- [ ] Style customization UI
- [ ] Canvas-based caption rendering
- [ ] FFmpeg.wasm export with burned-in captions

### Week 3: Trimming + Timeline
- [ ] Timeline component with trim handles
- [ ] Waveform visualization
- [ ] Trim preview
- [ ] Export trimmed segment

### Week 4: Payments + Polish
- [ ] NextAuth.js (email + Google OAuth)
- [ ] Stripe subscription integration
- [ ] Watermark on free exports
- [ ] Export limits enforcement
- [ ] Landing page
- [ ] Bug fixes + testing

---

## 10. Post-MVP Roadmap (If Validated)

Only build these if users request them:

| Feature | Trigger to Build |
|---------|------------------|
| Aspect ratio reframing | 20+ user requests |
| Filler word removal | 20+ user requests |
| More caption styles | Users complain about limited options |
| Transcript-based editing | Power users request it |
| Cloud processing | Users hitting file size limits |
| Team features | Business inquiries |

---

## 11. Definition of Done (MVP)

MVP is complete when:
- [ ] User can drop a video and see transcript in <60 seconds
- [ ] User can apply any of 5 caption styles
- [ ] User can trim video with timeline
- [ ] User can export 720p MP4 with watermark (free)
- [ ] User can pay $9/mo for 1080p without watermark
- [ ] Export success rate >90%
- [ ] Works in Chrome, Firefox, Edge

---

*Ship fast. Validate demand. Iterate based on paying users, not assumptions.*
