# ==============================================================================
# FRONTEND  -  generic Dockerfile for Angular / React / Vue (any npm-built single-page app) on nginx
#
# HOW TO USE:  copy this file into your frontend folder as `Dockerfile`.
#              Do NOT edit it; every setting comes from .env through docker-compose.yml.
#
# WHERE TO CHANGE:  nothing in this file. Every value below comes from .env
# (the FRONTEND_* keys) through docker-compose.yml:
#
#   build arg      .env key               meaning                                  default
#   NODE_VERSION   FRONTEND_NODE_VERSION  Node used to build                       22
#   BUILD_SCRIPT   FRONTEND_BUILD_SCRIPT  npm script making the prod bundle        build
#   BUILD_ARGS     FRONTEND_BUILD_ARGS    extra arguments for the build script,    ""
#                                         e.g. --configuration development
#   PRE_BUILD_CMD  FRONTEND_PRE_BUILD_CMD a command run just before the build       ""
#   DIST_DIR       FRONTEND_DIST_DIR      folder with index.html; "" = detect      ""
#   BUILD_ENV      FRONTEND_BUILD_ENV     NAME=value pairs exported for the build  ""
#                                         (for a prebuild script that reads env)
#   BUILD_APK      FRONTEND_BUILD_APK     extra Alpine packages for the build,     ""
#                                         e.g. openjdk17-jre-headless (OpenAPI generator)
#   EXTRA_DEST     FRONTEND_EXTRA_DEST    where an outside folder is copied to     ""
#                                         (with FRONTEND_EXTRA_DIR; e.g. a shared
#                                         config file the prebuild script reads)
#
#   run-time env   .env key               meaning
#   URL_REWRITES   FRONTEND_URL_REWRITES  space-separated  from=to  pairs. Every
#                                         occurrence of `from` in the built page is
#                                         replaced by `to` each time the container
#                                         starts. This is how a page built with
#                                         http://localhost:3000 baked in is pointed
#                                         at the real address WITHOUT a code change
#                                         or a rebuild (see the header of docker-compose.yml).
#
# Server-side rendering is not run: an Angular SSR project is served as a plain
# client-side app (index.csr.html). Needs BuildKit.
# ==============================================================================

ARG NODE_VERSION=22

# Empty stand-in so a plain `docker build` works; docker-compose replaces it with
# the folder named by FRONTEND_EXTRA_DIR (additional_contexts: extra).
FROM scratch AS extra

# ---------- Stage 1: build the static bundle ---------------------------------
FROM node:${NODE_VERSION}-alpine AS build
WORKDIR /app

# Extra Alpine packages the build needs, e.g. openjdk17-jre-headless when an
# OpenAPI generator runs during install or build.
ARG BUILD_APK=""
RUN if [ -n "$BUILD_APK" ]; then apk add --no-cache $BUILD_APK; fi

# Everything is copied before the install because install scripts (postinstall,
# prepare) of some projects read files of the project or the outside folder.
COPY . .
ARG EXTRA_DEST=""
RUN --mount=type=bind,from=extra,target=/mnt/extra \
    if [ -n "$EXTRA_DEST" ]; then mkdir -p "$EXTRA_DEST" && cp -a /mnt/extra/. "$EXTRA_DEST"/; fi

# No interactive questions from the Angular CLI during the build.
ENV CI=true NG_CLI_ANALYTICS=false
RUN npm ci --no-fund --no-audit || npm install --no-fund --no-audit

# A build script that insists on a .env file in this folder gets the committed
# .env.example (frontend build settings are addresses, not secrets; the addresses
# are corrected at start by URL_REWRITES).
RUN if [ ! -f .env ] && [ -f .env.example ]; then cp .env.example .env; fi

ARG BUILD_ENV=""
ARG BUILD_SCRIPT=build
ARG DIST_DIR=""
# A command to run just before the build, e.g. installing the packages of a
# shared config folder:  cd /config && npm install
ARG PRE_BUILD_CMD=""
RUN if [ -n "$PRE_BUILD_CMD" ]; then sh -c "$PRE_BUILD_CMD"; fi
ARG BUILD_ARGS=""
RUN if [ -n "$BUILD_ENV" ]; then export $BUILD_ENV; fi; npm run ${BUILD_SCRIPT} ${BUILD_ARGS:+-- $BUILD_ARGS}

# Find the folder holding the page (Angular 17+: dist/<app>/browser, Vite: dist,
# create-react-app: build). An SSR build has index.csr.html instead of index.html.
# With several index.html files (a second app in a sub-folder) the one nearest the
# top is the site root.
RUN set -eu; \
    if [ -n "$DIST_DIR" ]; then root="$DIST_DIR"; \
    else \
      shallowest() { awk '{ n = gsub("/", "/"); print n " " $0 }' | sort -n | head -n 1 | cut -d' ' -f2-; }; \
      idx="$(find dist build out \( -name index.html -o -name index.csr.html \) -path '*/browser/*' 2>/dev/null | shallowest)"; \
      [ -n "$idx" ] || idx="$(find dist build out -name index.html 2>/dev/null | shallowest)"; \
      [ -n "$idx" ] || { echo "No index.html found under dist/ build/ out/. Set FRONTEND_DIST_DIR." >&2; exit 1; }; \
      root="$(dirname "$idx")"; \
    fi; \
    mkdir -p /site && cp -r "$root"/. /site/; \
    if [ ! -f /site/index.html ] && [ -f /site/index.csr.html ]; then cp /site/index.csr.html /site/index.html; fi; \
    [ -f /site/index.html ] || { echo "No index.html in $root" >&2; exit 1; }; \
    echo "Serving $root"

