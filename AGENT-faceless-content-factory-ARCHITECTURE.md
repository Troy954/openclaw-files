# Faceless Content Factory — Architecture Document

## 1. System Overview

The system is a standalone video generation engine designed for autonomous operation. It consists of a PostgreSQL database for job management, a Node.js worker for video processing, and a minimal local web interface for quality control. There is no public-facing frontend, no authentication, and no multi-tenant architecture. This is a single-operator production system.

```
┌─────────────────────────────────────────────────────────────────┐
│                        INPUT METHODS                             │
│     CLI Commands  │  CSV Import  │  Direct DB Insert            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      POSTGRESQL DATABASE                         │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │   channels    │  │  video_jobs  │  │  generation_logs     │   │
│  │  (configs)    │  │   (queue)    │  │  (history + costs)   │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      VIDEO WORKER (Node.js)                      │
│                                                                  │
│  ┌─────────────┐  ┌─────────────────────────────────────────┐   │
│  │ Job Poller   │  │          Processing Pipeline            │   │
│  │ (5s cycle)   │  │                                         │   │
│  │             │  │  Script → Voice → Footage → Captions    │   │
│  │             │  │         → Assembly → Metadata            │   │
│  └─────────────┘  └─────────────────────────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  External APIs                           │    │
│  │   OpenAI GPT-4  │  ElevenLabs  │  Pexels  │  Whisper    │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      LOCAL FILE SYSTEM                           │
│                                                                  │
│  output/                                                         │
│  ├── {channel}/                                                  │
│  │   ├── {date}/                                                │
│  │   │   ├── {video-slug}/                                      │
│  │   │   │   ├── video.mp4                                      │
│  │   │   │   ├── thumbnail.jpg                                  │
│  │   │   │   └── metadata.json                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   QC INTERFACE (Local Web UI)                    │
│                                                                  │
│  Review Videos  │  Approve/Reject  │  View Logs  │  Monitor     │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Tech Stack

| Layer              | Technology                  | Purpose                              |
|--------------------|-----------------------------|--------------------------------------|
| Language           | TypeScript                  | Type safety across all components    |
| Database           | PostgreSQL (local Docker)   | Job queue and configuration storage  |
| Worker Runtime     | Node.js 20                  | Video processing orchestration       |
| AI — Scripts       | OpenAI GPT-4                | Script and metadata generation       |
| AI — Voice         | ElevenLabs                  | Text-to-speech voiceover             |
| AI — Captions      | OpenAI Whisper              | Timed caption generation             |
| Stock Footage      | Pexels Video API            | Free stock footage clips             |
| Video Assembly     | FFmpeg                      | Combine audio, video, captions       |
| QC Interface       | Express + simple HTML/JS    | Local review interface               |
| CLI                | Commander.js                | Command-line job management          |
| Process Manager    | PM2 or systemd              | Keep worker running continuously     |

## 3. Database Schema

Three core tables. No Row Level Security needed (single operator system).

### 3.1 channels

Configuration profiles for each content channel you operate.

| Column              | Type         | Notes                                         |
|---------------------|--------------|-----------------------------------------------|
| id                  | TEXT (PK)    | Channel identifier (e.g., "finance-shorts")   |
| name                | TEXT         | Display name                                  |
| niche               | TEXT         | Category (finance, motivation, tech, etc.)    |
| voice_id            | TEXT         | ElevenLabs voice ID                           |
| voice_settings      | JSONB        | Stability, similarity, style settings         |
| caption_style       | JSONB        | Font, size, color, position, animation        |
| content_guidelines  | TEXT         | Prompt context for tone and style             |
| output_directory    | TEXT         | Base path for this channel's outputs          |
| is_active           | BOOLEAN      | Whether to process jobs for this channel      |
| created_at          | TIMESTAMPTZ  | Auto-set                                      |
| updated_at          | TIMESTAMPTZ  | Auto-set                                      |

### 3.2 video_jobs

The job queue. Each row represents a video to be generated.

| Column              | Type         | Notes                                         |
|---------------------|--------------|-----------------------------------------------|
| id                  | UUID (PK)    | Auto-generated                                |
| channel_id          | TEXT (FK)    | References channels table                     |
| topic               | TEXT         | The video topic/title prompt                  |
| duration            | INTEGER      | Target length in seconds (30, 60, 90)         |
| status              | TEXT         | queued / processing / completed / failed      |
| current_stage       | TEXT         | script / voice / footage / captions / assembly / metadata |
| script              | TEXT         | Generated script (populated during processing)|
| footage_urls        | JSONB        | Array of Pexels video URLs selected           |
| voiceover_path      | TEXT         | Local path to generated audio file            |
| captions_path       | TEXT         | Local path to generated SRT/ASS file          |
| output_path         | TEXT         | Final video output directory path             |
| error_message       | TEXT         | Populated on failure                          |
| attempts            | INTEGER      | Retry counter (max 3)                         |
| cost_cents          | INTEGER      | Total API cost for this job in cents          |
| created_at          | TIMESTAMPTZ  | When job was queued                           |
| started_at          | TIMESTAMPTZ  | When processing began                         |
| completed_at        | TIMESTAMPTZ  | When processing finished                      |

Indexes: `status`, `channel_id`, `created_at`

### 3.3 generation_logs

Detailed logging for debugging and cost tracking.

| Column              | Type         | Notes                                         |
|---------------------|--------------|-----------------------------------------------|
| id                  | UUID (PK)    | Auto-generated                                |
| job_id              | UUID (FK)    | References video_jobs                         |
| stage               | TEXT         | Which pipeline stage                          |
| status              | TEXT         | started / completed / failed                  |
| duration_ms         | INTEGER      | How long the stage took                       |
| cost_cents          | INTEGER      | API cost for this stage                       |
| details             | JSONB        | Stage-specific data (tokens used, files, etc.)|
| error               | TEXT         | Error message if failed                       |
| created_at          | TIMESTAMPTZ  | Auto-set                                      |

Index: `job_id`

## 4. Project Structure

```
faceless-content-factory/
├── src/
│   ├── cli/
│   │   ├── index.ts                # CLI entry point
│   │   ├── commands/
│   │   │   ├── generate.ts         # Queue new video jobs
│   │   │   ├── import.ts           # Import jobs from CSV
│   │   │   ├── status.ts           # Check queue status
│   │   │   ├── retry.ts            # Retry failed jobs
│   │   │   ├── channels.ts         # Manage channel configs
│   │   │   └── logs.ts             # View generation logs
│   │   └── utils.ts
│   │
│   ├── worker/
│   │   ├── index.ts                # Worker entry point
│   │   ├── poller.ts               # Job queue polling logic
│   │   ├── pipeline.ts             # Orchestrates all stages
│   │   ├── stages/
│   │   │   ├── script-generator.ts     # GPT-4 script generation
│   │   │   ├── voice-synthesizer.ts    # ElevenLabs TTS
│   │   │   ├── footage-selector.ts     # Pexels API + download
│   │   │   ├── caption-generator.ts    # Whisper transcription
│   │   │   ├── video-assembler.ts      # FFmpeg composition
│   │   │   └── metadata-generator.ts   # GPT-4 title/desc/tags
│   │   └── utils/
│   │       ├── ffmpeg.ts               # FFmpeg wrapper functions
│   │       ├── file-manager.ts         # Output organization
│   │       └── cost-tracker.ts         # API cost calculation
│   │
│   ├── qc/
│   │   ├── server.ts               # Express server for QC UI
│   │   ├── routes/
│   │   │   ├── videos.ts           # Video listing and actions
│   │   │   ├── jobs.ts             # Job status and monitoring
│   │   │   └── channels.ts         # Channel management
│   │   └── public/
│   │       ├── index.html          # Simple review interface
│   │       ├── styles.css
│   │       └── app.js
│   │
│   ├── db/
│   │   ├── client.ts               # PostgreSQL client setup
│   │   ├── migrations/
│   │   │   └── 001_initial_schema.sql
│   │   └── queries/
│   │       ├── jobs.ts             # Job CRUD operations
│   │       ├── channels.ts         # Channel CRUD operations
│   │       └── logs.ts             # Log queries
│   │
│   ├── lib/
│   │   ├── openai.ts               # OpenAI client wrapper
│   │   ├── elevenlabs.ts           # ElevenLabs client wrapper
│   │   ├── pexels.ts               # Pexels client wrapper
│   │   ├── config.ts               # Environment configuration
│   │   └── logger.ts               # Logging utility
│   │
│   └── types/
│       └── index.ts                # Shared TypeScript types
│
├── output/                         # Generated videos (gitignored)
├── temp/                           # Working files during processing
├── docker-compose.yml              # PostgreSQL container
├── package.json
├── tsconfig.json
├── .env.example
└── README.md
```

## 5. Video Processing Pipeline

Each job passes through six stages in sequence. The worker updates `current_stage` as it progresses.

### Stage 1: Script Generation

**Input:** Topic, channel config (niche, content guidelines, duration)

**Process:**
1. Build prompt with topic, target duration, niche context, and style guidelines
2. Call GPT-4 with structured output request
3. Parse response for script text, scene descriptions, and visual cues

**Output:** Script text with embedded visual markers, stored on job row

**Prompt structure:**
```
You are a viral short-form video scriptwriter for {niche} content.

