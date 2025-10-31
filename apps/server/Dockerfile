#---------------------------------------------------------------------
# Stage 1: BUILDER
# This stage installs all dependencies (dev included) and builds the
# entire application, including all workspace packages.
#---------------------------------------------------------------------
FROM oven/bun:1.0 AS builder

WORKDIR /app

# 1. Copy all package.json files first.
# All paths are relative to the build context (monorepo root).
COPY package.json bun.lockb ./
COPY apps/server/package.json ./apps/server/
COPY apps/web/package.json ./apps/web/
COPY packages/api/package.json ./packages/api/
COPY packages/auth/package.json ./packages/auth/
COPY packages/db/package.json ./packages/db/

# 2. Install all dependencies (including devDependencies for building)
RUN bun install --frozen-lockfile --production=false

# 3. Copy all source code
COPY . .

# 4. Build all workspace dependencies AND the server.
# This builds dependencies first, then the server package itself.
RUN bun run --filter="@deploy-test/api" build
RUN bun run --filter="@deploy-test/auth" build
RUN bun run --filter="@deploy-test/db" build
RUN bun run --filter="server" build

#---------------------------------------------------------------------
# Stage 2: PRODUCTION
# This stage builds the final, lean production image. It copies
# only the necessary built artifacts and production dependencies.
#---------------------------------------------------------------------
FROM oven/bun:1.0-alpine AS production

ENV NODE_ENV=production

WORKDIR /app

# Create a non-root user for better security
RUN addgroup -S appuser && adduser -S appuser -G appuser

# 1. Copy only the package.json files again
COPY --from=builder /app/package.json /app/bun.lockb ./
COPY --from=builder /app/apps/server/package.json ./apps/server/
COPY --from=builder /app/apps/web/package.json ./apps/web/
COPY --from=builder /app/packages/api/package.json ./packages/api/
COPY --from=builder /app/packages/auth/package.json ./packages/auth/
COPY --from=builder /app/packages/db/package.json ./packages/db/

# 2. Install ONLY production dependencies
RUN bun install --frozen-lockfile --production

# 3. Copy the built code (the `dist` folders) from the builder stage
COPY --from=builder /app/apps/server/dist ./apps/server/dist
COPY --from=builder /app/packages/api/dist ./packages/api/dist
COPY --from=builder /app/packages/auth/dist ./packages/auth/dist
COPY --from=builder /app/packages/db/dist ./packages/db/dist

# 4. Set the non-root user
USER appuser

# 5. Set the working directory to the server package
WORKDIR /app/apps/server

# 6. Expose the default Hono port
EXPOSE 3000

# 7. Define the command to run your server
# This executes "start": "bun run dist/index.js" from `apps/server/package.json`
CMD ["bun", "run", "start"]

