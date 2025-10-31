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
RUN pnpm run build --filter=server

#---------------------------------------------------------------------
# Stage 2: PRODUCTION
# This stage uses a minimal Bun image for the final runtime environment.
#---------------------------------------------------------------------
FROM oven/bun:1.3-alpine AS production

ENV NODE_ENV=production

RUN apk add nodejs npm

# Set the final working directory to the server app itself
WORKDIR /app/apps/server

# 1. Copy the necessary package files and built code
# Copying only the required files to the final working directory
COPY --from=builder /app/package.json /app/pnpm-lock.yaml /app/pnpm-workspace.yaml ./
COPY --from=builder /app/apps/server/package.json .
COPY --from=builder /app/apps/server/dist ./dist

# 2. Copy the entire *built* packages tree from the monorepo root
# This includes the built `dist` folders of shared packages required by the server.
COPY --from=builder /app/packages ./packages

# 3. Install *only* production dependencies for the server package.
# This ensures a local node_modules is created for the server's runtime.
RUN npm install -g pnpm
RUN pnpm install --production --frozen-lockfile

# 4. Create a non-root user and set permissions (best practice)
RUN addgroup -S appuser && adduser -S appuser -G appuser
USER appuser

# 5. Expose the default Hono port
EXPOSE 3000

# 6. Define the command to run your server
# This relies on the local node_modules created by the production install step above.
CMD ["pnpm", "start"]