Write a {duration}-second script for: "{topic}"

Requirements:
- Hook in first 3 seconds that stops the scroll
- Clear value proposition by second 5
- Punchy, conversational tone
- End with engagement CTA (comment, follow, share)
- Include [VISUAL: description] markers for b-roll cues

{channel_content_guidelines}

Output the script only, no explanations.
```

### Stage 2: Voice Synthesis

**Input:** Script text, channel voice settings

**Process:**
1. Strip visual markers from script (keep only spoken text)
2. Call ElevenLabs API with voice ID and settings
3. Download MP3 audio file
4. Calculate actual audio duration

**Output:** MP3 file path, actual duration in seconds

### Stage 3: Footage Selection

**Input:** Script with visual markers, audio duration

**Process:**
1. Extract visual cues from script markers
2. Calculate time allocation per scene based on audio duration
3. Query Pexels Video API for each visual concept
4. Select best matches based on relevance and quality
5. Download video clips to temp directory
6. Trim clips to required durations

**Output:** Array of local video file paths with timing data

### Stage 4: Caption Generation

**Input:** Voiceover audio file

**Process:**
1. Call Whisper API with audio file
2. Get word-level timestamps
3. Generate ASS subtitle file with styling from channel config
4. Apply caption animation settings (word-by-word highlight, etc.)

**Output:** ASS subtitle file path

### Stage 5: Video Assembly

**Input:** Footage clips, audio file, subtitle file, channel config

**Process:**
1. Build FFmpeg filter complex:
   - Concatenate footage clips
   - Scale/crop to 1080x1920 (9:16)
   - Overlay audio track
   - Burn in subtitles with styling
   - Apply any color grading or effects
2. Export as H.264 MP4, 30fps, high quality
3. Generate thumbnail (extract compelling frame or generate separately)

**Output:** Final MP4 path, thumbnail path

### Stage 6: Metadata Generation

**Input:** Topic, script, channel config

**Process:**
1. Call GPT-4 with script and niche context
2. Generate click-optimized title (under 100 chars)
3. Generate description with hashtags
4. Generate platform-specific tags

**Output:** metadata.json file with title, description, tags, script

**Prompt structure:**
```
You are a YouTube Shorts / TikTok optimization expert.