# ---------- Stage 2: nginx serves the files ----------------------------------
FROM nginx:1.27-alpine AS runtime

# The untouched build is kept aside; the entrypoint copies it over the web root on
# every start and then applies URL_REWRITES, so restarts are safe.
COPY --from=build /site /opt/site-pristine

# nginx config. The nginx image fills the ${FRONTEND_PROXY_*} names from the
# environment each time the container starts (docker-compose sets them from .env).
ENV FRONTEND_PROXY_AUTH_PREFIX=_auth FRONTEND_PROXY_AUTH_TARGET=http://auth-service:3000 \
    FRONTEND_PROXY_ORDER_PREFIX=_order FRONTEND_PROXY_ORDER_TARGET=http://order-service:8081 \
    FRONTEND_PROXY_KEEP_PATHS=__unused__ FRONTEND_PROXY_KEEP_TARGET=http://order-service:8081 \
    FRONTEND_PROXY_KEEP2_PATHS=__unused2__ FRONTEND_PROXY_KEEP2_TARGET=http://auth-service:3000 \
    NGINX_ENVSUBST_FILTER=^FRONTEND_PROXY_
COPY <<'NGINX' /etc/nginx/templates/default.conf.template
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      '';
}

server {
  listen 80;
  server_name _;
  root /usr/share/nginx/html;
  index index.html;

  # Docker's DNS, re-resolved so a restarted backend is picked up.
  resolver 127.0.0.11 valid=10s;

  gzip on;
  gzip_min_length 1024;
  gzip_types text/css application/javascript application/json image/svg+xml;
  client_max_body_size 20m;

  proxy_http_version 1.1;
  proxy_set_header Host $http_host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  # WebSocket and server-sent events (live price streams)
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection $connection_upgrade;
  proxy_buffering off;
  proxy_read_timeout 3600s;

  # Route 1 - prefix removed:  /<prefix>/x  ->  Node API  /x
  location ~ ^/(${FRONTEND_PROXY_AUTH_PREFIX})/ {
    set $auth_upstream ${FRONTEND_PROXY_AUTH_TARGET};
    rewrite ^/(${FRONTEND_PROXY_AUTH_PREFIX})/(.*)$ /$2 break;
    proxy_pass $auth_upstream;
    # A cookie the backend scopes to /foo must reach the browser as /<prefix>/foo.
    proxy_cookie_path ~^/(.+)$ /${FRONTEND_PROXY_AUTH_PREFIX}/$1;
  }

  # Route 2 - prefix removed:  /<prefix>/x  ->  Java API  /x
  location ~ ^/(${FRONTEND_PROXY_ORDER_PREFIX})/ {
    set $order_upstream ${FRONTEND_PROXY_ORDER_TARGET};
    rewrite ^/(${FRONTEND_PROXY_ORDER_PREFIX})/(.*)$ /$2 break;
    proxy_pass $order_upstream;
    proxy_cookie_path ~^/(.+)$ /${FRONTEND_PROXY_ORDER_PREFIX}/$1;
  }

  # Route 3 - path passed through unchanged:  /api/x  ->  Java API /api/x
  location ~ ^/(${FRONTEND_PROXY_KEEP_PATHS})(/|$) {
    set $keep_upstream ${FRONTEND_PROXY_KEEP_TARGET};
    proxy_pass $keep_upstream;
  }

  # Route 4 - path passed through unchanged, to the Node API
  location ~ ^/(${FRONTEND_PROXY_KEEP2_PATHS})(/|$) {
    set $keep2_upstream ${FRONTEND_PROXY_KEEP2_TARGET};
    proxy_pass $keep2_upstream;
  }

  # Single-page app: unknown paths get index.html and the app routes them.
  location / {
    try_files $uri $uri/ /index.html;
  }

  location ~* \.(?:js|mjs|css|html|json|webmanifest)$ {
    add_header Cache-Control "no-cache";
    try_files $uri =404;
  }

  location ~* \.(?:png|jpe?g|gif|svg|ico|webp|woff2?|ttf)$ {
    add_header Cache-Control "public, max-age=2592000";
    try_files $uri =404;
  }
}
NGINX

# nginx's image runs every executable /docker-entrypoint.d/*.sh before starting.
COPY --chmod=755 <<'SCRIPT' /docker-entrypoint.d/40-url-rewrites.sh
#!/bin/sh
# Points the built page at the real backend addresses. Runs on every container start.
set -eu
HTML_ROOT="${HTML_ROOT:-/usr/share/nginx/html}"
PRISTINE_ROOT="${PRISTINE_ROOT:-/opt/site-pristine}"

mkdir -p "$HTML_ROOT"
find "$HTML_ROOT" -mindepth 1 -delete
cp -a "$PRISTINE_ROOT"/. "$HTML_ROOT"/

re()  { printf '%s' "$1" | sed -e 's/[][\.*^$]/\\&/g'; }
rep() { printf '%s' "$1" | sed -e 's/[&|\\]/\\&/g'; }

for pair in ${URL_REWRITES:-}; do
  from="${pair%%=*}"
  to="${pair#*=}"
  to="${to%/}"
  [ -n "$from" ] || continue
  n=0
  for f in $(grep -rlF -- "$from" "$HTML_ROOT" 2>/dev/null || true); do
    sed -i "s|$(re "$from")|$(rep "$to")|g" "$f"
    n=$((n + 1))
  done
  echo "frontend: $from -> $to ($n file(s))"
done
exit 0
SCRIPT

EXPOSE 80
HEALTHCHECK --interval=15s --timeout=5s --start-period=5s --retries=5 \
    CMD wget -qO /dev/null http://127.0.0.1/ || exit 1
