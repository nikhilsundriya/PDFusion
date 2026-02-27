# Deploy with Docker / Podman

The easiest way to self-host PDFusion in a production environment.

> [!IMPORTANT]
> **Required Headers for Office File Conversion**
>
> LibreOffice-based tools (Word, Excel, PowerPoint conversion) require these HTTP headers for `SharedArrayBuffer` support:
>
> - `Cross-Origin-Opener-Policy: same-origin`
> - `Cross-Origin-Embedder-Policy: require-corp`
>
> The official container images include these headers. If using a reverse proxy (Traefik, Caddy, etc.), ensure these headers are preserved or added.

> [!TIP]
> **Podman Users:** All `docker` commands work with Podman by replacing `docker` with `podman` and `docker-compose` with `podman-compose`.

## Quick Start

```bash
# Docker
docker run -d \
  --name PDFusion \
  -p 3000:8080 \
  --restart unless-stopped \
  ghcr.io/nikhilsundriya/PDFusion:latest

# Podman
podman run -d \
  --name PDFusion \
  -p 3000:8080 \
  ghcr.io/nikhilsundriya/PDFusion:latest
```

## Docker Compose / Podman Compose

Create `docker-compose.yml`:

```yaml
services:
  PDFusion:
    image: ghcr.io/nikhilsundriya/PDFusion:latest
    container_name: PDFusion
    ports:
      - '3000:8080'
    restart: unless-stopped
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:8080']
      interval: 30s
      timeout: 10s
      retries: 3
```

Run:

```bash
# Docker Compose
docker compose up -d

# Podman Compose
podman-compose up -d
```

## Build Your Own Image

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginxinc/nginx-unprivileged:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

Build and run:

```bash
docker build -t PDFusion:custom .
docker run -d -p 3000:8080 PDFusion:custom
```

## Environment Variables

| Variable                | Description                     | Default                                                        |
| ----------------------- | ------------------------------- | -------------------------------------------------------------- |
| `SIMPLE_MODE`           | Build without LibreOffice tools | `false`                                                        |
| `BASE_URL`              | Deploy to subdirectory          | `/`                                                            |
| `VITE_WASM_PYMUPDF_URL` | PyMuPDF WASM module URL         | `https://cdn.jsdelivr.net/npm/@PDFusion/pymupdf-wasm@0.11.16/` |
| `VITE_WASM_GS_URL`      | Ghostscript WASM module URL     | `https://cdn.jsdelivr.net/npm/@PDFusion/gs-wasm/assets/`       |
| `VITE_WASM_CPDF_URL`    | CoherentPDF WASM module URL     | `https://cdn.jsdelivr.net/npm/coherentpdf/dist/`               |
| `VITE_DEFAULT_LANGUAGE` | Default UI language             | `en`                                                           |
| `VITE_BRAND_NAME`       | Custom brand name               | `PDFusion`                                                     |
| `VITE_BRAND_LOGO`       | Logo path relative to `public/` | `images/favicon-no-bg.svg`                                     |
| `VITE_FOOTER_TEXT`      | Custom footer/copyright text    | `© 2026 PDFusion. All rights reserved.`                        |

WASM module URLs are pre-configured with CDN defaults — all advanced features work out of the box. Override these for air-gapped or self-hosted deployments.

`VITE_DEFAULT_LANGUAGE` sets the UI language for first-time visitors. Supported values: `en`, `ar`, `be`, `fr`, `de`, `es`, `zh`, `zh-TW`, `vi`, `tr`, `id`, `it`, `pt`, `nl`, `da`. Users can still switch languages — this only changes the default.

Example:

```bash
# Build with French as the default language
docker build --build-arg VITE_DEFAULT_LANGUAGE=fr -t PDFusion .
docker run -d -p 3000:8080 PDFusion
```

### Custom Branding

Replace the default PDFusion logo, name, and footer text with your own. Place your logo file in the `public/` folder (or use an existing image), then pass the branding variables at build time:

```bash
docker build \
  --build-arg VITE_BRAND_NAME="AcmePDF" \
  --build-arg VITE_BRAND_LOGO="images/acme-logo.svg" \
  --build-arg VITE_FOOTER_TEXT="© 2026 Acme Corp. Internal use only." \
  -t acmepdf .
```

