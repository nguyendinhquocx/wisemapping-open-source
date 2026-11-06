################################################################################
# Multi-stage Dockerfile for WiseMapping (Frontend + Backend)
# This Dockerfile builds both the backend API and frontend UI in a single image
################################################################################

################################################################################
# Stage 1: Build Backend (Spring Boot API)
################################################################################
FROM eclipse-temurin:26-jdk-alpine@sha256:7cc9b8649a735f80bb5216e135306c4b9ab428db2893d770c4516f469fa08623 AS backend-builder
LABEL maintainer="Paulo Gustavo Veiga <pveiga@wisemapping.com>"

WORKDIR /build

# Install Maven
RUN apk add --no-cache maven

# Copy backend source code
COPY wise-api/pom.xml ./wise-api/
COPY wise-api/src ./wise-api/src/

# Build the backend JAR
WORKDIR /build/wise-api
RUN mvn clean package -DskipTests \
    -Dgit.branch=unknown \
    -Dgit.commit.id.abbrev=unknown \
    -Dgit.commit.id.full=unknown \
    -Dgit.commit.time=unknown

################################################################################
# Stage 2: Build Frontend (React/TypeScript)
################################################################################
FROM node:24-alpine@sha256:8a32c8cf649a49915a66e81e7a609223c708d0adabf288ee3f4077d765542f85 AS frontend-builder

WORKDIR /build

# Pin the frontend source used for the 6.0.11 release. The tag is only a fetch
# locator; the full commit check below is the actual authority.
ARG FRONTEND_REF="6.0.11"
ARG FRONTEND_COMMIT="03e59b2cbfac56e4e6ffbf8874d8aff738fc44b0"

# Install git
RUN apk add --no-cache git

# Retain an explicit cache-bust for controlled rebuilds; source remains pinned.
ARG CACHEBUST=1

RUN git init /build/wisemapping-frontend \
    && git -C /build/wisemapping-frontend remote add origin https://github.com/wisemapping/wisemapping-frontend.git \
    && git -C /build/wisemapping-frontend fetch --depth 1 origin "${FRONTEND_REF}" \
    && git -C /build/wisemapping-frontend checkout --detach FETCH_HEAD \
    && test "$(git -C /build/wisemapping-frontend rev-parse HEAD)" = "${FRONTEND_COMMIT}"

COPY distribution/app/frontend-patches/wisemapping-frontend-03e59b2cbfac56e4e6ffbf8874d8aff738fc44b0-vi.patch /tmp/wisemapping-frontend-vi.patch
RUN git -C /build/wisemapping-frontend apply --unidiff-zero --check /tmp/wisemapping-frontend-vi.patch \
    && git -C /build/wisemapping-frontend apply --unidiff-zero /tmp/wisemapping-frontend-vi.patch

# Patch the production config to use relative URL
WORKDIR /build/wisemapping-frontend/packages/webapp
RUN sed -i 's|"url": "http://localhost:3000/api/restful/app/config"|"url": "/api/restful/app/config"|' config.prod.json

# Build the frontend (will use patched config.prod.json)
WORKDIR /build/wisemapping-frontend
ENV COREPACK_ENABLE_STRICT=0
RUN corepack enable && corepack prepare yarn@4.13.0 --activate
RUN yarn install
ENV NODE_OPTIONS="--max-old-space-size=4096"
RUN APP_CONFIG_TYPE=file:prod yarn build --concurrency 1

# Ensure proper base tag for nginx sub_filter replacement:
# 1. Remove any base tag webpack created (with empty, relative, or absolute href)
# 2. Add our placeholder <base> tag for nginx to replace with absolute URL at runtime
RUN sed -i 's|<base[^>]*>||g' /build/wisemapping-frontend/packages/webapp/dist/index.html && \
    sed -i 's|<head>|<head>\n  <base>|' /build/wisemapping-frontend/packages/webapp/dist/index.html  

################################################################################
# Stage 3: Final Runtime Image (Backend + Frontend + Nginx)
################################################################################
FROM eclipse-temurin:26-jre-alpine@sha256:158186ab168dc35e5a8e799f6195eef61d57a86a73348fc3a1188317fc40f01c
LABEL maintainer="Paulo Gustavo Veiga <pveiga@wisemapping.com>"
LABEL org.opencontainers.image.source="https://github.com/nguyendinhquocx/wisemapping-open-source"
LABEL org.opencontainers.image.licenses="WPL-1.0"

# Install Nginx, tini (PID 1 init) and su-exec (privilege drop).
# Supervisor is intentionally NOT installed: its Python dependency on
# Alpine breaks at runtime due to a libexpat/python skew (see issue #70).
RUN apk add --no-cache nginx tini su-exec

# Create non-root user for security
RUN addgroup -g 1001 -S wisemapping && \
    adduser -S -u 1001 -G wisemapping wisemapping

# Create necessary directories
RUN mkdir -p /app \
    /var/log/nginx \
    /run/nginx \
    /usr/share/nginx/html \
    /var/lib/nginx/tmp/client_body \
    /var/lib/nginx/tmp/proxy && \
    chown -R wisemapping:wisemapping /app \
    /var/log/nginx \
    /run/nginx \
    /usr/share/nginx/html && \
    chown -R 100:101 /var/lib/nginx

# Copy backend JAR
COPY --from=backend-builder --chown=wisemapping:wisemapping /build/wise-api/target/wisemapping-api.jar /app/wisemapping-api.jar

# Create config directory for optional app.yml
RUN mkdir -p /app/config && chown wisemapping:wisemapping /app/config

# Copy frontend build
COPY --from=frontend-builder --chown=wisemapping:wisemapping /build/wisemapping-frontend/packages/webapp/dist /usr/share/nginx/html/

# Copy Nginx configuration
COPY distribution/app/nginx.conf /etc/nginx/http.d/default.conf

# Copy startup script (launches nginx + Spring Boot under tini)
COPY distribution/app/start.sh /app/start.sh
RUN chmod +x /app/start.sh

# Expose port 80 only (Nginx handles routing to backend)
EXPOSE 80

WORKDIR /app

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["/app/start.sh"]
