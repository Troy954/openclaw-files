# Faceless Content Factory — Build Plan

## Overview

This build plan is optimized for one goal: get to generating videos as fast as possible. No SaaS features, no user-facing frontend, no payment processing. Just the core engine that produces videos from topics.

The estimated timeline is 2-3 weeks for a focused solo developer. Each phase produces a working increment you can test before moving on.

---

## Phase 1: Project Setup & Database

**Goal:** Skeleton project with database running and basic CLI structure.

**Timeline:** 1 day

### Steps

1. Create project directory with the following structure:
   ```
   faceless-content-factory/
   ├── src/
   │   ├── cli/
   │   ├── worker/
   │   ├── db/
   │   ├── lib/
   │   └── types/
   ├── output/
   ├── temp/
   └── docker-compose.yml
   ```

2. Initialize with `npm init`, install core dependencies:
   - `typescript`, `ts-node`, `@types/node`
   - `pg` (PostgreSQL client)
   - `commander` (CLI framework)
   - `dotenv`

3. Create `docker-compose.yml` for PostgreSQL:
   ```yaml
   services:
     db:
       image: postgres:16
       environment:
         POSTGRES_USER: fcf
         POSTGRES_PASSWORD: fcf
         POSTGRES_DB: fcf
       ports:
         - "5432:5432"
       volumes:
         - pgdata:/var/lib/postgresql/data
   volumes:
     pgdata:
   ```

4. Create database migration `src/db/migrations/001_initial_schema.sql`:
   - `channels` table
   - `video_jobs` table
   - `generation_logs` table

5. Create `src/db/client.ts` with PostgreSQL connection setup

6. Create migration runner script

7. Create `.env.example` with all required variables

8. Run `docker-compose up -d` and apply migrations

9. Create basic CLI entry point `src/cli/index.ts` with Commander.js

10. Verify: `npx ts-node src/cli/index.ts --help` shows usage

### Deliverable

Project structure in place, PostgreSQL running, empty CLI framework ready.

---

## Phase 2: Channel Configuration

**Goal:** Ability to create and manage channel configs via CLI.

**Timeline:** 0.5 days

### Steps

1. Create `src/db/queries/channels.ts` with CRUD functions:
   - `createChannel(config)`
   - `getChannel(id)`
   - `listChannels()`
   - `updateChannel(id, updates)`
   - `deleteChannel(id)`

2. Create `src/cli/commands/channels.ts`:
   - `channels list` — show all channels
   - `channels add` — interactive or flag-based channel creation
   - `channels update` — modify existing channel
   - `channels delete` — remove channel

3. Define TypeScript types in `src/types/index.ts`:
   ```typescript
   interface Channel {
     id: string;
     name: string;
     niche: string;
     voice_id: string;
     voice_settings: VoiceSettings;
     caption_style: CaptionStyle;
     content_guidelines: string;
     output_directory: string;
     is_active: boolean;
   }
   ```

4. Test: Create a test channel via CLI, verify it appears in database

### Deliverable

Working channel management. You can create channel profiles that will be used for video generation.

---

## Phase 3: Job Queue System

**Goal:** Ability to queue video jobs and check their status.

**Timeline:** 0.5 days

### Steps

1. Create `src/db/queries/jobs.ts` with queue functions:
   - `createJob(channelId, topic, duration)`
   - `createJobsBatch(jobs[])` — for CSV import
   - `getNextQueuedJob()` — for worker polling
   - `updateJobStatus(id, status, updates)`
   - `getJobsByStatus(status)`
   - `getRecentJobs(limit)`

2. Create `src/cli/commands/generate.ts`:
   - `generate --channel X --topic "..." --duration 60` — queue single job
   - `generate --channel X --topics-file topics.txt` — queue from file

3. Create `src/cli/commands/import.ts`:
   - `import jobs.csv` — parse CSV and queue all jobs

4. Create `src/cli/commands/status.ts`:
   - `status` — show queue summary (queued, processing, completed, failed)
   - `status --queued` — list queued jobs
   - `status --completed --limit 10` — list recent completions
   - `status --failed` — list failures

5. Test: Queue 5 jobs via CLI, verify they appear in database with "queued" status

### Deliverable

Working job queue. You can add topics and see them waiting for processing.

---

## Phase 4: Script Generation Stage

**Goal:** Worker can pick up a job and generate a script using GPT-4.

**Timeline:** 1 day

### Steps

1. Install OpenAI SDK: `npm install openai`

2. Create `src/lib/openai.ts` with configured client

3. Create `src/lib/config.ts` centralizing all environment variables

4. Create `src/worker/index.ts` — worker entry point:
   - Poll database every 5 seconds for queued jobs
   - Claim job by setting status to "processing"
   - Pass to pipeline
   - Handle errors, update status

5. Create `src/worker/poller.ts` with polling logic:
   ```typescript
   async function pollForJobs() {
     const job = await getNextQueuedJob();
     if (job) {
       await processJob(job);
     }
   }
   ```

6. Create `src/worker/stages/script-generator.ts`:
   - Build prompt from topic, channel config, duration
   - Call GPT-4
   - Parse response
   - Return script text with visual markers

