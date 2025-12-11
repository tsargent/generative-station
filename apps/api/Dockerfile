# 1. BASE
FROM node:18-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
ENV NPM_CONFIG_INJECT_WORKSPACE_PACKAGES=true
RUN corepack enable

# 2. BUILDER
FROM base AS builder
WORKDIR /app
COPY . .
# Filter for the API package
RUN pnpm --filter @generative-station/api install && \
	pnpm --filter @generative-station/api build && \
	pnpm --filter @generative-station/api deploy --prod /prod/api

# 3. RUNNER
FROM base AS runner
WORKDIR /app
COPY --from=builder /prod/api ./

CMD ["node", "dist/index.js"]
# Or for dev: CMD ["pnpm", "run", "dev"]