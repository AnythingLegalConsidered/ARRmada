# Architecture Research: ARRmada

**Domain:** *ARR media stack installer and auto-configurator
**Researched:** 2026-02-22
**Confidence:** MEDIUM (API details from training data, not verified against live Swagger docs)

> **Important caveat:** WebSearch and WebFetch were unavailable during this research.
> All API endpoint details come from training data (Servarr wiki, Swagger specs, community docs up to mid-2025).
> Each service exposes its Swagger UI at `{base_url}/swagger` -- the implementation phase MUST verify
> endpoint shapes against actual running instances before hardcoding.

---

## System Overview

```
                          USER
                           |
              +------------+-------------+
              |                          |
     +--------v--------+     +----------v---------+
     |  Textual TUI    |     |  FastAPI Web Wizard |
     |  (CLI frontend) |     |  (browser frontend) |
     +--------+--------+     +----------+----------+
              |                          |
              +------------+-------------+
                           |
                  +--------v--------+
                  |  Core Engine     |
                  |  (Orchestrator)  |
                  +--------+--------+
                           |
          +------+---------+---------+------+
          |      |         |         |      |
    +-----v-+ +-v------+ +v-------+ +v---+ +v---------+
    |Config | |Compose | |Deploy  | |Heal| |Auto-     |
    |Model  | |Gen     | |Manager | |th  | |Configur- |
    |(Pydan)| |(Jinja2)| |(docker)| |Chk | |ator      |
    +-------+ +--------+ +--------+ +----+ +-----+----+
                                                  |
                    +-----------+----------+------+------+-----+
                    |           |          |      |      |     |
              +-----v--+ +-----v--+ +-----v-+ +-v----+ +v---+ |
              |Prowlarr| |Sonarr/ | |Jelly- | |Bazarr| |Recy| |
              |Client  | |Radarr  | |seerr  | |Client| |cla-| |
              |        | |Client  | |Client | |      | |rr  | |
              +--------+ +--------+ +-------+ +------+ +----+ |
                                                               |
                                                         +-----v------+
                                                         | qBittorrent|
                                                         | Client     |
                                                         +------------+

        Each "Client" = httpx async wrapper around that service's REST API
```

### Component Responsibilities

| Component | Responsibility | Communicates With |
|-----------|----------------|-------------------|
| **Config Model** (Pydantic) | Validates user choices, serializes to YAML, provides typed config | All components read config |
| **Compose Generator** (Jinja2) | Renders `docker-compose.yml` + `.env` from config model | Config Model (reads), filesystem (writes) |
| **Deploy Manager** | Runs `docker compose up/down/pull`, manages container lifecycle | Docker daemon (subprocess) |
| **Health Checker** | Polls service HTTP endpoints until healthy, with timeout | Running containers (HTTP) |
| **Auto-Configurator** | Orchestrates API wiring sequence across all services | All service API clients |
| **API Key Extractor** | Reads config.xml files from mounted volumes to get API keys | Filesystem (config volumes) |
| **Service API Clients** | Typed wrappers around each service's REST API | Individual *ARR services (HTTP) |
| **Textual TUI** | Terminal-based wizard for service selection, config | Core Engine |
| **FastAPI Web Wizard** | Browser-based wizard with WebSocket progress | Core Engine |

---

## Recommended Project Structure

```
src/arrmada/
├── __init__.py
├── cli.py                    # Textual TUI entry point
├── main.py                   # Core orchestrator (init/deploy/configure/status/destroy)
│
├── models/                   # Pydantic models
│   ├── __init__.py
│   ├── config.py             # UserConfig: tier, paths, VPN, selected services
│   ├── services.py           # ServiceDefinition: name, image, ports, volumes, deps
│   └── state.py              # DeploymentState: running, api_keys, wiring status
│
├── core/                     # Engine components
│   ├── __init__.py
│   ├── compose.py            # Jinja2 docker-compose rendering
│   ├── deployer.py           # docker compose up/down/pull subprocess wrapper
│   ├── health.py             # HTTP health polling with exponential backoff
│   ├── api_keys.py           # config.xml parser for API key extraction
│   └── directories.py        # Create /data tree with permissions
│
├── clients/                  # One async httpx client per service API
│   ├── __init__.py
│   ├── base.py               # BaseArrClient with auth, retry, error handling
│   ├── prowlarr.py           # ProwlarrClient: add_application(), test_connection()
│   ├── sonarr.py             # SonarrClient: add_download_client(), add_root_folder()
│   ├── radarr.py             # RadarrClient: add_download_client(), add_root_folder()
│   ├── jellyseerr.py         # JellyseerrClient: setup_jellyfin(), add_sonarr(), add_radarr()
│   ├── bazarr.py             # BazarrClient: configure_sonarr(), configure_radarr()
│   ├── qbittorrent.py        # QBittorrentClient: login(), get_preferences()
│   └── recyclarr.py          # RecyclarrRunner: generate_config(), run_sync()
│
├── configurator/             # Auto-wiring orchestration
│   ├── __init__.py
│   ├── wiring.py             # WiringOrchestrator: run_full_wiring() with dependency order
│   └── steps.py              # Individual wiring steps (each is idempotent)
│
├── templates/                # Jinja2 templates
│   ├── docker-compose.yml.j2 # Main compose with conditional service blocks
│   ├── .env.j2               # Environment variables
│   └── recyclarr.yml.j2      # Recyclarr config with API keys injected
│
└── web/                      # Web wizard (Phase 3)
    ├── app.py                # FastAPI app
    ├── api/
    │   ├── setup.py          # Wizard step endpoints
    │   └── status.py         # Health/progress endpoints
    └── static/               # Minimal frontend (Alpine.js or vanilla)
```