7. Create `src/worker/pipeline.ts` — orchestrator (initially just calls script stage):
   - Update `current_stage` as processing
   - Call script generator
   - Save script to job row
   - Mark as completed (temporarily, until other stages exist)

8. Create `src/db/queries/logs.ts` for generation logging

9. Test: Queue a job, run worker, verify script appears in database

### Deliverable

Worker generates scripts. You can queue a topic and get back an AI-written script.

---

## Phase 5: Voice Synthesis Stage

**Goal:** Convert scripts to voiceover audio using ElevenLabs.

**Timeline:** 1 day

### Steps

1. Install axios for API calls: `npm install axios`

2. Create `src/lib/elevenlabs.ts`:
   - `synthesizeSpeech(text, voiceId, settings)` — returns audio buffer
   - Handle API errors and rate limits

3. Create `src/worker/stages/voice-synthesizer.ts`:
   - Strip visual markers from script (extract spoken text only)
   - Call ElevenLabs API
   - Save MP3 to temp directory
   - Calculate audio duration using ffprobe
   - Return file path and duration

4. Install fluent-ffmpeg for audio/video operations:
   `npm install fluent-ffmpeg @types/fluent-ffmpeg`

5. Create `src/worker/utils/ffmpeg.ts` with helper functions:
   - `getAudioDuration(filePath)`
   - Verify FFmpeg is installed

6. Update pipeline to call voice stage after script stage

7. Test: Queue a job, verify MP3 file is generated in temp directory

### Deliverable

Worker generates voiceovers. Scripts become audio files.

---

## Phase 6: Footage Selection Stage

**Goal:** Find and download relevant stock footage from Pexels.

**Timeline:** 1-2 days

### Steps

1. Create `src/lib/pexels.ts`:
   - `searchVideos(query, options)` — search for videos
   - `downloadVideo(url, outputPath)` — download video file

2. Create `src/worker/stages/footage-selector.ts`:
   - Parse visual markers from script
   - For each marker, search Pexels for relevant footage
   - Score and select best matches
   - Download selected clips to temp directory
   - Calculate time allocation based on audio duration
   - Trim clips to required lengths using FFmpeg
   - Return array of clip paths with timing data

3. Add FFmpeg helper for trimming:
   - `trimVideo(inputPath, outputPath, startTime, duration)`

4. Handle edge cases:
   - No results for a query → try broader terms
   - Not enough footage → extend clips or add transitions

5. Update pipeline to call footage stage

6. Test: Queue a job, verify video clips are downloaded and trimmed

### Deliverable

Worker selects and prepares stock footage for each video.

---

## Phase 7: Caption Generation Stage

**Goal:** Generate timed captions from voiceover audio.

**Timeline:** 0.5-1 day

### Steps

1. Create `src/worker/stages/caption-generator.ts`:
   - Send audio to Whisper API
   - Parse word-level timestamps
   - Generate ASS subtitle file with styling
   - Apply channel's caption style config

2. Create `src/lib/captions.ts`:
   - `generateASSFile(words, style)` — create styled subtitle file
   - Support word-by-word highlighting
   - Support different animation styles

3. Update pipeline to call caption stage

4. Test: Queue a job, verify ASS file is generated with proper timing

### Deliverable

Worker generates styled captions synced to voiceover.

---

## Phase 8: Video Assembly Stage

**Goal:** Combine all assets into final MP4 video.

**Timeline:** 1-2 days

### Steps

1. Create `src/worker/stages/video-assembler.ts`:
   - Build FFmpeg filter complex:
     - Concatenate footage clips in sequence
     - Scale/crop to 1080x1920 (9:16 vertical)
     - Overlay voiceover audio
     - Burn in captions
   - Export as H.264 MP4

2. Create `src/worker/utils/file-manager.ts`:
   - `createOutputDirectory(channelId, date, slug)` — create organized folder
   - `moveToOutput(tempPath, outputDir)` — move final files
   - `cleanupTemp(jobId)` — remove intermediate files

3. Add thumbnail generation:
   - Extract frame at ~3 seconds (or analyze for best frame)
   - Save as JPEG

4. Update pipeline:
   - Call assembly stage
   - Move outputs to organized directory structure
   - Update job with output path

5. Test: Queue a job, verify complete MP4 is generated and plays correctly

### Deliverable

End-to-end video generation works. Topic goes in, MP4 comes out.

---

## Phase 9: Metadata Generation Stage

**Goal:** Generate optimized titles, descriptions, and tags for each video.

**Timeline:** 0.5 days

### Steps

1. Create `src/worker/stages/metadata-generator.ts`:
   - Send script and channel context to GPT-4
   - Request JSON output with title, description, tags
   - Parse and validate response

2. Create `metadata.json` output:
   ```json
   {
     "title": "...",
     "description": "...",
     "tags": ["...", "..."],
     "script": "...",
     "generated_at": "...",
     "duration_seconds": 60,
     "channel": "finance-shorts"
   }
   ```

3. Update pipeline to call metadata stage and write JSON file

4. Test: Queue a job, verify metadata.json is created with all fields

### Deliverable

Each video comes with platform-ready metadata.

