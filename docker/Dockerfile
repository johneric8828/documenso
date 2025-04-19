###########################
#     BASE CONTAINER      #
###########################
FROM node:22-alpine3.20 AS base

RUN apk add --no-cache openssl


###########################
#    BUILDER CONTAINER    #
###########################
FROM base AS builder

RUN apk add --no-cache libc6-compat jq
WORKDIR /app

COPY . .

RUN npm install -g "turbo@^1.9.3"

# Include both remix (web) and worker packages
RUN turbo prune --scope=@documenso/remix --scope=@documenso/worker --docker


###########################
#   INSTALLER CONTAINER   #
###########################
FROM base AS installer

RUN apk add --no-cache libc6-compat jq make cmake g++ openssl bash

WORKDIR /app

ENV HUSKY=0
ENV DOCKER_OUTPUT=1
ENV NEXT_TELEMETRY_DISABLED=1

ARG NEXT_PRIVATE_ENCRYPTION_KEY="CAFEBABE"
ENV NEXT_PRIVATE_ENCRYPTION_KEY="$NEXT_PRIVATE_ENCRYPTION_KEY"

ARG NEXT_PRIVATE_ENCRYPTION_SECONDARY_KEY="DEADBEEF"
ENV NEXT_PRIVATE_ENCRYPTION_SECONDARY_KEY="$NEXT_PRIVATE_ENCRYPTION_SECONDARY_KEY"

COPY .gitignore .gitignore
COPY --from=builder /app/out/json/ .
COPY --from=builder /app/out/package-lock.json ./package-lock.json
COPY --from=builder /app/lingui.config.ts ./lingui.config.ts

RUN npm ci

COPY --from=builder /app/out/full/ .
COPY turbo.json turbo.json

RUN npm install -g "turbo@^1.9.3"

# Build both remix and worker
RUN turbo run build --filter=@documenso/remix... --filter=@documenso/worker...


###########################
#     RUNNER CONTAINER    #
###########################
FROM base AS runner

ENV HUSKY=0
ENV DOCKER_OUTPUT=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nodejs
USER nodejs

WORKDIR /app

COPY --from=builder --chown=nodejs:nodejs /app/out/json/ .
COPY --from=builder --chown=nodejs:nodejs /app/out/full/packages/tailwind-config ./packages/tailwind-config

RUN npm ci --only=production

# Web app build output
COPY --from=installer --chown=nodejs:nodejs /app/apps/remix/build ./apps/remix/build
COPY --from=installer --chown=nodejs:nodejs /app/apps/remix/public ./apps/remix/public

# Worker package (source or build depending on execution)
COPY --from=installer --chown=nodejs:nodejs /app/apps/worker ./apps/worker

# Prisma
COPY --from=installer --chown=nodejs:nodejs /app/packages/prisma/schema.prisma ./packages/prisma/schema.prisma
COPY --from=installer --chown=nodejs:nodejs /app/packages/prisma/migrations ./packages/prisma/migrations

RUN npx prisma generate --schema ./packages/prisma/schema.prisma

# Web startup script
COPY --chown=nodejs:nodejs ./docker/start.sh /app/apps/remix/start.sh

WORKDIR /app/apps/remix
CMD ["sh", "start.sh"]
