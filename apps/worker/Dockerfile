# 1. BASE: Setup Node & PNPM
FROM node:18-bullseye-slim AS base
# We need these for Csound (Worker Only)
RUN apt-get update && apt-get install -y \
    csound \
    ffmpeg \
    python3 \
    make \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Install pnpm globally
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable
ENV NPM_CONFIG_INJECT_WORKSPACE_PACKAGES=true

# 2. BUILDER: Prune the monorepo
FROM base AS builder
WORKDIR /app
COPY . .
# This command isolates only the "worker" app and its specific deps
RUN pnpm --filter @generative-station/worker install && \
    pnpm --filter @generative-station/worker build && \
    pnpm --filter @generative-station/worker deploy --prod /prod/worker

# 3. RUNNER: The final small image
FROM base AS runner
WORKDIR /app

# Copy the isolated production dependencies from the builder
COPY --from=builder /prod/worker ./

# Start
CMD ["node", "dist/index.js"] 
# Note: If you aren't compiling TS to dist yet, use:
# CMD ["pnpm", "run", "dev"] for now.