Given this script for a {niche} video:
"{script}"

Generate:
1. A click-worthy title (max 100 chars) that creates curiosity
2. A description (2-3 sentences) with 5-10 relevant hashtags
3. 10 tags for SEO

Format as JSON: { "title": "...", "description": "...", "tags": [...] }
```

## 6. Output File Structure

After successful completion, each video is organized as:

```
output/
└── {channel_id}/
    └── {YYYY-MM-DD}/
        └── {video-slug}/
            ├── video.mp4           # Final rendered video
            ├── thumbnail.jpg       # Thumbnail image
            ├── metadata.json       # Title, description, tags, script
            └── assets/             # Optional: source files for debugging
                ├── script.txt
                ├── voiceover.mp3
                └── captions.ass
```

The `video-slug` is derived from the topic, sanitized for filesystem use.

## 7. CLI Commands

### Queue Management

```bash
# Add single video job
./fcf generate --channel finance-shorts --duration 60 --topic "5 ways to save money"

# Add multiple jobs from file (one topic per line)
./fcf generate --channel finance-shorts --duration 60 --topics-file topics.txt

# Import from CSV
./fcf import jobs.csv

# Check queue status
./fcf status

# View recent completions
./fcf status --completed --limit 20

# View failures
./fcf status --failed

# Retry all failed jobs
./fcf retry --all

# Retry specific job
./fcf retry --job-id abc123
```

### Channel Management

```bash
# List channels
./fcf channels list