### Structure Rationale

- **`models/`** separate from `core/`: Config model is read by everything -- it is the contract between UI and engine. Isolating it prevents circular deps.
- **`clients/`** not `services/`: The word "services" is overloaded (Docker services vs API clients). `clients/` is unambiguous -- these are HTTP API client wrappers.
- **`configurator/`** as orchestration layer: The wiring sequence has complex ordering (API keys first, then Prowlarr, then download clients, etc.). This layer owns the ordering logic, while `clients/` own the API calls.
- **`templates/`** with Jinja2: One template with conditionals beats N separate compose files. Jinja2 `{% if config.services.bazarr %}` blocks keep everything in one file.

---

## API Key Extraction (Step 0 -- Foundation)

**Confidence: HIGH** (well-documented, consistent across all Servarr-based apps)

All Servarr-based applications (Sonarr, Radarr, Prowlarr, Lidarr, Readarr, Bazarr) generate a `config.xml` file on first boot. The API key is inside this file.

### config.xml Location (inside container)

| Service | Config Path (container) | Config Path (host mount) |
|---------|------------------------|--------------------------|
| Sonarr | `/config/config.xml` | `{install_path}/config/sonarr/config.xml` |
| Radarr | `/config/config.xml` | `{install_path}/config/radarr/config.xml` |
| Prowlarr | `/config/config.xml` | `{install_path}/config/prowlarr/config.xml` |
| Lidarr | `/config/config.xml` | `{install_path}/config/lidarr/config.xml` |
| Readarr | `/config/config.xml` | `{install_path}/config/readarr/config.xml` |
| Bazarr | N/A (uses `config.ini` or DB) | See Bazarr section below |

### config.xml Structure

```xml
<Config>
  <BindAddress>*</BindAddress>
  <Port>8989</Port>
  <SslPort>9898</SslPort>
  <EnableSsl>False</EnableSsl>
  <LaunchBrowser>True</LaunchBrowser>
  <ApiKey>abc123def456abc123def456abc123de</ApiKey>
  <AuthenticationMethod>None</AuthenticationMethod>
  <Branch>main</Branch>
  <LogLevel>info</LogLevel>
  <UrlBase></UrlBase>
  <InstanceName>Sonarr</InstanceName>
</Config>
```

### Extraction Code Pattern

```python
import xml.etree.ElementTree as ET
from pathlib import Path

def extract_api_key(config_path: Path) -> str:
    """Extract API key from a Servarr config.xml file."""
    tree = ET.parse(config_path)
    root = tree.getroot()
    api_key_element = root.find("ApiKey")
    if api_key_element is None or not api_key_element.text:
        raise ValueError(f"No ApiKey found in {config_path}")
    return api_key_element.text
```

### Bazarr Special Case

Bazarr is NOT a Servarr application. It stores its API key differently:
- **Bazarr uses a SQLite database** (`config/db/bazarr.db`) or a `config.ini`/`config.yaml` depending on version.
- The API key is in the `system` table of the SQLite DB, or in a config file.
- **Alternative approach**: Bazarr also exposes its API key via its settings page, and can be set via environment variables.
- **Recommended**: Check `config/bazarr/config/config.yaml` first (newer versions), fall back to `config/bazarr/db/bazarr.db` SQLite query.

**Confidence for Bazarr: LOW** -- Bazarr config storage has changed between versions. Must verify against current version during implementation.

### Jellyseerr Special Case

Jellyseerr does NOT use config.xml. It stores its API key in its internal SQLite database.
- The API key is generated during the initial setup wizard (first-run).
- **Problem**: Jellyseerr requires manual first-run setup through its web UI (user creation, Jellyfin connection). There is no documented way to skip this programmatically.
- **Recommended approach**: Use the Jellyseerr API to complete the setup flow programmatically (see Jellyseerr section below).

### qBittorrent Special Case

qBittorrent does not use API keys. It uses session-based authentication:
- `POST /api/v2/auth/login` with `username` and `password` form data.
- Returns a session cookie (`SID`).
- Default credentials: `admin` / random password printed to container logs on first boot (since qBittorrent 4.6.1+).
- **Important change**: Since qBittorrent 4.6.1, the default password is NO LONGER `adminadmin`. A random password is generated and logged. ARRmada must either:
  1. Read the temporary password from Docker logs: `docker logs qbittorrent 2>&1 | grep "temporary password"`
  2. Or set a known password via environment variables (`WEBUI_PASSWORD` in some images like `linuxserver/qbittorrent`).

**Confidence for qBittorrent password change: MEDIUM** -- widely reported in community, but env var approach depends on container image used.

### Timing: Wait for config.xml to Exist

Services need time after first boot to generate `config.xml`. The health checker must confirm:
1. Container is running (Docker API)
2. HTTP endpoint responds (e.g., `GET /api/v3/system/status` returns 200)
3. config.xml exists on disk

```python
async def wait_for_api_key(config_path: Path, timeout: float = 120.0) -> str:
    """Poll until config.xml appears and contains an API key."""
    start = time.monotonic()
    while time.monotonic() - start < timeout:
        if config_path.exists():
            try:
                return extract_api_key(config_path)
            except (ET.ParseError, ValueError):
                pass  # File exists but not fully written yet
        await asyncio.sleep(2.0)
    raise TimeoutError(f"API key not available in {config_path} after {timeout}s")
```

---

## Prowlarr API: Adding Applications

**Confidence: MEDIUM** (based on Prowlarr Swagger docs from training data, endpoint shape may vary by version)

Prowlarr uses API v1. All requests require `X-Api-Key` header or `?apikey=` query param.

