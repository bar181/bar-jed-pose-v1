# Docker Containerization - Phase 5 Deployment

## 🐳 Docker Strategy Overview

### Multi-Stage Build Architecture
```dockerfile
# Dockerfile - Production-ready multi-stage build
FROM node:18-alpine AS base
WORKDIR /app

# Install dependencies only when needed
FROM base AS deps
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine
RUN apk add --no-cache libc6-compat
COPY package.json package-lock.json* ./
RUN npm ci --only=production && npm cache clean --force

# Rebuild the source code only when needed
FROM base AS builder
COPY . .
COPY --from=deps /app/node_modules ./node_modules
RUN npm run build

# Production image, copy all the files and run next
FROM nginx:alpine AS runtime

# Install runtime dependencies
RUN apk add --no-cache \
    curl \
    jq \
    && rm -rf /var/cache/apk/*

# Copy built application
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf.template /etc/nginx/templates/default.conf.template
COPY docker-entrypoint.sh /docker-entrypoint.sh

# Set permissions
RUN chmod +x /docker-entrypoint.sh

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:80/health || exit 1

# Expose port
EXPOSE 80

# Environment variables
ENV BASE_PATH=/
ENV NODE_ENV=production

# Start application
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```

### Docker Compose Configuration
```yaml
# docker-compose.yml - Development environment
version: '3.8'

services:
  pose-detection-app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    ports:
      - "8080:80"
    environment:
      - BASE_PATH=/pose
      - NODE_ENV=development
      - VITE_FEATURE_GAIT_ANALYSIS=true
      - VITE_FEATURE_PERFORMANCE=true
    volumes:
      - ./nginx-dev.conf:/etc/nginx/nginx.conf:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.pose-detection.rule=PathPrefix(`/pose`)"
      - "traefik.http.services.pose-detection.loadbalancer.server.port=80"

  # Reverse proxy for development
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped

# docker-compose.prod.yml - Production environment
version: '3.8'

services:
  pose-detection-app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    environment:
      - BASE_PATH=/pose
      - NODE_ENV=production
      - VITE_MONITORING_ENABLED=true
      - VITE_ANALYTICS_ENABLED=true
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
    secrets:
      - monitoring_key
      - analytics_token
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    networks:
      - pose-detection-network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

configs:
  nginx_config:
    file: ./nginx.prod.conf

secrets:
  monitoring_key:
    external: true
  analytics_token:
    external: true

networks:
  pose-detection-network:
    driver: overlay
    attachable: true
```

### Nginx Configuration Templates
```nginx
# nginx.conf.template - Dynamic configuration
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    # Performance optimizations
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 16M;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        application/atom+xml
        application/geo+json
        application/javascript
        application/x-javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rdf+xml
        application/rss+xml
        application/xhtml+xml
        application/xml
        font/eot
        font/otf
        font/ttf
        image/svg+xml
        text/css
        text/javascript
        text/plain
        text/xml;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; media-src 'self' blob:; worker-src 'self' blob:; connect-src 'self';" always;

    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;

        # Dynamic base path configuration
        set $base_path "${BASE_PATH}";
        
        # Remove trailing slash from base path
        if ($base_path ~ ^(.+)/$) {
            set $base_path $1;
        }

        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # Readiness check endpoint
        location /ready {
            access_log off;
            try_files /index.html =503;
            add_header Content-Type text/plain;
        }

        # Main application
        location $base_path {
            alias /usr/share/nginx/html;
            try_files $uri $uri/ $base_path/index.html;
            
            # Cache static assets
            location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
                expires 1y;
                add_header Cache-Control "public, immutable";
                add_header Vary "Accept-Encoding";
                access_log off;
            }

            # No cache for HTML files
            location ~* \.html$ {
                expires -1;
                add_header Cache-Control "no-cache, no-store, must-revalidate";
                add_header Pragma "no-cache";
            }
        }

        # Root redirect to base path
        location = / {
            return 301 $base_path/;
        }

        # Security: Deny access to sensitive files
        location ~ /\. {
            deny all;
            access_log off;
            log_not_found off;
        }

        location ~ /(package\.json|Dockerfile|docker-compose|\.env) {
            deny all;
            access_log off;
            log_not_found off;
        }
    }
}
```

