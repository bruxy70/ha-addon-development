---
name: ha-app-packaging
description: Home Assistant app (formerly add-on) packaging and configuration. Use when creating, configuring, or debugging HA apps — config.yaml, Dockerfile, repository structure, ingress, options schema, translations, or publishing to HACS or the official repository.
---

# HA App (Add-on) Packaging — Non-Obvious Reference

This skill covers creating and packaging Home Assistant apps (formerly called add-ons). Only non-obvious details and common mistakes — standard Docker knowledge applies for everything else.

## 1. Terminology (2025+)

"Add-on" has been renamed to "App" in the HA ecosystem. Documentation and UI are transitioning. Both terms refer to the same thing — Docker containers managed by the HA Supervisor.

## 2. Repository Structure

### Single-app repository:
```
my-ha-app/
├── config.yaml          # App metadata and configuration
├── Dockerfile           # Container build instructions
├── run.sh               # Entry point script (or any entrypoint)
├── DOCS.md              # Documentation shown in HA UI
├── CHANGELOG.md         # Version history shown in HA UI
├── icon.png             # 256x256 app icon
├── logo.png             # 256x256 app logo
├── translations/        # UI translations
│   └── en.yaml
└── apparmor.txt         # Optional: AppArmor security profile
```

### Multi-app repository:
```
my-ha-repo/
├── repository.yaml      # Repository metadata
├── my-app-1/
│   ├── config.yaml
│   ├── Dockerfile
│   └── ...
└── my-app-2/
    ├── config.yaml
    ├── Dockerfile
    └── ...
```

### `repository.yaml` (multi-app repos only):
```yaml
name: My HA Apps
url: https://github.com/user/my-ha-repo
maintainer: Your Name <email>
```

## 3. config.yaml — Key Fields

```yaml
name: "My App"
description: "Short description of what this app does"
version: "1.2.3"                    # Semantic versioning required
slug: "my_app"                      # URL-safe identifier, cannot change after publish
url: "https://github.com/user/repo" # Source code URL

# Architecture support
arch:
  - amd64
  - aarch64
  - armv7
  - armhf
  - i386

# Startup behavior
init: false                          # Set false if app manages its own process
startup: application                 # before|initialize|system|services|application
boot: auto                          # auto|manual

# Resource access
homeassistant_api: true              # Access to HA REST API via Supervisor proxy
hassio_api: true                     # Access to Supervisor API
hassio_role: default                 # default|homeassistant|backup|manager|admin

# Networking
ports:
  8080/tcp: 8080                     # container_port: host_port (null = disabled by default)
ports_description:
  8080/tcp: "Web interface"
ingress: true                        # Serve web UI through HA (recommended)
ingress_port: 8099                   # Internal port for ingress (default: 8099)
ingress_entry: "/"                   # Entry path for ingress

# Volumes and data
map:
  - config:rw                        # Mount /config (HA config dir)
  - ssl:ro                           # Mount /ssl (certificates)
  - share:rw                         # Mount /share (shared between apps)
  - media:rw                         # Mount /media
  - backup:rw                        # Mount /backup
tmpfs: true                          # Mount tmpfs at /tmp (size limited)

# Options (user configuration)
options:
  log_level: info
  port: 8080
  database_path: /config/data/myapp.db
schema:
  log_level: list(trace|debug|info|warning|error|critical)
  port: port
  database_path: str

# Security
privileged:                          # Avoid unless absolutely necessary
  - NET_ADMIN
  - SYS_ADMIN
apparmor: true                       # Enable AppArmor profile

# Dependencies
homeassistant: "2024.1.0"            # Minimum HA version
```

## 4. Options Schema Types

```yaml
schema:
  # Basic types
  name: str                          # Any string
  port: port                         # Integer 1-65535
  enabled: bool                      # Boolean
  count: int                         # Integer
  ratio: float                       # Float
  email: email                       # Valid email format
  url: url                           # Valid URL
  password: password                 # Masked in UI

  # Constrained types
  log_level: list(debug|info|warning|error)  # Enum
  timeout: int(1,3600)               # Integer with min,max

  # Optional (with ?)
  api_key: str?                      # Optional string

  # Lists
  allowed_ips:
    - str                            # List of strings

  # Nested objects
  database:
    host: str
    port: port
    name: str
```

**Gotcha:** The schema validates on save in the UI but NOT at app startup. Always validate options in your app code too.

## 5. Ingress — Non-Obvious Details

Ingress serves your web UI through the HA reverse proxy, avoiding extra ports.

**How it works:**
- HA proxies requests to `http://localhost:{ingress_port}` inside the container
- Your app receives requests at the ingress path (available via `$INGRESS_ENTRY` env var)
- All requests are authenticated through HA — no separate login needed

