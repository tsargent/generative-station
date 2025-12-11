# Generative Radio Station — PRD (MVP)

- Version: 2.0 (One-Box Monorepo)
- Owner: Tyler Sargent
- Status: In Development

## Overview

The Generative Radio Station is an automated "Robot DJ" that continuously synthesizes, renders, and broadcasts unique audio. Unlike traditional radio, this system generates discrete audio files using Csound, uploads them to object storage (MinIO/S3), and coordinates playback across clients via WebSockets to create a synchronized, gapless experience.

## Goals

### Primary Goals

- Synthesis Pipeline: Automatically render unique 30–60 second audio segments using Csound templates driven by Node.js logic.
- One-Box Architecture: Run the entire stack (Worker, API, DB, Redis, Storage) on a single VPS using Docker Compose.
- Gapless Delivery: Implement a look-ahead strategy where audio is generated and announced 30 seconds before playback is required.
- Strict Typing: Use a TypeScript monorepo to share data structures (Segment, Event) between the Generator and API.

## System Architecture

### Components

1. Apps (Services)
   - Worker (`apps/worker`): Schedules generation, runs Csound CLI, uploads to S3, writes to DB.
   - API (`apps/api`): Stateless Express server; relays Redis events to WebSockets and serves metadata.
2. Infrastructure (Dockerized)
   - Postgres: Persists segment metadata.
   - Redis: Pub/Sub bus for real-time signaling.
   - MinIO: S3-compatible object storage (self-hosted).
3. Shared Code
   - Types (`packages/types`): Shared TypeScript interfaces.

## Functional Requirements

### Audio Generation (Worker)

- Engine: Must use Csound (CLI) to render audio.
- Templates: Logic must select from a pool of `.csd` files (e.g., `drone.csd`, `noise.csd`).
- Parameterization: Logic must inject randomness into Csound macros (e.g., `--omacro:FREQ=440`) to ensure variety.
- Output: Must render to `.wav`, verify file integrity, and upload to the `radio-segments` bucket.
- Cleanup: Must delete local `.wav` files immediately after upload to save container space.

### Real-Time Events (API)

- The system uses a look-ahead event model.
- Event: `segmentReady`
  - Trigger: Published by Worker via Redis when upload completes.
  - Timing: ~30 seconds before the current track ends.
  - Payload:

```json
{
  "type": "segmentReady",
  "payload": {
    "id": "seg_123",
    "audioUrl": "http://minio:9000/radio-segments/seg_123.wav",
    "startTime": "2023-10-27T12:00:30Z",
    "duration": 45
  }
}
```

### Storage & Persistence

- Object Storage
  - Dev: MinIO (local Docker)
  - Prod: DigitalOcean Spaces (or S3)
  - Bucket Policy: Must be public-read
- Database
  - Table: `Segment` (`id`, `audioUrl`, `startTime`, `endTime`, `params`)

## Technical Stack

| Layer        | Technology       | Reason                                                |
|--------------|------------------|-------------------------------------------------------|
| Monorepo     | pnpm v9          | Efficient disk usage; strict workspace isolation      |
| Language     | TypeScript       | Shared interfaces between API and Worker              |
| Synthesis    | Csound           | Powerful, CLI-driven, lightweight DSP                 |
| Server       | Node.js 18       | Common runtime for both logic and API                 |
| Container    | Docker Compose   | Simplifies one-box deployment                         |
| Base OS      | Debian Slim      | Required for Csound/FFmpeg binaries                   |

## Project Structure

```text
generative-station/
├── .npmrc                    # inject-workspace-packages=true
├── pnpm-workspace.yaml       # Defines 'apps' and 'packages'
├── docker-compose.yml        # Orchestrates the services
├── apps/
│   ├── api/                  # Express + WebSocket
│   │   └── Dockerfile        # Multi-stage build
│   └── worker/               # Csound logic
│       ├── Dockerfile        # Installs Csound + FFmpeg
│       └── templates/        # .csd files
└── packages/
    └── types/                # Shared TS interfaces
```

## Deployment Strategy

- Method: One-box Docker Compose
- Target: Single VPS (e.g., DigitalOcean, Hetzner)
- Build Process
  1. Host runs `pnpm install` (generates lockfile).
  2. `docker compose build` copies lockfile and uses `pnpm --filter ... deploy` to build isolated images.
  3. `docker compose up` starts the radio.

## Risks & Mitigations (v2.0)

- Risk: pnpm v10 / Docker build failures
  - Mitigation: Downgrade to pnpm v9.7.0 and implement `.dockerignore` rules.
- Risk: Csound render time exceeds segment duration
  - Mitigation: Worker timeout logic; fall back to a "Station ID" if render fails.
- Risk: Disk space exhaustion
  - Mitigation: Worker deletes local files post-upload (future: cron job to clear old S3 files).