### Docker Entrypoint Script
```bash
#!/bin/sh
# docker-entrypoint.sh - Runtime configuration

set -e

# Function to log messages
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Function to substitute environment variables in templates
substitute_env_vars() {
    local template_file="$1"
    local output_file="$2"
    
    envsubst < "$template_file" > "$output_file"
    log "Generated $output_file from $template_file"
}

# Function to validate environment variables
validate_env() {
    log "Validating environment variables..."
    
    # Set defaults
    export BASE_PATH="${BASE_PATH:-/}"
    export NODE_ENV="${NODE_ENV:-production}"
    
    # Validate BASE_PATH format
    if [[ ! "$BASE_PATH" =~ ^/ ]]; then
        export BASE_PATH="/$BASE_PATH"
    fi
    
    log "BASE_PATH: $BASE_PATH"
    log "NODE_ENV: $NODE_ENV"
}

# Function to setup nginx configuration
setup_nginx() {
    log "Setting up Nginx configuration..."
    
    # Generate nginx config from template
    if [ -f "/etc/nginx/templates/default.conf.template" ]; then
        substitute_env_vars "/etc/nginx/templates/default.conf.template" "/etc/nginx/conf.d/default.conf"
    fi
    
    # Test nginx configuration
    if ! nginx -t; then
        log "ERROR: Nginx configuration test failed"
        exit 1
    fi
    
    log "Nginx configuration validated successfully"
}

# Function to setup application assets
setup_assets() {
    log "Setting up application assets..."
    
    # Update index.html with runtime configuration
    if [ -f "/usr/share/nginx/html/index.html" ]; then
        # Replace base href with actual BASE_PATH
        sed -i "s|<base href=\"/\">|<base href=\"$BASE_PATH/\">|g" /usr/share/nginx/html/index.html
        
        # Inject runtime configuration
        cat > /usr/share/nginx/html/config.js << EOF
window.__APP_CONFIG__ = {
  basePath: '$BASE_PATH',
  environment: '$NODE_ENV',
  features: {
    gaitAnalysis: ${VITE_FEATURE_GAIT_ANALYSIS:-true},
    performance: ${VITE_FEATURE_PERFORMANCE:-true},
    export: ${VITE_FEATURE_EXPORT:-true}
  },
  monitoring: {
    enabled: ${VITE_MONITORING_ENABLED:-false},
    sampleRate: ${VITE_MONITORING_SAMPLE_RATE:-0.1}
  }
};
EOF
        log "Generated runtime configuration"
    fi
}

# Function to run health checks
health_check() {
    log "Running initial health checks..."
    
    # Check if required files exist
    if [ ! -f "/usr/share/nginx/html/index.html" ]; then
        log "ERROR: Application files not found"
        exit 1
    fi
    
    # Check if nginx can start
    if ! nginx -t > /dev/null 2>&1; then
        log "ERROR: Nginx configuration invalid"
        exit 1
    fi
    
    log "Health checks passed"
}

# Function to setup monitoring
setup_monitoring() {
    if [ "$VITE_MONITORING_ENABLED" = "true" ]; then
        log "Setting up monitoring..."
        
        # Create monitoring configuration
        cat > /usr/share/nginx/html/monitoring.js << EOF
window.__MONITORING_CONFIG__ = {
  enabled: true,
  endpoint: '${VITE_MONITORING_ENDPOINT:-}',
  apiKey: '${MONITORING_KEY:-}',
  sampleRate: ${VITE_MONITORING_SAMPLE_RATE:-0.1}
};
EOF
        log "Monitoring configuration generated"
    fi
}

# Function to handle graceful shutdown
cleanup() {
    log "Received shutdown signal, cleaning up..."
    
    # Stop nginx gracefully
    if [ -f "/var/run/nginx.pid" ]; then
        kill -QUIT $(cat /var/run/nginx.pid)
    fi
    
    log "Cleanup completed"
    exit 0
}

# Setup signal handlers
trap cleanup SIGTERM SIGINT

# Main execution
main() {
    log "Starting Pose Detection Application container..."
    log "Version: ${APP_VERSION:-unknown}"
    log "Build: ${BUILD_NUMBER:-unknown}"
    
    validate_env
    setup_nginx
    setup_assets
    setup_monitoring
    health_check
    
    log "Container initialization completed"
    log "Starting services..."
    
    # Start the main process
    exec "$@"
}

# Run main function with all arguments
main "$@"
```

## 🔧 Container Optimization

### Multi-Architecture Build
```yaml
# .github/workflows/docker-build.yml
name: Build Multi-Architecture Docker Images

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
            
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_NUMBER=${{ github.run_number }}
            BUILD_SHA=${{ github.sha }}
            BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
```

