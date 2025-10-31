#---------------------------------------------------------------------
# Stage 1: BUILDER
# Using a Node image for easy installation of PNPM
#---------------------------------------------------------------------
FROM node:20-alpine AS builder

WORKDIR /app

# Install PNPM globally
RUN npm install -g pnpm

# 1. Copy all package.json files and the PNPM lockfile
# All paths are relative to the build context (monorepo root).
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY apps/server/package.json ./apps/server/
COPY apps/web/package.json ./apps/web/
COPY packages/api/package.json ./packages/api/
COPY packages/auth/package.json ./packages/auth/
COPY packages/db/package.json ./packages/db/

# 2. Install all dependencies using PNPM
# --frozen-lockfile ensures deterministic builds
RUN pnpm install --frozen-lockfile --production=false

# 3. Copy all source code
COPY . .

# 4. Build all workspace dependencies AND the server using PNPM and Turbo
# This uses your existing Turbo config's 'build' command with filters
RUN pnpm run build --filter="@deploy-test/api..."
RUN pnpm run build --filter="server"

#---------------------------------------------------------------------
# Stage 2: PRODUCTION
# This stage builds the final, lean production image. Since the original
# 'start' script uses 'bun', we return to a bun-based image for the runtime.
#---------------------------------------------------------------------
FROM oven/bun:1.3-alpine AS production

ENV NODE_ENV=production

ENV NODE_PACKAGE_URL https://nodejs.org/dist/v24.11.0/node-v24.11.0-linux-x64.tar.xz
RUN apk add libstdc++
WORKDIR /opt
RUN wget $NODE_PACKAGE_URL
RUN mkdir -p /opt/nodejs
RUN tar -zxvf *.tar.xz --directory /opt/nodejs --strip-components=1
RUN rm *.tar.gz
RUN ln -s /opt/nodejs/bin/node /usr/local/bin/node
RUN ln -s /opt/nodejs/bin/npm /usr/local/bin/npm

RUN npm install -g npm@latest

WORKDIR /app

# Create a non-root user for better security
RUN addgroup -S appuser && adduser -S appuser -G appuser

# 1. Copy only the package.json files and lockfile again
COPY --from=builder /app/package.json /app/pnpm-lock.yaml ./
COPY --from=builder /app/apps/server/package.json ./apps/server/
COPY --from=builder /app/apps/web/package.json ./apps/web/
COPY --from=builder /app/packages/api/package.json ./packages/api/
COPY --from=builder /app/packages/auth/package.json ./packages/auth/
COPY --from=builder /app/packages/db/package.json ./packages/db/

# 2. Install PNPM again in this stage and run production install
RUN npm install -g pnpm
RUN pnpm install --frozen-lockfile --production

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

# 7. Define the command to run your server via pnpm
CMD ["pnpm", "start"]