**Swagger UI**: `http://prowlarr:9696/swagger`

### Add Application (Sonarr/Radarr)

```
POST /api/v1/applications
Content-Type: application/json
X-Api-Key: {prowlarr_api_key}
```

**Body for adding Sonarr:**

```json
{
  "name": "Sonarr",
  "syncLevel": "fullSync",
  "implementation": "Sonarr",
  "implementationName": "Sonarr",
  "configContract": "SonarrSettings",
  "fields": [
    { "name": "prowlarrUrl", "value": "http://prowlarr:9696" },
    { "name": "baseUrl", "value": "http://sonarr:8989" },
    { "name": "apiKey", "value": "{sonarr_api_key}" },
    { "name": "syncCategories", "value": [5000, 5010, 5020, 5030, 5040, 5045, 5050] }
  ],
  "tags": []
}
```

**Body for adding Radarr:**

```json
{
  "name": "Radarr",
  "syncLevel": "fullSync",
  "implementation": "Radarr",
  "implementationName": "Radarr",
  "configContract": "RadarrSettings",
  "fields": [
    { "name": "prowlarrUrl", "value": "http://prowlarr:9696" },
    { "name": "baseUrl", "value": "http://radarr:7878" },
    { "name": "apiKey", "value": "{radarr_api_key}" },
    { "name": "syncCategories", "value": [2000, 2010, 2020, 2030, 2040, 2045, 2050, 2060, 2070, 2080] }
  ],
  "tags": []
}
```

### Key Fields Explained

| Field | Purpose | Values |
|-------|---------|--------|
| `syncLevel` | How indexers sync to apps | `"fullSync"` (add+remove), `"addOnly"`, `"disabled"` |
| `prowlarrUrl` | How the app reaches back to Prowlarr | Must be reachable from the app container |
| `baseUrl` | How Prowlarr reaches the app | Docker service name + port |
| `apiKey` | The target app's API key | Extracted from config.xml |
| `syncCategories` | Newznab categories to sync | 5000-series for TV, 2000-series for Movies |

### Sync Categories Reference

| Category Range | Content Type |
|---------------|--------------|
| 2000-2080 | Movies |
| 3000-3040 | Audio/Music |
| 5000-5050 | TV |
| 7000-7020 | Books |

### Test Before Add

```
POST /api/v1/applications/test
```
Same body as above. Returns 200 if connection works, or error details. Call this before the actual POST to /applications.

### Lidarr/Readarr (Tier 2)

Same pattern, different `implementation` and `configContract`:
- Lidarr: `implementation: "Lidarr"`, `configContract: "LidarrSettings"`, categories `[3000, 3010, 3020, 3030, 3040]`
- Readarr: `implementation: "Readarr"`, `configContract: "ReadarrSettings"`, categories `[7000, 7010, 7020]`

---

## Sonarr/Radarr API: Download Clients and Root Folders

**Confidence: MEDIUM** (Servarr API v3 is well-documented but field names vary slightly between versions)

Sonarr and Radarr share the same Servarr codebase. Their APIs are nearly identical. Both use API v3.

**Swagger UI**: `http://sonarr:8989/swagger` / `http://radarr:7878/swagger`

### Add Download Client (qBittorrent)

```
POST /api/v3/downloadclient
Content-Type: application/json
X-Api-Key: {sonarr_api_key}
```

**Body:**

```json
{
  "name": "qBittorrent",
  "implementation": "QBittorrent",
  "implementationName": "qBittorrent",
  "configContract": "QBittorrentSettings",
  "protocol": "torrent",
  "enable": true,
  "priority": 1,
  "removeCompletedDownloads": true,
  "removeFailedDownloads": true,
  "fields": [
    { "name": "host", "value": "qbittorrent" },
    { "name": "port", "value": 8080 },
    { "name": "username", "value": "admin" },
    { "name": "password", "value": "{qbittorrent_password}" },
    { "name": "category", "value": "tv-sonarr" },
    { "name": "recentPriority", "value": 0 },
    { "name": "olderPriority", "value": 0 },
    { "name": "initialState", "value": 0 },
    { "name": "sequentialOrder", "value": false },
    { "name": "firstAndLast", "value": false }
  ],
  "tags": []
}
```

**Key differences between Sonarr and Radarr:**
- Sonarr: `category` should be `"tv-sonarr"` (convention)
- Radarr: `category` should be `"radarr"` (convention)
- The category field tells qBittorrent to sort downloads, enabling Sonarr/Radarr to find their own downloads.

### Important: host field when Gluetun is active

When qBittorrent uses `network_mode: "service:gluetun"`, it shares Gluetun's network namespace. Other containers must reach qBittorrent via `gluetun` hostname (not `qbittorrent`):

```json
{ "name": "host", "value": "gluetun" }
```

This is a critical detail ARRmada must handle conditionally based on VPN config.

### Test Download Client

```
POST /api/v3/downloadclient/test
```
Same body. Returns 200 on success or validation errors.

### Add Root Folder

```
POST /api/v3/rootfolder
Content-Type: application/json
X-Api-Key: {sonarr_api_key}
```

**Body (Sonarr):**

```json
{
  "path": "/data/media/tv",
  "accessible": true,
  "freeSpace": 0,
  "unmappedFolders": []
}
```

**Body (Radarr):**

```json
{
  "path": "/data/media/movies",
  "accessible": true,
  "freeSpace": 0,
  "unmappedFolders": []
}
```

The `path` is the path **inside the container** -- since all containers mount the same `/data` volume, this path is consistent.

### Check Existing Root Folders

```
GET /api/v3/rootfolder
```