---

## Phase 10: Batch Processing & Robustness

**Goal:** Handle multiple jobs reliably with proper error recovery.

**Timeline:** 1 day

### Steps

1. Implement retry logic in worker:
   - Catch errors at each stage
   - Increment attempts counter
   - If attempts < 3, reset to queued with backoff
   - If attempts >= 3, mark as failed with error message

2. Implement job timeout handling:
   - Jobs stuck in "processing" for >30 minutes reset to queued
   - Periodic cleanup task

3. Add cost tracking:
   - Calculate cost at each API stage
   - Sum total cost on job row
   - Log to generation_logs

4. Create `src/cli/commands/retry.ts`:
   - `retry --all` — retry all failed jobs
   - `retry --job-id X` — retry specific job

5. Add concurrency support (optional):
   - Process multiple jobs in parallel
   - Respect API rate limits
   - Configure via WORKER_CONCURRENCY env var

6. Test: Queue 10 jobs, simulate some failures, verify retry and completion

### Deliverable

Robust batch processing. Queue many jobs, walk away, come back to completed videos.

---

## Phase 11: QC Interface

**Goal:** Simple web UI to review and approve generated videos.

**Timeline:** 1-2 days

### Steps

1. Install Express: `npm install express`

2. Create `src/qc/server.ts`:
   - Express app on port 3333
   - Serve static files from `src/qc/public`
   - API routes for video data

3. Create API routes `src/qc/routes/`:
   - `GET /api/videos` — list videos with filters
   - `GET /api/videos/:id` — single video details
   - `POST /api/videos/:id/approve` — mark approved
   - `POST /api/videos/:id/reject` — mark rejected with reason
   - `GET /api/jobs` — queue status
   - `GET /api/stats` — summary statistics

4. Create simple HTML interface `src/qc/public/`:
   - Dashboard with stats
   - Video grid with thumbnails
   - Video detail modal with player
   - Approve/Reject buttons

5. Add video file serving:
   - Route to serve MP4 files from output directory
   - Route to serve thumbnails

6. Test: Generate some videos, open QC interface, review and approve

### Deliverable

Visual interface to review your generated content before uploading.

---

## Phase 12: Polish & Optimization

**Goal:** Smooth out rough edges, optimize for daily use.

**Timeline:** 1-2 days

### Steps

1. Improve prompts based on output quality:
   - Refine script generation prompt
   - Refine metadata generation prompt
   - Add more niche-specific guidelines

2. Add logging throughout:
   - Structured logging with timestamps
   - Log levels (debug, info, warn, error)
   - Log rotation

3. Create startup scripts:
   - `npm run worker` — start worker process
   - `npm run qc` — start QC server
   - PM2 config for production

4. Add CLI improvements:
   - `logs --costs` — cost report for time period
   - `logs --job-id X` — detailed log for specific job
   - Better error messages and formatting

5. Document everything:
   - README with setup instructions
   - Channel configuration guide
   - Troubleshooting common issues

6. Optimize FFmpeg commands for quality/speed balance

### Deliverable

Production-ready system you can run daily with confidence.

---

## Phase Summary

| Phase | Name                      | Timeline   | Key Deliverable                         |
|-------|---------------------------|------------|-----------------------------------------|
| 1     | Project Setup & Database  | 1 day      | Project structure, PostgreSQL running   |
| 2     | Channel Configuration     | 0.5 days   | CLI channel management                  |
| 3     | Job Queue System          | 0.5 days   | Queue and status commands               |
| 4     | Script Generation         | 1 day      | Worker generates scripts via GPT-4      |
| 5     | Voice Synthesis           | 1 day      | Scripts become audio via ElevenLabs     |
| 6     | Footage Selection         | 1-2 days   | Stock footage downloaded and trimmed    |
| 7     | Caption Generation        | 0.5-1 day  | Styled captions from Whisper            |
| 8     | Video Assembly            | 1-2 days   | Complete MP4 output via FFmpeg          |
| 9     | Metadata Generation       | 0.5 days   | Titles, descriptions, tags              |
| 10    | Batch & Robustness        | 1 day      | Reliable multi-job processing           |
| 11    | QC Interface              | 1-2 days   | Web UI for review                       |
| 12    | Polish & Optimization     | 1-2 days   | Production-ready system                 |
|       | **Total**                 | **10-15 days** |                                      |

---

## First Video Milestone

After completing Phases 1-8, you will have generated your first complete video. This is approximately 6-9 days of focused work. Phases 9-12 add polish but aren't required to start producing content.

**Recommended approach:** Push through Phases 1-8 as fast as possible. Generate your first batch of videos. Learn what needs improvement. Then circle back to Phases 9-12 based on what you actually need.

---

## Post-MVP Roadmap

After the core system is running:

1. **Auto-posting** — Direct upload to YouTube, TikTok, Instagram APIs
2. **Topic research** — AI identifies trending topics per niche
3. **Performance tracking** — Pull analytics, correlate with content
4. **A/B testing** — Multiple thumbnails/titles per video
5. **Scheduling** — Queue posts for optimal times
6. **Additional formats** — Horizontal videos, longer content