Branding works in both full mode and Simple Mode, and can be combined with all other build-time options.

### Custom WASM URLs (Air-Gapped / Self-Hosted)

> [!IMPORTANT]
> WASM URLs are baked into the JavaScript at **build time**. The WASM files are downloaded by the **user's browser** at runtime — Docker does not download them during the build. For air-gapped networks, you must host the WASM files on an internal server that browsers can reach.

**Full air-gapped workflow:**

```bash
# 1. On a machine WITH internet — download WASM packages
npm pack @PDFusion/pymupdf-wasm@0.11.14
npm pack @PDFusion/gs-wasm
npm pack coherentpdf

# 2. Build the image with your internal server URLs
docker build \
  --build-arg VITE_WASM_PYMUPDF_URL=https://internal-server.example.com/wasm/pymupdf/ \
  --build-arg VITE_WASM_GS_URL=https://internal-server.example.com/wasm/gs/ \
  --build-arg VITE_WASM_CPDF_URL=https://internal-server.example.com/wasm/cpdf/ \
  -t PDFusion .

# 3. Export the image
docker save PDFusion -o PDFusion.tar

# 4. Transfer PDFusion.tar + the .tgz WASM packages into the air-gapped network

# 5. Inside the air-gapped network — load and run
docker load -i PDFusion.tar

# Extract WASM packages to your internal web server
mkdir -p /var/www/wasm/pymupdf /var/www/wasm/gs /var/www/wasm/cpdf
tar xzf PDFusion-pymupdf-wasm-0.11.14.tgz -C /var/www/wasm/pymupdf --strip-components=1
tar xzf PDFusion-gs-wasm-*.tgz -C /var/www/wasm/gs --strip-components=1
tar xzf coherentpdf-*.tgz -C /var/www/wasm/cpdf --strip-components=1

# Run PDFusion
docker run -d -p 3000:8080 --restart unless-stopped PDFusion
```

Set a variable to empty string to disable that module (users must configure manually via Advanced Settings).

## Custom User ID (PUID/PGID)

For environments that require running as a specific non-root user (NAS devices, Kubernetes with security contexts, organizational policies), PDFusion provides a separate Dockerfile with LSIO-style PUID/PGID support.

### Build and Run

```bash
# Build the non-root image
docker build -f Dockerfile.nonroot -t PDFusion-nonroot .

# Run with custom UID/GID
docker run -d \
  --name PDFusion \
  -p 3000:8080 \
  -e PUID=1000 \
  -e PGID=1000 \
  --restart unless-stopped \
  PDFusion-nonroot
```

### Environment Variables

| Variable       | Description           | Default |
| -------------- | --------------------- | ------- |
| `PUID`         | User ID to run as     | `1000`  |
| `PGID`         | Group ID to run as    | `1000`  |
| `DISABLE_IPV6` | Disable IPv6 listener | `false` |

### Docker Compose

```yaml
services:
  PDFusion:
    build:
      context: .
      dockerfile: Dockerfile.nonroot
    container_name: PDFusion
    ports:
      - '3000:8080'
    environment:
      - PUID=1000
      - PGID=1000
    restart: unless-stopped
```

### How It Works

The container starts as root, creates a user with the specified PUID/PGID, adjusts ownership on all writable directories, then drops privileges using `su-exec`. The nginx process runs entirely as your specified user.

> [!NOTE]
> The standard `Dockerfile` uses `nginx-unprivileged` (UID 101) and is recommended for most deployments. Use `Dockerfile.nonroot` only when you need a specific UID/GID.

> [!WARNING]
> PUID/PGID cannot be `0` (root). The entrypoint validates inputs and will exit with an error for invalid values.

## With Traefik (Reverse Proxy)

