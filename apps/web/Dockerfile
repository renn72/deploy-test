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
COPY package.json pnpm-lock.yaml ./
COPY apps/server/package.json ./apps/server/
COPY apps/web/package.json ./apps/web/
COPY packages/api/package.json ./packages/api/
COPY packages/auth/package.json ./packages/auth/
COPY packages/db/package.json ./packages/db/

ARG VITE_SERVER_URL
ENV VITE_SERVER_URL=${VITE_SERVER_URL}

# 2. Install all dependencies using PNPM
RUN pnpm install --frozen-lockfile --production=false
RUN pnpm install --frozen-lockfile --dev

# 3. Copy all source code
COPY . .

# 4. Build the 'web' package using PNPM
# This runs the "build": "vite build" script from `apps/web/package.json`
# The output will be in `apps/web/dist`
RUN pnpm run build --filter="web"

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
EXPOSE 80

# 4. Start Nginx in the foreground
CMD ["nginx", "-g", "daemon off;"]
