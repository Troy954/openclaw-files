# Faceless Content Factory — Product Requirements Document

## 1. Overview

Faceless Content Factory is an autonomous video generation engine that produces faceless short-form videos at scale for your own YouTube, TikTok, and Instagram Reels channels. You provide topics in batch, the system generates complete videos — script, voiceover, stock footage, captions — and outputs them organized by channel, ready for upload. The goal is passive income through ad revenue and affiliate monetization across a portfolio of faceless content channels.

## 2. Problem Statement

Running multiple faceless content channels requires producing 5-10+ videos per day across niches. Manual production takes 4-6 hours per video. Even with editing skills, the volume required for meaningful ad revenue is unsustainable for a solo operator. You need a system that generates content autonomously while you sleep, allowing you to focus on strategy, quality control, and scaling to new niches.

## 3. Target User

You. This is not a SaaS product. This is an internal tool optimized for one operator running multiple faceless channels as a media business.

## 4. Core Value Proposition

Queue 50 topics before bed. Wake up to 50 upload-ready videos organized by channel, complete with titles, descriptions, and hashtags. Your only job is quality review and uploading.

## 5. System Capabilities

### 5.1 Batch Video Generation (Core Feature)

The system processes video jobs from a queue. Each job produces a complete, upload-ready video.

**Input per job:**
- Topic (e.g., "5 Passive Income Ideas for Beginners")
- Channel/Niche (e.g., "finance-channel-1")
- Video length (30s, 60s, or 90s)
- Voice profile (configurable per channel)

**Output per job:**
- MP4 video file (1080x1920, 9:16 vertical, H.264)
- Thumbnail image (auto-generated or extracted frame)
- Metadata file containing:
  - Suggested title (optimized for clicks)
  - Description with hashtags
  - Tags for platform SEO
  - The full script text

**Processing pipeline:**
1. GPT-4 generates the script with viral hook structure, pacing, and CTA
2. ElevenLabs converts script to voiceover audio
3. Pexels API selects relevant stock footage clips based on script keywords
4. Whisper generates timed caption data from the audio
5. FFmpeg assembles all assets into final MP4 with burned-in captions
6. GPT-4 generates optimized title, description, and tags
7. Output files are organized into channel-specific folders

### 5.2 Batch Input System

Multiple methods to queue video jobs:

**CSV Import:**
Upload a CSV with columns: topic, channel, duration, voice_profile. System parses and queues all rows as jobs.

**Database Direct:**
Insert rows directly into the `video_jobs` table. Useful for programmatic job creation or integrations.

**CLI Commands:**
Simple command-line interface for quick job creation:
```
./generate --channel finance-1 --duration 60 --topic "How compound interest works"
./generate --channel finance-1 --duration 60 --topics-file topics.txt
```

### 5.3 Channel Management

Channels are configuration profiles that define consistent output settings:

**Per-channel configuration:**
- Channel name/identifier
- Niche category
- Voice profile (ElevenLabs voice ID + settings)
- Caption style (font, color, position, animation)
- Content guidelines (tone, vocabulary, topics to avoid)
- Output directory path

**Channel examples:**
- `finance-shorts`: Deep narrator voice, blue/white captions, formal tone
- `ai-news`: Energetic host voice, tech-style captions, conversational tone
- `motivation-daily`: Inspirational voice, bold captions, uplifting tone

### 5.4 Output Organization

All outputs are organized for efficient review and upload:

```
output/
├── finance-shorts/
│   ├── 2024-01-15/
│   │   ├── passive-income-ideas/
│   │   │   ├── video.mp4
│   │   │   ├── thumbnail.jpg
│   │   │   └── metadata.json
│   │   ├── compound-interest-explained/
│   │   │   ├── video.mp4
│   │   │   ├── thumbnail.jpg
│   │   │   └── metadata.json
│   │   └── ...
├── ai-news/
│   ├── 2024-01-15/
│   │   └── ...
└── motivation-daily/
    └── ...
```

### 5.5 Quality Control Interface

A minimal local web UI for reviewing generated content:

- List of recent generations grouped by channel and date
- Video preview player
- Metadata display (title, description, tags)
- Approve / Reject / Flag buttons
- Rejection feedback (used to improve prompts over time)
- Bulk approve option for efficient review
- Export approved videos to upload-ready folder

### 5.6 Job Monitoring

Simple visibility into system status:

- Current queue depth
- Jobs processing / completed / failed
- Failure logs with error details
- Retry failed jobs command
- Cost tracking (API calls per job, running totals)

## 6. Features Explicitly Excluded

The following are not needed for autonomous operation:

- User authentication (you are the only user)
- Subscription/payment processing
- Multi-tenant data isolation
- Public-facing landing page
- Real-time progress animations
- User onboarding flows
- Team collaboration features
- API access for external developers

## 7. Operational Flows

### 7.1 Daily Content Generation

1. You prepare a list of topics (research trending topics, brainstorm ideas)
2. Import topics via CSV or CLI, assigning each to a channel
3. System queues all jobs
4. Worker processes jobs continuously (overnight or throughout day)
5. You review completed videos in the QC interface
6. Approve quality content, reject or flag poor outputs
7. Upload approved content to platforms manually (MVP) or via future automation

### 7.2 New Channel Launch

1. Create a new channel config (voice, style, niche guidelines)
2. Generate 10-20 test videos to calibrate quality
3. Review outputs, adjust prompts and settings
4. Once satisfied, begin regular content production
5. Monitor channel performance, adjust topic strategy

### 7.3 Handling Failures

1. Check job monitoring for failed jobs
2. Review error logs (API timeout? Content policy? FFmpeg error?)
3. Fix underlying issue if systemic
4. Retry failed jobs with single command
5. Permanently failed jobs logged for manual topic replacement

## 8. Success Metrics

- **Throughput:** Videos generated per day
- **Success rate:** % of jobs completing without failure
- **Approval rate:** % of generated videos you approve for upload
- **Cost per video:** Total API costs / videos generated
- **Time to review:** Minutes spent in QC per video
- **Channel growth:** Subscribers and views per channel (tracked externally)

## 9. Constraints & Assumptions

- Video generation takes approximately 3-8 minutes per video depending on length
- Worker can process jobs in parallel (configurable concurrency)
- Target throughput: 50-100 videos per day with single worker instance
- Pexels API provides sufficient stock footage variety across niches (free tier)
- ElevenLabs API cost approximately $0.30 per voiceover
- OpenAI GPT-4 cost approximately $0.15-0.20 per video (script + metadata)
- Total cost per video approximately $0.50
- English-language content only (MVP)
- Vertical 9:16 format only (optimized for Shorts/Reels/TikTok)
- Manual upload to platforms (auto-posting deferred to future phase)

## 10. Future Enhancements (Post-MVP)

After the core system is running and generating revenue:

1. **Auto-posting integration** — Direct upload to YouTube, TikTok, Instagram via APIs
2. **Scheduling system** — Queue videos for specific publish times
3. **Performance tracking** — Pull analytics from platforms, correlate with content attributes
4. **Topic research automation** — AI identifies trending topics in each niche
5. **A/B thumbnail generation** — Multiple thumbnail options per video
6. **Voice cloning** — Custom voices for brand differentiation
7. **Music library** — Background music selection and licensing
8. **Horizontal format support** — Standard YouTube videos, not just Shorts
