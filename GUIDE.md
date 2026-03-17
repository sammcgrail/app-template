# Adding a New App to sebland.com

Step-by-step guide for deploying a new app at `sebland.com/<app-name>`.

---

## Architecture

```
Internet
  └── box-web-1 (Caddy, ports 80+443, handles TLS via Cloudflare + Let's Encrypt)
        ├── sebland.com/          → box SPA (/srv static files)
        ├── sebland.com/seb*      → seb-status systemd (port 8765)
        ├── sebland.com/mptodo*   → box-mptodo-1 container (port 8767)
        ├── sebland.com/ascii*    → ascii container (port 8768)
        ├── sebland.com/paint*    → paint container (port 8769)
        ├── sebland.com/mppaint*  → mppaint container (port 8770)
        ├── sebland.com/tetris*   → tetris container (port 8771)
        └── sebland.com/brick*    → brick container (port 8772)
```

Each app is its own Docker container with its own `docker-compose.yml`, proxied by Caddy in `box/app/Caddyfile`. Caddy auto-provisions HTTPS. DNS goes through Cloudflare (Full strict SSL).

---

## Port Registry

| Port | Service | Type |
|------|---------|------|
| 80, 443 | Caddy (TLS, routing) | Docker (box-web-1) |
| 6001 | signal-cli REST API | Docker (127.0.0.1 only) |
| 8765 | seb-status dashboard | systemd (not Docker) |
| 8767 | mptodo | Docker (box-mptodo-1) |
| 8768 | ascii (Game of Life) | Docker |
| 8769 | paint (MSPaint clone) | Docker |
| 8770 | mppaint (Multiplayer Paint) | Docker |
| 8771 | tetris | Docker |
| 8772 | brick (Brick Breaker) | Docker |
| 8773 | minesweeper (Win95 Minesweeper) | Docker |
| **8774+** | **next available** | — |

---

## Step 1: Create the GitHub repo

```bash
gh repo create sammcgrail/<app-name> --public --clone
cd /root/<app-name>
git commit --allow-empty -m "init"
git push origin main
gh repo add-collaborator sammcgrail/<app-name> svenflow --permission push
```

## Step 2: Required files

Every app needs these files at minimum:

### `docker-compose.yml`

```yaml
services:
  <app-name>:
    build: .
    container_name: <app-name>
    restart: unless-stopped
    ports:
      - "<PORT>:8080"    # See port binding gotcha below
    mem_limit: 32m
    cpus: 0.1
```

### `nginx.conf` (for static apps)

```nginx
server {
    listen 8080;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### `Dockerfile` (static app pattern)

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist/ /usr/share/nginx/html/dist/
COPY index.html /usr/share/nginx/html/
EXPOSE 8080
```

For self-contained single-file apps (no build step):

```dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/
EXPOSE 8080
```

### `Dockerfile` (WebSocket/server app pattern)

```dockerfile
FROM oven/bun:1-alpine
WORKDIR /app
COPY package.json bun.lock* ./
RUN bun install --production
COPY . .
EXPOSE <PORT>
CMD ["bun", "run", "server.js"]
```

### `.dockerignore`

```
node_modules
dist
.git
```

---

## Step 3: Add Caddy route

Edit `/root/box/app/Caddyfile`. Add the route **BEFORE** the SPA catch-all `handle { }` block.

### For apps with external assets (JS bundles, CSS files, images)

This is the **recommended pattern** for any app that loads external resources (scripts, stylesheets, etc.). Without the trailing-slash redirect, relative paths in the HTML will resolve incorrectly.

```
# <app-name> — description
@<app-name>_noslash path /<app-name>
redir @<app-name>_noslash /<app-name>/ permanent
handle /<app-name>/* {
    uri strip_prefix /<app-name>
    reverse_proxy host.docker.internal:<PORT>
}
```

**Why the redirect is needed:** When a browser visits `sebland.com/brick` (no trailing slash), the HTML loads fine. But a relative script src like `<script src="dist/bundle.js">` resolves to `sebland.com/dist/bundle.js` — which misses the `/brick*` Caddy route entirely and gets served the main SPA HTML instead. The browser then rejects it with `MIME type ('text/html') is not executable`. Redirecting `/brick` → `/brick/` makes relative paths resolve to `/brick/dist/bundle.js`, which correctly matches the route.

### For self-contained single-file apps (no external assets)

If the entire app is a single `index.html` with inline CSS and JS (no external scripts or stylesheets), the simpler pattern works:

```
# <app-name> — description
handle /<app-name>* {
    uri strip_prefix /<app-name>
    reverse_proxy host.docker.internal:<PORT>
}
```

This works for apps like paint, tetris, and ascii because there are no relative resource paths to break.

