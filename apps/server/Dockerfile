#---------------------------------------------------------------------
# Stage 1: BUILDER
# Using a Node image for easy installation of PNPM
#---------------------------------------------------------------------
FROM node:20-alpine AS builder

WORKDIR /app

# Install PNPM globally
RUN npm install -g pnpm

# 1. Copy package files
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY apps/server/package.json ./apps/server/
COPY apps/web/package.json ./apps/web/
COPY packages/api/package.json ./packages/api/
COPY packages/auth/package.json ./packages/auth/
COPY packages/db/package.json ./packages/db/

# 2. Install all dependencies
RUN pnpm install --frozen-lockfile --production=false

# 3. Copy source code
COPY . .

# 4. Build all necessary packages
RUN pnpm run build --filter=@deploy-test/api...
RUN pnpm run build --filter=@deploy-test/db...
RUN pnpm run build --filter=@deploy-test/auth...
RUN pnpm run build --filter=server

#---------------------------------------------------------------------
# Stage 2: PRODUCTION
# This stage uses a minimal Bun image for the final runtime environment.
#---------------------------------------------------------------------
FROM oven/bun:1.3-debian AS production

ENV NODE_ENV=production

WORKDIR /app

# Install dependencies, download key, add repo, install Node.js, and clean up
RUN set -ex; \
    # 1. Update and install core utilities
    apt-get update; \
    apt-get install -y --no-install-recommends \
        curl \
        gnupg \
        dirmngr; \
    \
    # 2. Use the official NodeSource script to add the LTS repository
    curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - ; \
    \
    # 3. Install Node.js (this step already ran inside the 'bash -' script,
    # but we run it again in case it was skipped or the source lists weren't re-updated)
    apt-get install -y nodejs npm; \
    \
    # 4. Clean up to minimize image size
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*


# Copy root files needed for pnpm to resolve dependencies
COPY --from=builder /app/package.json .
COPY --from=builder /app/pnpm-lock.yaml .
COPY --from=builder /app/pnpm-workspace.yaml .

# Copy only the package.json files necessary for the production install filter
COPY --from=builder /app/apps/server/package.json ./apps/server/
COPY --from=builder /app/packages/api/package.json ./packages/api/
COPY --from=builder /app/packages/auth/package.json ./packages/auth/
COPY --from=builder /app/packages/db/package.json ./packages/db/

# 2. Install PNPM globally in this stage
RUN npm install -g pnpm

# 3. Install *only* production dependencies for the server package,
# running from the root where the lockfile is. This will create the necessary
# node_modules structure for the server app to run.
RUN pnpm install --production --frozen-lockfile

# 4. Copy the built code and run artifacts
# The server's built code
COPY --from=builder /app/apps/server/dist ./apps/server/dist
# The entire built packages tree (required for workspace imports)
COPY --from=builder /app/packages ./packages

# 5. Set the final working directory to the server app itself
WORKDIR /app/apps/server

# 6. Create a non-root user and set permissions (best practice)
# RUN addgroup -S appuser && adduser -S appuser -G appuser
RUN addgroup --system appuser && adduser --system --no-create-home --ingroup appuser appuser
USER appuser

# 7. Expose the default Hono port
EXPOSE 3000

# 8. Define the command to run your server
# This relies on the dependency structure established in step 3.
CMD ["pnpm", "start"]

