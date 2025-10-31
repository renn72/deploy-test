#---------------------------------------------------------------------
# Stage 1: BUILDER
# This stage installs all dependencies (dev included) and builds the
# React/Vite application.
#---------------------------------------------------------------------
FROM oven/bun:1.0 AS builder

WORKDIR /app

# 1. Copy all package.json files first (monorepo setup)
# This optimizes Docker's cache layer.
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
# This is done after `bun install` to leverage caching
COPY . .

# 4. Build the 'web' package
# This runs the "build": "vite build" script from `apps/web/package.json`
# The output will be in `apps/web/dist`
RUN bun run --filter="web" build

#---------------------------------------------------------------------
# Stage 2: PRODUCTION
# This stage builds the final, lean production image using Nginx
# to serve the static files created in the builder stage.
#---------------------------------------------------------------------
FROM nginx:alpine AS production

# 1. Copy the custom Nginx configuration
# This path is relative to the build context (monorepo root)
COPY apps/web/nginx.conf /etc/nginx/conf.d/default.conf

# 2. Copy the built static assets from the 'builder' stage
# The `apps/web/dist` directory contains the output from `vite build`
COPY --from=builder /app/apps/web/dist /usr/share/nginx/html

# 3. Expose Nginx's default port
# Your docker-compose.yml will map this (e.g., "80:80" or "3001:80")
EXPOSE 80

# 4. Start Nginx in the foreground
# This is the standard command for Nginx in Docker.
CMD ["nginx", "-g", "daemon off;"]