### Resource Optimization
```dockerfile
# Dockerfile.optimized - Size and performance optimized
FROM node:18-alpine AS base

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache dumb-init

# Create app user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodeapp -u 1001

FROM base AS deps
WORKDIR /app
COPY package*.json ./

# Install dependencies with exact versions
RUN npm ci --only=production --frozen-lockfile && \
    npm cache clean --force && \
    rm -rf ~/.npm

FROM base AS builder
WORKDIR /app
COPY . .
COPY --from=deps /app/node_modules ./node_modules

# Build with optimizations
RUN npm run build && \
    npm prune --production

FROM nginx:alpine AS runtime

# Install security updates and curl for health checks
RUN apk update && apk upgrade && \
    apk add --no-cache curl dumb-init && \
    rm -rf /var/cache/apk/*

# Create non-root user
RUN addgroup -g 1001 -S nginx
RUN adduser -S nginxapp -u 1001 -G nginx

# Copy built application
COPY --from=builder --chown=nginxapp:nginx /app/dist /usr/share/nginx/html

# Copy configuration files
COPY --chown=nginxapp:nginx nginx.conf /etc/nginx/nginx.conf
COPY --chown=nginxapp:nginx docker-entrypoint.sh /docker-entrypoint.sh

# Set permissions
RUN chmod +x /docker-entrypoint.sh && \
    chown -R nginxapp:nginx /var/cache/nginx && \
    chown -R nginxapp:nginx /var/log/nginx && \
    chown -R nginxapp:nginx /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && \
    chown -R nginxapp:nginx /var/run/nginx.pid

# Switch to non-root user
USER nginxapp

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Expose port
EXPOSE 8080

# Use dumb-init for proper signal handling
ENTRYPOINT ["dumb-init", "--"]
CMD ["/docker-entrypoint.sh", "nginx", "-g", "daemon off;"]
```

### Security Hardening
```dockerfile
# Dockerfile.secure - Security-focused build
FROM node:18-alpine AS security-base

# Install security updates and scanning tools
RUN apk update && apk upgrade && \
    apk add --no-cache \
        ca-certificates \
        curl \
        dumb-init \
        su-exec && \
    rm -rf /var/cache/apk/*

# Create non-root user with specific UID/GID
RUN addgroup -g 10001 -S appgroup && \
    adduser -S appuser -u 10001 -G appgroup

# Set security-focused umask
RUN echo "umask 027" >> /etc/profile

FROM security-base AS deps
WORKDIR /app

# Copy dependency files with proper ownership
COPY --chown=appuser:appgroup package*.json ./

# Switch to non-root user for dependency installation
USER appuser
RUN npm ci --only=production --no-audit --no-fund && \
    npm cache clean --force

FROM security-base AS builder
WORKDIR /app

# Copy source files
COPY --chown=appuser:appgroup . .
COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules

# Build as non-root user
USER appuser
RUN npm run build

FROM nginx:alpine AS runtime

# Security updates
RUN apk update && apk upgrade && \
    apk add --no-cache curl dumb-init && \
    rm -rf /var/cache/apk/*

# Remove unnecessary packages and files
RUN apk del --purge wget && \
    rm -rf /var/cache/apk/* \
           /tmp/* \
           /var/tmp/* \
           /usr/share/man \
           /usr/share/doc

# Create non-root user
RUN addgroup -g 10001 -S nginxgroup && \
    adduser -S nginxuser -u 10001 -G nginxgroup

# Copy application files with proper ownership
COPY --from=builder --chown=nginxuser:nginxgroup /app/dist /usr/share/nginx/html

# Copy and secure configuration files
COPY --chown=root:root nginx.secure.conf /etc/nginx/nginx.conf
COPY --chown=root:root docker-entrypoint.secure.sh /docker-entrypoint.sh

# Set strict permissions
RUN chmod 755 /docker-entrypoint.sh && \
    chmod 644 /etc/nginx/nginx.conf && \
    chown -R nginxuser:nginxgroup /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html

# Remove shell access for security
RUN rm -rf /bin/sh /bin/bash

# Set read-only root filesystem
VOLUME ["/tmp", "/var/cache/nginx", "/var/run"]

# Switch to non-root user
USER nginxuser

# Expose non-privileged port
EXPOSE 8080

# Security labels
LABEL security.scan="enabled" \
      security.policy="restricted" \
      security.user="nginxuser"

# Use dumb-init and run as non-root
ENTRYPOINT ["dumb-init", "--"]
CMD ["/docker-entrypoint.sh"]
```

This comprehensive Docker containerization strategy ensures secure, optimized, and production-ready deployment of the pose detection application with proper configuration management and monitoring capabilities.