### For apps that handle their own prefix (no strip)

If the app is aware of its URL prefix (e.g., a FastAPI backend serving at `/mptodo/...`):

```
handle /<app-name>* {
    reverse_proxy host.docker.internal:<PORT>
}
```

---

## Step 4: Deploy

```bash
# Build and start the app container
cd /root/<app-name>
docker compose build && docker compose up -d

# Rebuild Caddy (only if Caddyfile changed)
cd /root/box
docker compose build web && docker compose up -d web
```

## Step 5: Verify

```bash
# Should return 200
curl -s -o /dev/null -w "%{http_code}" https://sebland.com/<app-name>

# If the app has external assets, also verify they serve with correct MIME type
curl -s -o /dev/null -w "%{http_code} %{content_type}" https://sebland.com/<app-name>/dist/bundle.js
# Should return: 200 application/javascript (NOT text/html)
```

---

## Gotchas (learned the hard way)

### 1. Port binding: NEVER use `127.0.0.1`

```yaml
ports:
  - "8772:8080"              # CORRECT — reachable via host.docker.internal
  - "127.0.0.1:8772:8080"   # WRONG — Caddy returns 502
```

**Why:** The Caddy container reaches other services via `host.docker.internal` which resolves to `172.17.0.1` (Docker bridge gateway), not `127.0.0.1`. Binding to loopback makes the service unreachable from Caddy.

### 2. Trailing slash + relative paths = broken assets

If your app has `<script src="dist/bundle.js">` and is served at `/brick` (no trailing slash), the browser resolves it to `/dist/bundle.js` — which bypasses the Caddy route and gets served as HTML from the SPA catch-all. The browser then fails with:

```
Refused to execute script from '.../dist/bundle.js' because its MIME type ('text/html') is not executable
```

**Fix:** Use the `@noslash` redirect pattern (see Step 3 above). This redirects `/brick` → `/brick/` so relative paths resolve under the correct prefix.

This only affects apps with external resources. Self-contained single-file apps don't have this problem.

### 3. WebSocket URL must include the full path prefix

If your app uses WebSockets behind `uri strip_prefix`, the client must connect to the full prefixed path:

```javascript
// WRONG — misses the Caddy route
const ws = new WebSocket(`wss://${location.host}/ws`);

// CORRECT — includes the path prefix so Caddy routes it
const base = location.pathname.replace(/\/$/, '');
const ws = new WebSocket(`wss://${location.host}${base}/ws`);
```

Caddy strips the prefix before forwarding, so the backend receives `/ws` either way. But the client URL must include the prefix for Caddy to match the route.

### 4. `host.docker.internal` is how Caddy reaches everything

Defined in `box/docker-compose.yml` as `extra_hosts: host.docker.internal:host-gateway`. This resolves to the Docker bridge gateway IP. Services running directly on the host (like seb-status on port 8765) are also reachable this way.

### 5. Nginx config required — don't rely on defaults

If you use `nginx:alpine`, you MUST include a custom `nginx.conf` that listens on your target port (e.g., 8080). Without it, nginx defaults to port 80, and your `EXPOSE 8080` in the Dockerfile is meaningless. The Dockerfile must copy the config:

```dockerfile
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

### 6. Docker containers auto-restart

All containers use `restart: unless-stopped`. After a server reboot, Docker automatically starts them. Verify with `docker ps` after reboot.

### 7. Cloudflare DNS proxy + Full strict SSL

sebland.com DNS goes through Cloudflare. SSL mode is Full (strict). Bot fight mode should be OFF or AI agents can't verify deployed apps via curl/fetch.

---

## App patterns we've used

| App | Type | Dockerfile base | Has build step | External assets |
|-----|------|-----------------|----------------|-----------------|
| ascii | Static single-file | nginx:alpine | No | No |
| paint | Static single-file | nginx:alpine | No | No |
| tetris | Static single-file | nginx:alpine | No | No |
| brick | Static + build | node:22-alpine → nginx:alpine | Yes (esbuild) | Yes (dist/bundle.js) |
| mppaint | WebSocket server | bun:1-alpine | No | No |
| minesweeper | Static single-file | nginx:alpine | No | No |
| mptodo | API server | python:3.12-slim | Yes (bun/vite) | Yes |

---

## Workflow

1. **Seb** creates GitHub repo, pushes initial commit, adds svenflow as collaborator
2. **Sven** builds the app, opens PR with all required files (docker-compose.yml, nginx.conf, Dockerfile, app code)
3. **Seb** reviews PR — checks port binding, nginx config, Dockerfile, external asset paths
4. **Seb** merges, pulls to server, builds container, adds Caddy route, rebuilds Caddy, verifies