Always check first. If root folder already exists, skip adding. This is key for idempotency.

### Check Existing Download Clients

```
GET /api/v3/downloadclient
```

Same pattern -- check before adding to avoid duplicates.

---

## Jellyseerr API: Setup and Connections

**Confidence: LOW** (Jellyseerr API is less documented than Servarr APIs; behavior from community reports)

Jellyseerr is the trickiest service to auto-configure because it has a **first-run setup wizard** that must be completed before the API is usable.

### The First-Run Problem

On first boot, Jellyseerr presents a setup wizard at `/setup`. The API returns 403 for most endpoints until setup is complete. The setup flow is:

1. **Create admin user** (or use Jellyfin login)
2. **Connect to Jellyfin** (server URL, credentials)
3. **Sync libraries**
4. **Configure Sonarr/Radarr** (optional during setup, can be done after)

### Programmatic Setup Flow

**Step 1: Initialize Jellyfin connection**

```
POST /api/v1/settings/jellyfin
Content-Type: application/json
```

```json
{
  "hostname": "http://jellyfin",
  "port": 8096,
  "useSsl": false,
  "urlBase": "",
  "externalHostname": "",
  "jellyfinForgotPasswordUrl": ""
}
```

**Step 2: Sign in with Jellyfin user (creates admin)**

```
POST /api/v1/auth/jellyfin
Content-Type: application/json
```

```json
{
  "username": "{jellyfin_admin_user}",
  "password": "{jellyfin_admin_password}"
}
```

This returns a session token to use for subsequent API calls.

**Step 3: Sync Jellyfin libraries**

```
POST /api/v1/settings/jellyfin/library
Content-Type: application/json
```

```json
{
  "libraries": []
}
```

(Empty array = sync all libraries. Or pass specific library IDs.)

**Step 4: Complete setup / initialize**

```
POST /api/v1/settings/initialize
```

This finalizes the setup wizard.

**Step 5: Add Sonarr**

```
POST /api/v1/settings/sonarr
Content-Type: application/json
```

```json
{
  "name": "Sonarr",
  "hostname": "sonarr",
  "port": 8989,
  "apiKey": "{sonarr_api_key}",
  "useSsl": false,
  "baseUrl": "",
  "activeProfileId": 1,
  "activeProfileName": "Any",
  "activeDirectory": "/data/media/tv",
  "activeLanguageProfileId": 1,
  "is4k": false,
  "isDefault": true,
  "enableSeasonFolders": true,
  "externalUrl": ""
}
```

**Step 6: Add Radarr**

```
POST /api/v1/settings/radarr
Content-Type: application/json
```

```json
{
  "name": "Radarr",
  "hostname": "radarr",
  "port": 7878,
  "apiKey": "{radarr_api_key}",
  "useSsl": false,
  "baseUrl": "",
  "activeProfileId": 1,
  "activeProfileName": "Any",
  "activeDirectory": "/data/media/movies",
  "minimumAvailability": "released",
  "is4k": false,
  "isDefault": true,
  "externalUrl": ""
}
```

### Critical Jellyseerr Issue: Profile IDs

The `activeProfileId` and `activeProfileName` must match actual profiles in Sonarr/Radarr. After Recyclarr runs and creates custom profiles, the default IDs may change. **Wiring order matters**:

1. First wire Sonarr/Radarr (root folders, download clients)
2. Run Recyclarr (creates quality profiles)
3. Then query Sonarr/Radarr for available profiles: `GET /api/v3/qualityprofile`
4. Then configure Jellyseerr with correct profile IDs

### Jellyfin First-Run Problem

Jellyfin ALSO has a first-run wizard. It must be completed before Jellyseerr can connect. Jellyfin's first-run can be automated:

```
POST /Startup/User
Content-Type: application/json
```

```json
{
  "Name": "admin",
  "Password": "{password}"
}
```

Then:

```
POST /Startup/Complete
```

**Confidence: LOW** -- Jellyfin startup API endpoints are not extensively documented. Must verify during implementation.

---

## Bazarr API: Connecting to Sonarr/Radarr