```yaml
services:
  traefik:
    image: traefik:v2.10
    command:
      - '--providers.docker=true'
      - '--entrypoints.web.address=:80'
      - '--entrypoints.websecure.address=:443'
      - '--certificatesresolvers.letsencrypt.acme.email=you@example.com'
      - '--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json'
      - '--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web'
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt

  PDFusion:
    image: ghcr.io/nikhilsundriya/PDFusion:latest
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.PDFusion.rule=Host(`pdf.example.com`)'
      - 'traefik.http.routers.PDFusion.entrypoints=websecure'
      - 'traefik.http.routers.PDFusion.tls.certresolver=letsencrypt'
      - 'traefik.http.services.PDFusion.loadbalancer.server.port=8080'
      # Required headers for SharedArrayBuffer (LibreOffice WASM)
      - 'traefik.http.routers.PDFusion.middlewares=PDFusion-headers'
      - 'traefik.http.middlewares.PDFusion-headers.headers.customresponseheaders.Cross-Origin-Opener-Policy=same-origin'
      - 'traefik.http.middlewares.PDFusion-headers.headers.customresponseheaders.Cross-Origin-Embedder-Policy=require-corp'
    restart: unless-stopped
```

## With Caddy (Reverse Proxy)

```yaml
services:
  caddy:
    image: caddy:2
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data

  PDFusion:
    image: ghcr.io/nikhilsundriya/PDFusion:latest
    restart: unless-stopped

volumes:
  caddy_data:
```

Caddyfile:

```
pdf.example.com {
    reverse_proxy PDFusion:8080
    header Cross-Origin-Opener-Policy "same-origin"
    header Cross-Origin-Embedder-Policy "require-corp"
}
```

## Resource Limits

```yaml
services:
  PDFusion:
    image: ghcr.io/nikhilsundriya/PDFusion:latest
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
```

## Podman Quadlet (Systemd Integration)

[Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) allows you to run Podman containers as systemd services. This is ideal for production deployments on Linux systems.

### Basic Quadlet Setup

Create a container unit file at `~/.config/containers/systemd/PDFusion.container` (user) or `/etc/containers/systemd/PDFusion.container` (system):

```ini
[Unit]
Description=PDFusion - Privacy-first PDF toolkit
After=network-online.target
Wants=network-online.target

[Container]
Image=ghcr.io/nikhilsundriya/PDFusion:latest
ContainerName=PDFusion
PublishPort=3000:8080
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

### Enable and Start

```bash
# Reload systemd to detect new unit
systemctl --user daemon-reload

# Start the service
systemctl --user start PDFusion

# Enable on boot
systemctl --user enable PDFusion

# Check status
systemctl --user status PDFusion

# View logs
journalctl --user -u PDFusion -f
```

> [!TIP]
> For system-wide deployment, use `systemctl` without `--user` flag and place the file in `/etc/containers/systemd/`.

### Simple Mode Quadlet

For Simple Mode deployment, create `PDFusion-simple.container`:

```ini
[Unit]
Description=PDFusion Simple Mode - Clean PDF toolkit
After=network-online.target
Wants=network-online.target

[Container]
Image=ghcr.io/nikhilsundriya/PDFusion-simple:latest
ContainerName=PDFusion-simple
PublishPort=3000:8080
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

### Quadlet with Health Check

```ini
[Unit]
Description=PDFusion with health monitoring
After=network-online.target
Wants=network-online.target

[Container]
Image=ghcr.io/nikhilsundriya/PDFusion:latest
ContainerName=PDFusion
PublishPort=3000:8080
AutoUpdate=registry
HealthCmd=curl -f http://localhost:8080 || exit 1
HealthInterval=30s
HealthTimeout=10s
HealthRetries=3

[Service]
Restart=always
TimeoutStartSec=300

[Install]
WantedBy=default.target
```

### Auto-Update with Quadlet

Podman can automatically update containers when new images are available:

```bash
# Enable auto-update timer
systemctl --user enable --now podman-auto-update.timer

# Check for updates manually
podman auto-update

# Dry run (check without updating)
podman auto-update --dry-run
```

### Quadlet Network Configuration

For custom network configuration, create a network file `PDFusion.network`:

```ini
[Network]
Subnet=10.89.0.0/24
Gateway=10.89.0.1
```

Then reference it in your container file:

```ini
[Container]
Image=ghcr.io/nikhilsundriya/PDFusion:latest
ContainerName=PDFusion
PublishPort=3000:8080
Network=PDFusion.network
```

## Updating

```bash
# Pull latest image
docker compose pull

# Recreate container
docker compose up -d
```
