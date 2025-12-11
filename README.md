# Generative Station

A one-box monorepo that generates short audio segments via Csound, stores them in S3-compatible object storage, and coordinates gapless playback via an API + WebSocket layer.

## Repo Map

- `apps/api`: Express + WebSocket server (ESM, TypeScript)
- `apps/worker`: Csound-driven generator (ESM, TypeScript)
- `packages/types`: Shared TypeScript interfaces
- `docker-compose.yml`: Orchestrates Postgres, Redis, MinIO, API, Worker
- `docs/PRD.md`: Product Requirements Document (MVP)

## Prerequisites

- Node.js 18+
- pnpm (managed by Corepack)
- Docker Desktop or compatible runtime

## Quick Start (Docker)

1. Create a `.env` at repo root (used by Compose):

```env
# Database
DB_USER=radio
DB_PASS=radio_pass

# S3-compatible storage (MinIO defaults)
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
```

2. Build and run:

```zsh
docker compose build
docker compose up
```

- API: `http://localhost:3000`
- MinIO Console: `http://localhost:9001`

## Local Development (pnpm)

Install deps and run services with hot reload using `tsx`:

```zsh
pnpm install

# API
pnpm --filter @generative-station/api dev

# Worker
pnpm --filter @generative-station/worker dev
```

Build artifacts:

```zsh
pnpm --filter @generative-station/api build && pnpm --filter @generative-station/api start
pnpm --filter @generative-station/worker build && pnpm --filter @generative-station/worker start
```

## Notes

- ESM is enabled for both apps (`type: module`, TS `module: NodeNext`).
- Dockerfiles compile in the builder stage and deploy pruned production output.
- See the PRD for architecture and requirements: `docs/PRD.md`.