**Confidence: LOW** (Bazarr's API is the least documented of all services in this stack)

Bazarr connects to Sonarr and Radarr to know which media files need subtitles.

### Bazarr Configuration API

Bazarr exposes its API at `/api/` with an API key passed as `X-API-KEY` header.

**Configure Sonarr connection:**

```
PATCH /api/system/settings
Content-Type: application/json
X-API-KEY: {bazarr_api_key}
```

```json
{
  "settings": {
    "sonarr": {
      "ip": "sonarr",
      "port": 8989,
      "apikey": "{sonarr_api_key}",
      "base_url": "",
      "ssl": false,
      "full_update": "Daily",
      "only_monitored": false,
      "series_sync": 60,
      "episodes_sync": 60
    },
    "radarr": {
      "ip": "radarr",
      "port": 7878,
      "apikey": "{radarr_api_key}",
      "base_url": "",
      "ssl": false,
      "full_update": "Daily",
      "only_monitored": false,
      "movies_sync": 60
    }
  }
}
```

**Alternative approach**: Bazarr can also be configured by directly editing its `config.yaml` file before first boot. This may be more reliable than the API:

```yaml
sonarr:
  ip: sonarr
  port: 8989
  apikey: "{sonarr_api_key}"
  base_url: ""
  ssl: false
radarr:
  ip: radarr
  port: 7878
  apikey: "{radarr_api_key}"
  base_url: ""
  ssl: false
```

**Recommendation**: Use config file injection as primary method, API as verification. The config file approach is more reliable and avoids needing Bazarr's own API key first.

---

## Gluetun + qBittorrent: VPN Network Isolation

**Confidence: HIGH** (well-documented Docker networking pattern, Gluetun docs are clear)

### How network_mode: service Works

When a Docker container uses `network_mode: "service:gluetun"`, it:
1. Shares Gluetun's network namespace (same IP, same interfaces)
2. All traffic from qBittorrent goes through Gluetun's VPN tunnel
3. qBittorrent has NO direct network access
4. Other containers reach qBittorrent through Gluetun's hostname and ports

### Docker Compose Pattern

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      # qBittorrent WebUI (exposed through Gluetun)
      - "8080:8080"
      # qBittorrent incoming connections
      - "6881:6881"
      - "6881:6881/udp"
    environment:
      - VPN_SERVICE_PROVIDER=${VPN_PROVIDER}
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${VPN_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=${VPN_ADDRESSES}
      - SERVER_COUNTRIES=${VPN_COUNTRIES}
      # Firewall: allow LAN access to qBittorrent WebUI
      - FIREWALL_INPUT_PORTS=8080
      - FIREWALL_OUTBOUND_SUBNETS=172.16.0.0/12
    volumes:
      - ${CONFIG_PATH}/gluetun:/gluetun
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"  # <-- This is the key line
    depends_on:
      gluetun:
        condition: service_healthy
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
    volumes:
      - ${CONFIG_PATH}/qbittorrent:/config
      - ${DATA_PATH}/downloads:/data/downloads
    # NOTE: NO "ports:" section here -- ports are on gluetun
    restart: unless-stopped
```

### Critical Details

1. **Ports go on Gluetun, not qBittorrent**: When using `network_mode: service:gluetun`, qBittorrent cannot define its own ports. All port mappings must be on the Gluetun service.

2. **FIREWALL_INPUT_PORTS**: Must include qBittorrent's WebUI port (8080) so other containers on the Docker network can reach it.

3. **FIREWALL_OUTBOUND_SUBNETS**: Must include the Docker network subnet (typically `172.16.0.0/12`) so qBittorrent's API can be reached by Sonarr/Radarr.

4. **depends_on with health check**: qBittorrent must wait for Gluetun to be healthy (VPN connected) before starting. Gluetun has a built-in health check.

5. **Host resolution for other services**: When other containers (Sonarr, Radarr) need to reach qBittorrent, they use `gluetun` as the hostname (not `qbittorrent`), because qBittorrent shares Gluetun's network identity.

### Jinja2 Template Conditional

```jinja2
{% if config.vpn.enabled %}
  qbittorrent:
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy
{% else %}
  qbittorrent:
    ports:
      - "${QBIT_PORT:-8080}:8080"
      - "6881:6881"
      - "6881:6881/udp"
{% endif %}
```

### Auto-Configurator Impact

The auto-configurator must know whether VPN is active to set the correct download client host:

```python
qbit_host = "gluetun" if config.vpn.enabled else "qbittorrent"
```

This affects every service that adds qBittorrent as a download client.

---

## Recyclarr: Configuration and Invocation

**Confidence: MEDIUM** (Recyclarr is well-documented at recyclarr.dev, but exact template names may change)

### How Recyclarr Works

Recyclarr is a standalone CLI tool (not a long-running service). It:
1. Reads a YAML config file
2. Fetches TRaSH Guides data from the TRaSH Guides GitHub repo
3. Applies quality profiles, custom formats, and quality definitions to Sonarr/Radarr via their APIs
4. Is idempotent -- safe to run multiple times

### Config Generation (Jinja2)

ARRmada generates Recyclarr's config with the correct API keys:

```yaml
# recyclarr.yml (generated by ARRmada)
sonarr:
  sonarr-main:
    base_url: http://sonarr:8989
    api_key: {{ sonarr_api_key }}

    include:
      # Quality definitions (file sizes)
      - template: sonarr-quality-definition-series

      # Quality profile: WEB-DL (1080p)
      - template: sonarr-v4-quality-profile-web-1080p

      # Custom formats for the above profile
      - template: sonarr-v4-custom-formats-web-1080p

    # Optional: naming convention from TRaSH
    # media_naming:
    #   series: default
    #   season: default
    #   episodes:
    #     rename: true
    #     standard: default

radarr:
  radarr-main:
    base_url: http://radarr:7878
    api_key: {{ radarr_api_key }}

    include:
      # Quality definitions (file sizes)
      - template: radarr-quality-definition-movie

      # Quality profile: HD Bluray + WEB (1080p)
      - template: radarr-quality-profile-hd-bluray-web

      # Custom formats for the above profile
      - template: radarr-custom-formats-hd-bluray-web
```

### Available Template Categories

| Template Pattern | What It Does |
|-----------------|--------------|
| `*-quality-definition-*` | Sets min/max file sizes per quality |
| `*-quality-profile-*` | Creates/updates quality profiles with ranking |
| `*-custom-formats-*` | Adds custom formats with scores |
| `*-v4-*` | Sonarr v4 specific templates |

### Invocation Approaches

**Option A: Run Recyclarr as a one-shot Docker container** (recommended)

```python
import subprocess

def run_recyclarr(config_path: str) -> subprocess.CompletedProcess:
    """Run Recyclarr sync via Docker."""
    return subprocess.run(
        [
            "docker", "run", "--rm",
            "--name", "recyclarr-sync",
            "-v", f"{config_path}:/config",
            "--network", "arrmada_default",  # Same network as *ARR services
            "ghcr.io/recyclarr/recyclarr:latest",
            "sync"
        ],
        capture_output=True,
        text=True,
        timeout=300
    )
```

**Option B: Add Recyclarr as a compose service in cron mode**

```yaml
recyclarr:
  image: ghcr.io/recyclarr/recyclarr:latest
  container_name: recyclarr
  volumes:
    - ${CONFIG_PATH}/recyclarr:/config
  environment:
    - TZ=${TZ}
    - CRON_SCHEDULE=@daily
  restart: unless-stopped
```

**Recommendation**: Use Option A for the initial sync (during `arrmada configure`), and Option B to keep profiles updated daily. ARRmada should:
1. Generate `recyclarr.yml` from template
2. Run a one-shot sync during auto-configuration
3. Include Recyclarr as a compose service for ongoing daily syncs

### Template Selection by User Preference

ARRmada can offer quality presets that map to Recyclarr templates:

| ARRmada Preset | Recyclarr Sonarr Templates | Recyclarr Radarr Templates |
|---------------|---------------------------|---------------------------|
| **Standard (1080p)** | `sonarr-v4-quality-profile-web-1080p` | `radarr-quality-profile-hd-bluray-web` |
| **High Quality (4K)** | `sonarr-v4-quality-profile-web-2160p` | `radarr-quality-profile-uhd-bluray-web` |
| **Anime** | `sonarr-v4-quality-profile-anime` | `radarr-quality-profile-anime` |
| **Remux (maximum quality)** | `sonarr-v4-quality-profile-remux-web-1080p` | `radarr-quality-profile-remux-web-1080p` |

---

## Auto-Configuration Wiring Sequence

This is the most critical architectural decision: the order in which services are wired.

### Dependency Graph

```
                    config.xml parse
                         |
                    API Keys Available
                         |
            +------------+------------+
            |            |            |
       Prowlarr     Sonarr/Radarr   qBittorrent
       (add apps)   (add download   (login, get
            |        clients +       password)
            |        root folders)       |
            |            |               |
            +------+-----+          (already done)
                   |
              Recyclarr Sync
              (quality profiles created)
                   |
              Jellyseerr Setup
              (needs profile IDs from Sonarr/Radarr)
                   |
              Bazarr Setup
              (connect to Sonarr/Radarr)
```

### Ordered Steps

```python
async def run_full_wiring(config: UserConfig) -> WiringResult:
    """
    Execute the full auto-configuration sequence.
    Each step is idempotent -- safe to re-run.
    """
    result = WiringResult()

    # Phase 0: Extract all API keys
    api_keys = await extract_all_api_keys(config)  # Parallel reads

    # Phase 1: qBittorrent auth (needed by later steps)
    qbit_password = await get_qbittorrent_password(config)
    qbit_host = "gluetun" if config.vpn.enabled else "qbittorrent"

    # Phase 2: Sonarr + Radarr setup (parallel -- independent of each other)
    async with asyncio.TaskGroup() as tg:
        tg.create_task(configure_sonarr(api_keys, qbit_host, qbit_password))
        tg.create_task(configure_radarr(api_keys, qbit_host, qbit_password))
        # Tier 2: Lidarr, Readarr in parallel too

    # Phase 3: Prowlarr (needs Sonarr/Radarr API keys, but independent of their config)
    await configure_prowlarr(api_keys)  # Add all *ARR apps

    # Phase 4: Recyclarr (needs Sonarr/Radarr to be reachable)
    if config.tier >= 2 or config.recyclarr_enabled:
        await run_recyclarr_sync(api_keys)

    # Phase 5: Jellyseerr (needs Jellyfin + Sonarr/Radarr profile IDs)
    if "jellyseerr" in config.services:
        sonarr_profiles = await get_sonarr_profiles(api_keys)
        radarr_profiles = await get_radarr_profiles(api_keys)
        await configure_jellyseerr(api_keys, sonarr_profiles, radarr_profiles)

    # Phase 6: Bazarr (needs Sonarr/Radarr API keys)
    if "bazarr" in config.services:
        await configure_bazarr(api_keys)

    # Phase 7: Huntarr (needs all *ARR API keys)
    if "huntarr" in config.services:
        await configure_huntarr(api_keys)

    # Phase 8: Final health check
    result.health = await run_health_check(config)

    return result
```

### Why This Order

1. **API keys first**: Everything depends on them. No API calls possible without them.
2. **qBittorrent password**: Needed by Sonarr/Radarr when adding download client.
3. **Sonarr/Radarr in parallel**: They don't depend on each other. Adding root folders + download clients can happen simultaneously.
4. **Prowlarr after API keys**: Prowlarr only needs API keys of target apps, not their full config. Could run in parallel with Phase 2, but serializing is safer.
5. **Recyclarr after Sonarr/Radarr**: Recyclarr modifies quality profiles in Sonarr/Radarr. They must be running and reachable.
6. **Jellyseerr last**: Needs Sonarr/Radarr profile IDs, which may be created by Recyclarr. Also needs Jellyfin to be set up first.
7. **Bazarr/Huntarr after Sonarr/Radarr**: They connect to Sonarr/Radarr, need their API keys.

---

## Architectural Patterns

### Pattern 1: Idempotent Wiring Steps

**What:** Every wiring step checks existing state before making changes. If a download client named "qBittorrent" already exists in Sonarr, skip adding it.

**When:** Always. Users will re-run `arrmada configure` after failures or changes.

**Trade-offs:** Slightly more API calls (GET before POST), but prevents duplicates and makes the tool safe to re-run.

**Example:**

```python
async def add_download_client_if_missing(
    client: SonarrClient,
    name: str,
    settings: dict
) -> bool:
    """Add download client only if one with this name doesn't exist."""
    existing = await client.get_download_clients()
    if any(dc["name"] == name for dc in existing):
        logger.info(f"Download client '{name}' already exists, skipping")
        return False

    await client.test_download_client(settings)
    await client.add_download_client(settings)
    logger.info(f"Download client '{name}' added successfully")
    return True
```

### Pattern 2: Base API Client with Shared Behavior

**What:** Abstract base class for all *ARR API clients with common auth, retry, error handling, and logging.

**When:** All service clients inherit from this.

```python
import httpx

class BaseArrClient:
    """Base client for Servarr-based APIs."""

    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key
        self._client = httpx.AsyncClient(
            base_url=self.base_url,
            headers={"X-Api-Key": self.api_key},
            timeout=30.0,
        )

    async def _get(self, path: str) -> dict:
        response = await self._client.get(path)
        response.raise_for_status()
        return response.json()

    async def _post(self, path: str, data: dict) -> dict:
        response = await self._client.post(path, json=data)
        response.raise_for_status()
        return response.json()

    async def close(self):
        await self._client.aclose()
```

### Pattern 3: Health Check with Exponential Backoff

**What:** Poll services after `docker compose up` with increasing delays until all are healthy.

**When:** Between deployment and auto-configuration.

```python
async def wait_for_service(
    url: str,
    timeout: float = 120.0,
    initial_delay: float = 2.0,
    max_delay: float = 10.0,
) -> bool:
    """Wait for a service to respond to HTTP GET."""
    delay = initial_delay
    start = time.monotonic()
    async with httpx.AsyncClient() as client:
        while time.monotonic() - start < timeout:
            try:
                resp = await client.get(url, timeout=5.0)
                if resp.status_code < 500:
                    return True
            except (httpx.ConnectError, httpx.TimeoutException):
                pass
            await asyncio.sleep(delay)
            delay = min(delay * 1.5, max_delay)
    return False
```

### Pattern 4: Conditional Compose Generation

**What:** Single Jinja2 template with conditional blocks for optional services rather than multiple template files.

**When:** Compose generation based on user selections.

**Trade-offs:** One large template is harder to read but easier to maintain than 4+ separate files. The conditionals are simple (service present or not).

```jinja2
version: "3.8"
services:
{% include 'services/prowlarr.yml.j2' %}
{% include 'services/sonarr.yml.j2' %}
{% include 'services/radarr.yml.j2' %}
{% include 'services/qbittorrent.yml.j2' %}
{% include 'services/jellyfin.yml.j2' %}
{% if 'jellyseerr' in services %}
{% include 'services/jellyseerr.yml.j2' %}
{% endif %}
{% if 'bazarr' in services %}
{% include 'services/bazarr.yml.j2' %}
{% endif %}
{# ... etc #}
```

**Actually recommended approach**: Use Jinja2 `include` with per-service fragment templates. Each service gets its own `.j2` file. The main template includes them conditionally. This balances maintainability (one file per service) with flexibility (conditional inclusion).

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Hardcoded Service URLs

**What people do:** Hardcode `http://sonarr:8989` throughout the codebase.
**Why it's wrong:** Users may use custom ports, URL bases, or non-default container names.
**Do this instead:** Derive all URLs from the config model. `config.services.sonarr.internal_url` computed from `hostname` + `port` + `url_base`.

### Anti-Pattern 2: Fire-and-Forget API Calls

**What people do:** POST to add a download client and assume it worked.
**Why it's wrong:** API may return 200 but with validation warnings. Or it may silently create a broken config.
**Do this instead:** Always call the `test` endpoint first, then POST, then GET to verify the resource was created correctly.

### Anti-Pattern 3: Sequential Service Startup

**What people do:** Wait for each service one by one before moving to the next.
**Why it's wrong:** Wastes time. Services boot independently.
**Do this instead:** Poll all services in parallel with `asyncio.gather()`. The health checker should monitor all services simultaneously and report which ones are ready.

### Anti-Pattern 4: Ignoring First-Run State

**What people do:** Assume services are ready to accept API calls immediately.
**Why it's wrong:** Jellyfin and Jellyseerr require first-run setup wizards. Sonarr/Radarr need time to generate config.xml.
**Do this instead:** Detect first-run state explicitly. Check for config.xml existence. Check Jellyseerr `/api/v1/settings/public` for setup completion status.

### Anti-Pattern 5: Monolithic Wiring Function

**What people do:** One giant function that does all wiring in sequence.
**Why it's wrong:** Can't re-run partial wiring, can't test individual steps, error in step 3 forces re-running steps 1-2.
**Do this instead:** Each wiring step is an independent, idempotent function. The orchestrator calls them in order but each can be called standalone.

---

## Data Flow

### Installation Flow

```
User Input (wizard/CLI)
    |
    v
Config Model (Pydantic validation)
    |
    v
Compose Generator (Jinja2 render)
    |
    +---> docker-compose.yml (written to disk)
    +---> .env (written to disk)
    +---> recyclarr.yml (written to disk)
    |
    v
Directory Creator
    |
    +---> /data/downloads/complete/
    +---> /data/downloads/incomplete/
    +---> /data/media/tv/
    +---> /data/media/movies/
    +---> /config/{service}/ (for each service)
    |
    v
Deploy Manager
    |
    +---> docker compose up -d
    |
    v
Health Checker (parallel polling)
    |
    +---> All services healthy? YES -->
    |                                  |
    v                                  v
    TIMEOUT (report errors)     API Key Extractor
                                       |
                                       v
                                Auto-Configurator (wiring sequence)
                                       |
                                       v
                                Final Health Check
                                       |
                                       v
                                Report to User
```

### Auto-Configuration Data Flow

```
config.xml files ──> API Key Extractor ──> Map<ServiceName, ApiKey>
                                                    |
                              +---------------------+
                              |
                              v
                     Wiring Orchestrator
                              |
        +----------+----------+----------+---------+
        |          |          |          |         |
        v          v          v          v         v
   Prowlarr    Sonarr     Radarr    Jellyseerr  Bazarr
   Client      Client     Client    Client      Client
        |          |          |          |         |
        v          v          v          v         v
   POST /api   POST /api  POST /api  POST /api  PATCH /api
   /v1/apps    /v3/dlcli  /v3/dlcli  /v1/sett   /system/
               /v3/root   /v3/root   /jellyfin   settings
```

### Key Data Flows

1. **Config propagation**: User choices flow through Pydantic model to Jinja2 templates to Docker Compose to running containers.
2. **API key extraction**: config.xml files on mounted volumes flow into a key map used by all API clients.
3. **Cross-service wiring**: API keys from one service (e.g., Sonarr) are sent TO another service (e.g., Prowlarr, Jellyseerr) to establish connections.
4. **VPN flag propagation**: The `vpn.enabled` flag changes qBittorrent's network mode, port mappings, and the hostname used by all download client configurations.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Docker daemon | Subprocess (`docker compose` CLI) | Needs socket access for web wizard container |
| Gluetun VPN | Env vars for provider config | 60+ providers, each with different env vars |
| TRaSH Guides | Via Recyclarr (fetches from GitHub) | Recyclarr handles the TRaSH API, not ARRmada |
| Container images | Docker Hub / GHCR / LSCR | linuxserver.io images preferred for consistency |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| CLI/Web <-> Core Engine | Python function calls | Same process, direct calls |
| Core Engine <-> Docker | Subprocess (`docker compose`) | Could use Docker SDK, but subprocess is simpler |
| Auto-Configurator <-> Services | HTTP (httpx async) | JSON REST APIs, each service on its own port |
| Compose Gen <-> Filesystem | File I/O (Jinja2 render) | Writes to user-specified install path |

---

## Suggested Build Order (Roadmap Implications)

Based on the dependency analysis above:

### Phase 1: Foundation (no API calls)
1. Pydantic config model
2. Jinja2 compose templates (Tier 1 services)
3. .env generation
4. Directory structure creation
5. Docker compose up/down/status via subprocess
6. Basic CLI (even just argparse, before Textual)

**Rationale:** Get containers running first. This alone is an Ezarr-equivalent.

### Phase 2: Auto-Configuration (the differentiator)
1. API key extraction (config.xml parser)
2. Health checker (wait for services)
3. Base API client class
4. qBittorrent auth (password extraction from logs or env var)
5. Sonarr client (download client + root folder)
6. Radarr client (download client + root folder)
7. Prowlarr client (add applications)
8. Wiring orchestrator (sequence above)

**Rationale:** This is the killer feature. Build it before the fancy UI. A working `arrmada configure` command proves the core value.

### Phase 3: Extended Wiring
1. Recyclarr config generation + invocation
2. Jellyseerr setup flow (the hardest -- first-run wizard automation)
3. Bazarr configuration
4. Jellyfin first-run automation

**Rationale:** Jellyseerr and Jellyfin first-run are the riskiest. Isolate them into a separate phase so Phase 2 can ship without them if needed.

### Phase 4: CLI Polish (Textual TUI)
1. Service selection screen
2. Path configuration
3. VPN provider selection
4. Deploy progress with live output
5. Status dashboard

### Phase 5: Web Wizard
1. FastAPI backend
2. Wizard steps as API endpoints
3. WebSocket for live progress
4. Static frontend

### Phase 6: Tier 2/3 Services
1. Gluetun VPN compose template
2. Conditional hostname logic (gluetun vs qbittorrent)
3. Lidarr, Readarr, Huntarr clients
4. Tdarr, Byparr compose templates

---

## Scalability Considerations

ARRmada is a single-user installer tool, not a multi-tenant SaaS. "Scalability" means:

| Concern | Consideration |
|---------|---------------|
| Number of services | Tier 3 = ~15 containers. All run on one host. Docker Compose handles this fine. |
| API call parallelism | `asyncio.TaskGroup` with httpx async. 10-15 concurrent API calls is trivial. |
| Template complexity | Single Jinja2 template with includes scales to 20+ services without issues. |
| Config model growth | Pydantic handles nested models well. Add new services as optional fields. |
| Multi-host deployment | Out of scope for V1. If ever needed, Docker Swarm or separate compose files per host. |

---

## Sources

- Servarr wiki: `https://wiki.servarr.com/` (Sonarr, Radarr, Prowlarr documentation) -- MEDIUM confidence (from training data, not verified live)
- Swagger UI on each service: `http://{service}:{port}/swagger` -- must verify at implementation time
- Gluetun docs: `https://github.com/qdm12/gluetun-wiki` -- MEDIUM confidence
- Recyclarr docs: `https://recyclarr.dev/` -- MEDIUM confidence
- Jellyseerr: `https://github.com/Fallenbagel/jellyseerr` -- LOW confidence (API least documented)
- Bazarr: `https://github.com/morpheus65535/bazarr` -- LOW confidence (API least documented)
- TRaSH Guides: `https://trash-guides.info/` -- HIGH confidence (well-known, stable)
- Docker network_mode documentation: `https://docs.docker.com/compose/networking/` -- HIGH confidence
- ARRmada project plan: local file `ARRmada_PROJECT_PLAN.md` -- authoritative
- ARR ecosystem analysis: local file `ARR_Ecosysteme_Alternatives_Detaillees.md` -- authoritative

---
*Architecture research for: ARRmada -- *ARR media stack installer and auto-configurator*
*Researched: 2026-02-22*