# Add new channel
./fcf channels add --id tech-news --name "Tech News Daily" --niche tech --voice-id xxx

# Update channel config
./fcf channels update tech-news --voice-id yyy

# Disable channel (stop processing its jobs)
./fcf channels disable tech-news
```

### Logs and Monitoring

```bash
# View recent logs
./fcf logs --limit 50

# View logs for specific job
./fcf logs --job-id abc123

# Cost report
./fcf logs --costs --days 7
```

## 8. QC Interface

A minimal local web interface at `http://localhost:3333` for reviewing generated content.

### Views

**Dashboard:**
- Queue depth and processing status
- Today's completions count
- Failed jobs alert
- Cost summary (today, this week, this month)

**Review Queue:**
- Grid of videos pending review
- Filter by channel
- Each card shows: thumbnail, title, channel, duration
- Click to expand: video player, full metadata, approve/reject buttons

**Video Detail:**
- Video player
- Title, description, tags display
- Script display
- Approve button → moves to "approved" status
- Reject button → prompts for reason, marks rejected
- Flag button → marks for manual review of prompts

**Logs:**
- Filterable log viewer
- Job details drill-down
- Error inspection

### Actions

- **Approve:** Marks video as approved, optionally moves to upload-ready folder
- **Reject:** Marks as rejected with reason, excluded from stats
- **Bulk Approve:** Select multiple videos, approve all
- **Retry:** Re-queue a failed job
- **Delete:** Remove a job and its outputs

## 9. Configuration

### Environment Variables

```bash
# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/fcf

# OpenAI
OPENAI_API_KEY=sk-...

# ElevenLabs
ELEVENLABS_API_KEY=...

# Pexels
PEXELS_API_KEY=...

# Paths
OUTPUT_DIR=/path/to/output
TEMP_DIR=/path/to/temp

# Worker
WORKER_CONCURRENCY=2          # Parallel jobs (careful with API limits)
WORKER_POLL_INTERVAL_MS=5000  # How often to check for new jobs

# QC Server
QC_PORT=3333
```

## 10. Deployment Options

### Option A: Local Development Machine

Run everything locally for development and small-scale production.

- PostgreSQL via Docker Compose
- Worker runs via `npm run worker` or PM2
- QC interface runs via `npm run qc`
- Outputs stored on local filesystem

### Option B: Dedicated Server / VPS

For higher throughput and always-on operation.

- PostgreSQL installed directly or via Docker
- Worker managed by systemd or PM2
- QC interface behind nginx (optional auth)
- Outputs stored on server, synced to local via rsync

### Option C: Cloud (Railway / Render)

For fully managed infrastructure.

- PostgreSQL as managed service
- Worker as always-on process
- Outputs to S3 or similar object storage
- QC interface as web service (add basic auth)

## 11. Cost Model

### Per-Video Costs

| Service         | Cost          |
|-----------------|---------------|
| OpenAI GPT-4    | ~$0.15-0.20   |
| ElevenLabs      | ~$0.30        |
| Pexels          | Free          |
| Whisper         | ~$0.01        |
| **Total**       | **~$0.50**    |

### Monthly Projections

| Volume          | API Cost    | Notes                              |
|-----------------|-------------|------------------------------------|
| 50 videos/month | ~$25        | Testing / single channel           |
| 300 videos/month| ~$150       | 10 videos/day across channels      |
| 1000 videos/month| ~$500      | Scaled operation, multiple channels|

### Revenue Potential (for reference)

YouTube Shorts RPM varies ($0.05-0.20 per 1000 views). A channel with 1M monthly views might generate $50-200/month. Multiple channels and affiliate integration increase potential significantly.

## 12. Error Handling and Resilience

### Retry Logic

- Each job allows 3 attempts
- Exponential backoff between retries
- Different failure types:
  - **Transient:** API timeout, rate limit → auto-retry
  - **Content:** Policy violation, no footage found → mark failed, log for review
  - **System:** FFmpeg crash, disk full → alert, pause processing

### Failure Recovery

- Jobs in "processing" status for >30 minutes are reset to "queued"
- Worker heartbeat to detect crashes
- All intermediate files preserved on failure for debugging

### Monitoring Alerts

- Queue depth exceeds threshold
- Failure rate exceeds threshold
- API costs exceed daily budget
- Worker unresponsive