**Critical requirements:**
- Your app MUST handle the base path from `$INGRESS_ENTRY` — all links, API calls, and static assets must be relative or prefixed
- Set `ingress_port` to match what your app listens on inside the container
- WebSocket connections work through ingress

**Common mistakes:**
```python
# ❌ WRONG — hardcoded root path
app = FastAPI()

# ✅ CORRECT — use ingress entry as root path
import os
ingress_entry = os.environ.get("INGRESS_ENTRY", "/")
app = FastAPI(root_path=ingress_entry)
```

```javascript
// ❌ WRONG — hardcoded API URL
fetch('/api/data')

// ✅ CORRECT — relative to document base
fetch(`${document.baseURI}api/data`)
// Or read from a <base href="..."> tag set server-side
```

## 6. Accessing HA API from Inside the App

When `homeassistant_api: true` is set:

```python
import os

SUPERVISOR_TOKEN = os.environ["SUPERVISOR_TOKEN"]
HA_API_URL = "http://supervisor/core/api"

headers = {
    "Authorization": f"Bearer {SUPERVISOR_TOKEN}",
    "Content-Type": "application/json",
}

# Get entity state
response = requests.get(f"{HA_API_URL}/states/sensor.temperature", headers=headers)

# Call a service
requests.post(
    f"{HA_API_URL}/services/light/turn_on",
    headers=headers,
    json={"entity_id": "light.living_room"},
)
```

**Key points:**
- Use `http://supervisor/core/api` — NOT the external HA URL
- The `SUPERVISOR_TOKEN` is auto-injected, rotated on restart — never hardcode it
- This token has the permissions defined by `hassio_role`

## 7. Supervisor API Access

When `hassio_api: true`:

```python
# Get app info
requests.get("http://supervisor/addons/self/info", headers=headers)

# Get app options (user config)
requests.get("http://supervisor/addons/self/options", headers=headers)

# Get Supervisor info
requests.get("http://supervisor/info", headers=headers)
```

## 8. Dockerfile — HA-Specific Patterns

```dockerfile
ARG BUILD_FROM
FROM ${BUILD_FROM}

# BUILD_FROM is injected by the HA build system
# It resolves to the correct base image for each architecture

# Install dependencies
RUN apk add --no-cache python3 py3-pip nodejs npm

# Copy app files
COPY rootfs /
COPY run.sh /

# Set permissions
RUN chmod a+x /run.sh

# Build args available from config.yaml
ARG BUILD_ARCH
ARG BUILD_VERSION

CMD [ "/run.sh" ]
```

**Base images:** Use `ghcr.io/home-assistant/{arch}-base` or `ghcr.io/home-assistant/{arch}-base-python` for Python apps.

**Multi-stage builds work** and are recommended for compiled languages or frontend builds.

## 9. Data Persistence

- **`/data`** — persists across app restarts and updates. Use for app databases, state files.
- **`/config`** — HA config directory (if mapped). Use for files the user needs to access.
- **`/share`** — shared across all apps. Use for inter-app data exchange.
- **`/tmp`** — cleared on restart. Use for temporary files.

**Critical:** `/data` is the only directory that survives app updates. Everything else in the container is rebuilt.

## 10. Translations

`translations/en.yaml`:
```yaml
configuration:
  log_level:
    name: Log Level
    description: The verbosity of log output
  port:
    name: Web UI Port
    description: Port for the web interface
  database_path:
    name: Database Path
    description: Path to the SQLite database file
```

File must match the options keys in `config.yaml` exactly.

## 11. Publishing

### Adding to HACS (Home Assistant Community Store):
1. Repository must be public on GitHub
2. Add `hacs.json` to the root:
```json
{
  "name": "My App",
  "homeassistant": "2024.1.0"
}
```
3. Submit via HACS default repository process

### Adding as a custom repository:
Users add your GitHub URL in **Settings → Apps → ⋮ → Repositories → Add**.

### Official HA app store:
Submit a PR to the `home-assistant/addons` repository. Requires review and quality standards.

## 12. Debugging

**View app logs:**
- HA UI: **Settings → Apps → Your App → Log**
- API: `GET http://supervisor/addons/{slug}/logs`

**Common issues:**
- App won't start → check Dockerfile `CMD`, file permissions (`chmod +x`)
- Ingress shows blank page → check `$INGRESS_ENTRY` handling, base path in frontend
- Can't reach HA API → verify `homeassistant_api: true` in config.yaml
- Options not loading → schema mismatch between `options` and `schema` in config.yaml
- Architecture build fails → verify `BUILD_FROM` arg and base image availability
