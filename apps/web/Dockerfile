FROM node:22-slim AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable && corepack prepare pnpm@9.15.4 --activate

# ── Install dependencies ─────────────────────────────────────────────────────
FROM base AS deps

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 make g++ \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy workspace structure files first (layer caching)
COPY pnpm-workspace.yaml pnpm-lock.yaml package.json turbo.json .npmrc ./

# Copy all package.json files to establish workspace structure
COPY apps/web/package.json apps/web/package.json
COPY packages/common/package.json packages/common/package.json
COPY packages/database/package.json packages/database/package.json
COPY packages/email/package.json packages/email/package.json
COPY packages/storage/package.json packages/storage/package.json

# Create public dir so the postinstall script (cp pdfjs worker) doesn't fail
RUN mkdir -p apps/web/public

RUN pnpm install --frozen-lockfile

# ── Build ─────────────────────────────────────────────────────────────────────
FROM deps AS builder
WORKDIR /app

COPY packages/ packages/
COPY apps/web/ apps/web/

# Re-run postinstall to copy pdfjs worker into public/
RUN pnpm --filter @locker/web run postinstall

# Build with Turborepo (builds dependent packages first, then next build)
RUN pnpm turbo build --filter=@locker/web

# ── Production runner ─────────────────────────────────────────────────────────
FROM node:22-slim AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy standalone output (mirrors monorepo structure)
COPY --from=builder /app/apps/web/.next/standalone ./
# Copy static assets and public files (not included in standalone)
COPY --from=builder /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=builder /app/apps/web/public ./apps/web/public

RUN mkdir -p /app/local-blobs && chown nextjs:nodejs /app/local-blobs

USER nextjs

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "apps/web/server.js"]
