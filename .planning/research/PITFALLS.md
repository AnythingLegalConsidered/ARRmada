# Pitfalls Research

**Domain:** *ARR media automation stack installer/auto-configurator
**Researched:** 2026-02-22
**Confidence:** MEDIUM (based on training data + project context; WebSearch/WebFetch/Bash unavailable for live verification)

> Note: External research tools were unavailable during this session. Findings are based on well-established community knowledge (TRaSH Guides, Servarr wiki, r/selfhosted, *ARR Discord patterns) present in training data. Confidence is MEDIUM rather than HIGH because live verification was not possible. All critical claims should be validated against current TRaSH Guides and Servarr wiki before implementation.

---

## Critical Pitfalls

### Pitfall 1: Broken Hardlinks from Separate Mount Points

**What goes wrong:**
When downloads (`/downloads`) and media (`/media`) are on different Docker volumes or different filesystem mount points, Sonarr/Radarr cannot create hardlinks. Instead they fall back to copy+delete, which doubles disk usage during import and takes significantly longer (minutes vs. instant for large files). Users see their disk fill up unexpectedly, and imports that should be instant take ages.

**Why it happens:**
Hardlinks only work within a single filesystem/mount point. Docker volumes are separate filesystems. Many tutorials (and even some Docker images' default configs) suggest separate volume mounts like `-v /downloads:/downloads` and `-v /media:/media`. This looks clean but breaks the one requirement hardlinks need: same filesystem.

**How to avoid:**
- Use a single `/data` root directory on the host, with `downloads/` and `media/` as subdirectories
- Map this single root into ALL containers: `-v /opt/arrmada/data:/data`
- In qBittorrent, set download path to `/data/downloads/complete`
- In Sonarr/Radarr, set root folders to `/data/media/tv`, `/data/media/movies`
- ARRmada's compose generator MUST enforce this layout -- it should not be configurable to separate volumes
- Validate at compose generation time: if the user's chosen data path is not a single partition, warn them

**Warning signs:**
- `docker exec sonarr ls -i /data/downloads/somefile` and `ls -i /data/media/somefile` show different inode numbers (hardlink = same inode)
- Sonarr/Radarr logs show "copy" operations instead of "hardlink" for imports
- Disk usage is roughly double what it should be
- Import operations take minutes instead of being instant

**Phase to address:** Phase 1 (Foundation) -- this is a compose template and directory structure decision. Getting it wrong means a full data restructure later.

---

### Pitfall 2: PUID/PGID Permission Chaos

**What goes wrong:**
Containers run as different users, creating files that other containers cannot read or modify. Sonarr downloads a file as UID 1000, but Jellyfin runs as UID 1001 and gets "Permission denied" when trying to read the media. Or qBittorrent creates downloads as root (UID 0) that Sonarr cannot move/hardlink.

**Why it happens:**
- LinuxServer.io images use PUID/PGID environment variables to set the internal user
- If these values differ between containers, each creates files with different ownership
- Default UIDs vary: some images default to root (0), others to 1000, others to 911
- Users running on VPS or multi-user systems often have UID 1000 for their user but Docker might run as a different user
- NAS devices (Synology, TrueNAS) have their own user/group schemes

**How to avoid:**
- Detect the current user's UID/GID at install time (`id -u`, `id -g`)
- Set identical PUID/PGID across ALL containers in the `.env` file
- Pre-create the data directory structure with correct ownership before first `docker compose up`
- Add a preflight check: verify all directories exist and have correct ownership
- For NAS users: document that PUID/PGID must match the NAS share owner

**Warning signs:**
- "Permission denied" errors in Sonarr/Radarr/Jellyfin logs
- Files owned by `root:root` or unexpected UID in the data directories
- `ls -la` shows mixed ownership in `/data/`
- Imports fail silently (file appears downloaded but never shows in library)

**Phase to address:** Phase 1 (Foundation) -- `.env` generation and directory creation. The installer MUST auto-detect UID/GID and validate permissions before deploying.

---

### Pitfall 3: API Wiring Race Conditions (Services Not Ready)

**What goes wrong:**
After `docker compose up`, the auto-configurator immediately tries to hit service APIs. But *ARR services need 10-60 seconds to boot (database migrations, initialization). API calls fail, wiring is incomplete, and the user has a partially configured stack. Worse: some services (Jellyseerr) require a manual first-run setup wizard before their API is usable.

**Why it happens:**
- Docker `healthcheck` is the right solution but many *ARR images have imprecise or missing healthchecks
- Service readiness != container running. A container can be "running" for 30+ seconds before its HTTP API responds
- *ARR apps on first boot: create database, run migrations, generate API keys in `config.xml` -- this takes time
- Jellyfin requires initial setup wizard (language, user creation, library paths) before API works
- Jellyseerr requires initialization (connect to Jellyfin, import users) before it accepts API config

**How to avoid:**
- Implement a robust health polling loop: HTTP GET to each service's `/api/v3/system/status` (Sonarr/Radarr) or `/ping` (Prowlarr) or `/api/v1/system/status` (Lidarr)
- Use exponential backoff with a maximum timeout (e.g., start at 2s, max 120s total)
- For Jellyfin: the auto-configurator must handle the first-run wizard programmatically via its API (`/Startup/*` endpoints)
- For Jellyseerr: similar first-run flow required (POST to `/api/v1/auth/jellyfin` then setup endpoints)
- Order matters: deploy and wait for base services (qBit, Jellyfin, *ARRs) THEN configure Jellyseerr/Bazarr which depend on them
- Parse `config.xml` for API keys ONLY after the service has fully started (the file may not exist or be incomplete during boot)

**Warning signs:**
- HTTP 502/503 responses during configuration
- Missing API keys (config.xml not yet generated)
- Partial wiring (Prowlarr has Sonarr but not Radarr, or vice versa)
- Jellyseerr shows "Setup required" when user first opens it

**Phase to address:** Phase 2 (Auto-Configuration) -- this is the core challenge of the auto-wiring engine. Needs careful implementation with retry logic, dependency ordering, and per-service readiness checks.

---

### Pitfall 4: Gluetun VPN Networking Breaks Inter-Container Communication

**What goes wrong:**
When qBittorrent uses `network_mode: "service:gluetun"`, it joins Gluetun's network namespace. This means qBittorrent no longer has its own IP on the Docker network. Other containers (Sonarr, Radarr) that try to reach qBittorrent at `http://qbittorrent:8080` get connection refused -- because qBittorrent's hostname no longer resolves on the Docker bridge network. The entire download pipeline breaks silently.

**Why it happens:**
- `network_mode: "service:gluetun"` makes qBittorrent share Gluetun's network stack entirely
- qBittorrent's ports must be exposed via Gluetun's container, not its own
- The hostname `qbittorrent` stops working on the Docker network; must use `gluetun` instead
- Sonarr/Radarr download client config must point to `gluetun:8080` (not `qbittorrent:8080`)
- If Gluetun crashes or the VPN connection drops, qBittorrent loses ALL network access (including to other containers)

**How to avoid:**
- When VPN is enabled, the compose template must:
  - Move qBittorrent's port mappings to Gluetun's container definition
  - Set `network_mode: "service:gluetun"` on qBittorrent
  - Remove qBittorrent from explicit Docker networks (it inherits Gluetun's)
  - Configure Gluetun to expose qBittorrent's web UI port
- Auto-wiring must use `gluetun` (not `qbittorrent`) as the download client hostname in Sonarr/Radarr/Lidarr
- Add Gluetun health check: `wget -q -O /dev/null https://ipinfo.io` or check VPN interface
- Implement kill switch verification: if VPN drops, qBittorrent should NOT fall back to unprotected traffic
- Gluetun environment variables differ per VPN provider -- need a provider-specific config mapper

**Warning signs:**
- Sonarr/Radarr show "Connection refused" for download client
- qBittorrent web UI unreachable from host
- `docker logs gluetun` shows VPN authentication failures
- Downloads work initially but stop after VPN reconnection

**Phase to address:** Phase 4 (Advanced / Tier 3) -- VPN integration. Needs dedicated compose template variant and wiring logic branch. Should be one of the last features due to complexity.

---

### Pitfall 5: Jellyfin/Jellyseerr First-Run Wizard Blocks Automation

**What goes wrong:**
Jellyfin and Jellyseerr both have mandatory first-run setup wizards. Until these are completed, their APIs reject most configuration calls. An auto-configurator that skips or doesn't handle these wizards leaves the user with services that appear "broken" -- they must manually complete setup before anything works. This defeats the entire value proposition of ARRmada.

**Why it happens:**
- Jellyfin's first run requires: language selection, creating an admin user, adding media libraries, configuring metadata sources, setting up remote access preferences
- Jellyseerr's first run requires: selecting server type (Jellyfin/Plex), entering Jellyfin URL + credentials, importing users, connecting Sonarr/Radarr
- These are stateful multi-step wizards, not simple API calls
- Most *ARR installer projects skip this problem entirely ("now go configure Jellyfin manually")

**How to avoid:**
- Jellyfin: use its Startup API endpoints to complete the wizard programmatically:
  - `POST /Startup/Configuration` (set metadata language, etc.)
  - `POST /Startup/User` (create admin user with user-provided credentials)
  - `POST /Library/VirtualFolders` (add media libraries: Movies at `/data/media/movies`, TV at `/data/media/tv`)
  - `POST /Startup/Complete`
- Jellyseerr: use its setup API:
  - `POST /api/v1/auth/jellyfin` (authenticate with Jellyfin)
  - `POST /api/v1/settings/jellyfin` (configure Jellyfin connection)
  - Import libraries and users
  - Add Sonarr/Radarr via settings API
- Collect required credentials (Jellyfin admin user/pass) during the wizard/CLI phase BEFORE deploying
- Store credentials securely in ARRmada config for the auto-configuration phase

**Warning signs:**
- Jellyfin shows "Welcome to Jellyfin!" setup page instead of login
- Jellyseerr shows "Getting Started" instead of the request interface
- API calls return 401 or redirect to setup endpoints
- Users report "services aren't connected" after deployment

**Phase to address:** Phase 2 (Auto-Configuration) -- this is an essential part of the auto-wiring engine. Without it, "zero manual configuration" is a lie.

---

### Pitfall 6: API Key Extraction Timing and Format Fragility

**What goes wrong:**
ARRmada reads API keys from `config.xml` files that *ARR services generate on first boot. But the file might not exist yet, might be partially written, or the XML format might change between versions. The configurator reads an empty/corrupt API key and all subsequent wiring silently fails.

**Why it happens:**
- *ARR services write `config.xml` during their first boot sequence
- If you read too early, the file doesn't exist or is incomplete
- Different *ARR versions have slightly different config.xml schemas
- Some services (qBittorrent) don't use config.xml at all -- they use different config formats
- Prowlarr v2+ may store some config differently than Prowlarr v1
- File encoding issues on some systems

**How to avoid:**
- NEVER read config.xml until the service's health endpoint responds 200
- Use the API itself to get API keys when possible: some services expose their key via an endpoint after first boot
- For Sonarr/Radarr/Prowlarr: parse config.xml with proper XML parser (not regex), look for `<ApiKey>` tag
- For qBittorrent: the API doesn't require an API key, it uses session-based auth (login endpoint)
- Implement fallback: if config.xml read fails, wait and retry; if API key is empty string, that's a failure
- Pin minimum supported versions of each *ARR service in documentation
- Add version detection: check `/api/v3/system/status` to know which API version to use

**Warning signs:**
- Empty or null API keys in ARRmada's stored configuration
- "Unauthorized" (401) responses when wiring services
- Prowlarr shows "0 Applications" after configuration
- Config.xml exists but has no `<ApiKey>` element

**Phase to address:** Phase 2 (Auto-Configuration) -- API key extraction is the first step of the wiring engine. Must be bulletproof.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Hardcoded service versions in compose templates | Simple, works now | Every *ARR update might break assumptions; users get outdated images | Never -- use `:latest` or let user pin versions |
| Hardcoded API endpoints (e.g., `/api/v3/`) | Quick implementation | Sonarr v4/v5 or Prowlarr v2 changes API version | MVP only -- add version detection in Phase 2 |
| Storing user passwords in plaintext YAML | Simple config format | Security incident waiting to happen | Never -- at minimum use restricted file permissions (600), ideally hash or don't store |
| Skipping Jellyfin first-run wizard | Faster to ship Phase 2 | Breaks the "zero config" promise, #1 user complaint guaranteed | Never -- it's the core value prop |
| Single retry for API calls | Quick to implement | Transient failures (service restart, DNS resolution delay) break entire wiring | MVP only -- implement proper retry with backoff by Phase 2 |
| No idempotency in configurator | First implementation is simpler | Running `arrmada configure` twice creates duplicate download clients, root folders, indexer connections | Never -- must be idempotent from day one |
| Bash `curl \| sh` installer without verification | Easy distribution | Supply chain attack vector; no integrity verification | Acceptable if script is human-readable and hosted on verified repo |

## Integration Gotchas

Common mistakes when connecting *ARR services together.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Prowlarr -> Sonarr/Radarr | Adding as "Indexer" instead of "Application" | Prowlarr uses "Applications" to sync indexers TO Sonarr/Radarr. Don't manually add indexers in Sonarr/Radarr -- let Prowlarr push them. |
| qBittorrent -> Sonarr/Radarr | Wrong category mapping | Each *ARR needs its own download category (e.g., `sonarr`, `radarr`). Without categories, Sonarr imports Radarr's downloads and vice versa. |
| qBittorrent download path | Using default download path `/downloads` | Must match container's mapped path: `/data/downloads/complete`. If paths don't match between what qBit sees and what Sonarr sees, imports fail with "file not found". |
| Jellyseerr -> Jellyfin | Using `localhost` as Jellyfin URL | In Docker, `localhost` refers to the container itself. Must use Docker service name (`http://jellyfin:8096`) or the Docker network IP. |
| Recyclarr -> Sonarr/Radarr | Running before quality profiles exist | Recyclarr syncs custom formats and quality profiles. On a fresh install, profiles may not exist yet. Must let Sonarr/Radarr fully initialize first. |
| Bazarr -> Sonarr/Radarr | Forgetting to enable the Bazarr provider in Sonarr/Radarr | Sonarr/Radarr need to have subtitle importing enabled; Bazarr's connection alone isn't enough. |
| Cross-service hostnames | Using `127.0.0.1` or host IP | Always use Docker service names (`sonarr`, `radarr`, `prowlarr`) for inter-container communication. Host IPs break on network changes; localhost doesn't work cross-container. |
| Prowlarr sync test | Not testing sync after adding application | After adding Sonarr/Radarr as Prowlarr applications, trigger a sync and verify indexers appear in Sonarr/Radarr. Silent sync failures are common. |

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| All services on same Docker network | Works fine initially | Use dedicated networks: one for *ARR intercommunication, one for VPN, one for frontend (Jellyfin/Jellyseerr exposed to LAN) | 15+ containers competing for DNS resolution |
| No resource limits on containers | Simple compose file | Add `mem_limit` and `cpus` constraints. Sonarr/Radarr can spike to 2-4GB RAM during large library scans | When library exceeds ~5000 items or Tdarr starts transcoding |
| Polling all service APIs simultaneously during health check | Fast feedback | Stagger health checks; don't DDoS your own services on startup | First boot on low-power hardware (mini-PCs, Raspberry Pi) |
| Recyclarr running on every boot | "Always up to date" | Run Recyclarr on a schedule (daily/weekly cron), not on every container restart | When user restarts stack frequently; Recyclarr hits API rate limits |
| Single download category for all *ARRs | Simpler config | Separate categories (`sonarr`, `radarr`, `lidarr`) prevent cross-import chaos | As soon as multiple *ARRs are active |
| No log rotation | Logs work fine | Configure Docker logging driver with max-size/max-file: `json-file` with `max-size: 10m, max-file: 3` | After weeks of operation, logs consume GBs |

## Security Mistakes

Domain-specific security issues for *ARR media stacks.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Exposing *ARR web UIs to the internet without auth | Anyone can access your Sonarr/Radarr, add content, modify settings, steal API keys | V1: bind ports to `127.0.0.1` or LAN subnet only. V2: add reverse proxy + auth. Generate a clear warning if ports are bound to `0.0.0.0`. |
| Docker socket mount (`/var/run/docker.sock`) in wizard container | Full root access to the host via Docker API | If the web wizard needs Docker access, use a Docker socket proxy (like Tecnativa/docker-socket-proxy) with read-only access. Never mount the raw socket. |
| API keys in environment variables visible via `docker inspect` | Anyone with Docker access sees all API keys | Use Docker secrets or file-based secrets where possible. At minimum, restrict `.env` file permissions to `600`. |
| Default qBittorrent credentials left unchanged | Anyone on the network can access qBit, modify downloads, potentially execute code | Auto-generate a random strong password during setup and configure it via qBittorrent's API. Store in ARRmada config. |
| VPN credentials in plaintext `.env` | VPN account compromise | Store VPN credentials in a separate file with `600` permissions, referenced via `env_file` in compose. Warn user clearly. |
| Running containers as root (PUID=0) | Container escape = full host compromise | Always detect and use the current non-root user's UID/GID. Refuse to set PUID=0 unless user explicitly overrides. |
| No HTTPS for Jellyfin on LAN | Credentials sent in cleartext on local network | Acceptable for V1 (LAN only), but document the risk. V2 reverse proxy solves this. |

## UX Pitfalls

Common user experience mistakes in *ARR installer tools.

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Dumping users into a wall of text after install | Overwhelmed, don't know what to do next | Show a clear status dashboard: checkmark for each service, direct URL links, "Your stack is ready" message |
| No progress indication during deployment | User thinks it's stuck, Ctrl+C's mid-deploy | Real-time progress: "Pulling image 3/7... Starting Sonarr... Waiting for health check... Wiring Prowlarr..." |
| Error messages show raw API responses | Cryptic JSON errors confuse beginners | Translate errors: "Sonarr rejected the download client configuration" with suggested fix, not raw 400 response body |
| Requiring user to know their PUID/PGID | Most beginners have no idea what this is | Auto-detect with `id -u` / `id -g`. Only ask if running as root (which user should they run as?) |
| All-or-nothing deployment (no partial recovery) | One failed service = restart everything | Deploy incrementally: base services first, then dependent services. If Bazarr fails, Sonarr/Radarr still work fine. |
| No way to re-run configuration only | User changed something manually, wants to re-wire | `arrmada configure` must be a standalone command that re-detects state and fixes wiring without redeploying containers |
| Assuming a fixed install path | Conflicts with existing setups, NAS conventions | Let user choose the data path during wizard. Default to `/opt/arrmada` but support any path. Validate the path is writable and on a single filesystem. |
| No post-install validation | User assumes everything works, discovers broken wiring days later | After configuration, run automated tests: can Sonarr see Prowlarr's indexers? Can Radarr reach qBittorrent? Is Jellyfin's library path valid? Report results. |

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Docker Compose generated:** Often missing `restart: unless-stopped` -- containers don't survive host reboot. Verify all services have restart policy.
- [ ] **Prowlarr wired to Sonarr/Radarr:** Often missing actual indexer sync -- adding the application is step 1, triggering a sync is step 2. Verify indexers appear in Sonarr/Radarr's indexer list.
- [ ] **qBittorrent configured as download client:** Often missing category setup -- without per-*ARR categories, cross-import happens. Verify each *ARR has its own category in qBit.
- [ ] **Media paths configured:** Often missing the actual directory creation inside the container's view -- paths are configured in Sonarr but the directories don't exist. Verify paths exist and are writable from inside each container.
- [ ] **Jellyfin library added:** Often missing library scan trigger -- library path is added but Jellyfin hasn't scanned it. Trigger an initial scan after adding the library.
- [ ] **VPN working:** Often missing kill switch verification -- VPN connects but if it drops, traffic leaks. Verify: disconnect VPN, confirm qBittorrent loses connectivity.
- [ ] **Recyclarr applied:** Often missing quality profile assignment -- custom formats are synced but no quality profile actually uses them. Verify the default quality profile includes the synced custom formats with correct scores.
- [ ] **Permissions correct:** Often missing recursive ownership -- top-level directory has correct ownership but subdirectories created by different containers don't. Verify with `find /data -not -user $PUID` returns nothing.
- [ ] **Download client path mapping:** Often missing the mapping between what qBittorrent sees as the download path and what Sonarr/Radarr see -- paths must be identical. Verify qBit's save path matches Sonarr/Radarr's "Remote Path Mapping" or (better) that no mapping is needed because all share `/data`.

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Broken hardlinks (separate mounts) | HIGH | Stop stack. Restructure directories to single `/data` root. Move all existing media. Update compose volumes. Regenerate compose. Restart and reconfigure paths in all *ARRs. Existing library metadata preserved if DB volumes unchanged. |
| Wrong PUID/PGID | MEDIUM | Stop stack. `chown -R $PUID:$PGID /data/`. Update `.env`. Restart. No data loss, but recursive chown on large libraries is slow. |
| Partial API wiring | LOW | Run `arrmada configure` again (must be idempotent). Or manually fix via each service's web UI. |
| Gluetun VPN not connecting | LOW | Check `docker logs gluetun` for auth errors. Fix env vars. Restart gluetun container only. Other services unaffected. |
| Jellyfin first-run not completed | MEDIUM | Delete Jellyfin's config volume (`docker volume rm`), restart Jellyfin, re-run auto-configuration. User loses any manual Jellyfin settings. |
| Duplicate download clients from non-idempotent config | LOW | Access Sonarr/Radarr UI, remove duplicate entries. Or reset *ARR config and re-run `arrmada configure`. |
| Port conflicts | LOW | Stop conflicting service. Change port in `.env`. Regenerate compose. Restart. |
| Corrupted config.xml API key read | LOW | Restart the *ARR service (it regenerates config.xml). Re-run `arrmada configure`. |
| Docker socket exposed in wizard | HIGH | Immediately stop wizard container. Audit for unauthorized containers or images. Remove socket mount. Use socket proxy instead. |

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Broken hardlinks (separate mounts) | Phase 1 (Foundation) | Integration test: create file in downloads dir, hardlink to media dir, verify same inode |
| PUID/PGID permission chaos | Phase 1 (Foundation) | Preflight check: verify all data dirs have correct ownership before deploying |
| API wiring race conditions | Phase 2 (Auto-Configuration) | Health check returns 200 for all services before any wiring begins; retry logic with backoff |
| Gluetun networking breaks qBit | Phase 4 (Advanced) | When VPN enabled: verify qBit reachable via `gluetun` hostname; verify Sonarr can connect to download client |
| Jellyfin/Jellyseerr first-run blocks | Phase 2 (Auto-Configuration) | After auto-config, Jellyfin login page loads (not setup wizard); Jellyseerr shows request UI (not setup) |
| API key extraction fragility | Phase 2 (Auto-Configuration) | Unit tests for config.xml parsing; integration test with fresh *ARR boot |
| Non-idempotent configurator | Phase 2 (Auto-Configuration) | Run `arrmada configure` twice; verify no duplicate entries in any service |
| Port conflicts | Phase 1 (Foundation) | Preflight check: scan all configured ports, warn on conflicts BEFORE deploying |
| Security (exposed ports, docker socket) | Phase 1 (Foundation) + Phase 3 (Web Wizard) | Default port bindings use `127.0.0.1:`; wizard uses socket proxy |
| No progress indication | Phase 1 (Foundation) | Every long-running operation streams status updates to the TUI/web UI |
| Cross-platform path issues | Phase 1 (Foundation) | Detect OS/environment; reject unsupported platforms with clear message |
| Recyclarr overwrites | Phase 2 (Auto-Configuration) | Recyclarr runs with default TRaSH templates; user customization docs clearly state which profiles are managed |
| Default qBit credentials | Phase 2 (Auto-Configuration) | Auto-generate password; verify login works with generated credentials |

## Platform-Specific Pitfalls

### Linux (Primary Target)

| Issue | Details | Mitigation |
|-------|---------|------------|
| Docker not installed | Many mini-PC / VPS ship without Docker | One-liner script must detect and install Docker + Docker Compose |
| Snap Docker vs. official Docker | Ubuntu ships Snap Docker which has different volume mount behavior | Detect Snap Docker and warn/block. Install official Docker from `get.docker.com`. |
| AppArmor/SELinux blocking mounts | Security modules can prevent container volume access | Detect and warn. Provide documented workarounds. |
| systemd-resolved port 53 conflict | On Ubuntu 22.04+, systemd-resolved uses port 53, conflicting with any DNS container | Not relevant for V1 scope (no DNS container), but document for future |
| Filesystem type matters | ext4 supports hardlinks. ZFS supports hardlinks within dataset. BTRFS uses copy-on-write (reflinks, not hardlinks). | Detect filesystem type on data path. Warn if using BTRFS (hardlinks work but don't save space due to CoW -- reflinks are better). |

### WSL2 (Best-Effort)

| Issue | Details | Mitigation |
|-------|---------|------------|
| Filesystem performance | Accessing Windows paths (`/mnt/c/`) from WSL2 is 10-50x slower than native Linux paths | Warn if data path starts with `/mnt/`. Recommend using WSL2 native filesystem (`~/`). |
| Docker Desktop vs. native Docker | WSL2 can use Docker Desktop (Windows) or native Linux Docker inside WSL | Detect which is in use. Docker Desktop has different volume mount semantics. |
| PUID/PGID mapping | WSL2 maps Windows users differently; UID/GID may not behave as expected | Document WSL2-specific UID behavior. Test with `id` command. |
| Path format differences | Windows backslashes vs. Linux forward slashes in mixed environments | Always use POSIX paths inside ARRmada. Never accept or generate Windows paths. |

## Recyclarr-Specific Pitfalls

| Pitfall | What Happens | Prevention |
|---------|-------------|------------|
| Running Recyclarr before Sonarr/Radarr are fully initialized | Recyclarr fails to connect or creates incomplete profiles | Health-check Sonarr/Radarr APIs before running Recyclarr |
| User manually configures quality profiles, then Recyclarr overwrites them | User's custom settings are replaced by TRaSH defaults | Document clearly: "Recyclarr manages these profiles. Customize via `recyclarr.yml`, not the UI." Alternatively, let user opt-out of Recyclarr. |
| Wrong Recyclarr template for user's needs | User gets 4K profiles but has a 1080p setup, or anime profiles but watches only Western content | Ask during wizard: "What is your maximum quality?" and "Do you watch anime?" Map answers to correct Recyclarr templates. |
| Recyclarr API key mismatch after *ARR reinstall | *ARR regenerates API key; Recyclarr config has the old one | `arrmada configure` must update Recyclarr's config with current API keys from *ARR instances |
| Custom formats synced but not assigned to quality profile | Formats exist but have no effect because no profile uses them | Recyclarr templates should include both custom format creation AND quality profile assignment. Verify post-sync that the default profile has scored CFs. |

## Sources

- TRaSH Guides (https://trash-guides.info/) -- hardlinks, folder structure, Docker setup [MEDIUM confidence -- from training data, not live-verified]
- Servarr Wiki Docker Guide (https://wiki.servarr.com/docker-guide) -- PUID/PGID, permissions, volume mapping [MEDIUM confidence]
- Gluetun documentation (https://github.com/qdm12/gluetun) -- VPN container networking [MEDIUM confidence]
- Recyclarr documentation (https://recyclarr.dev/) -- configuration, templates, idempotency [MEDIUM confidence]
- Jellyfin API documentation (https://api.jellyfin.org/) -- startup/setup endpoints [LOW confidence -- API endpoints from training data, verify against current docs]
- Jellyseerr API (https://github.com/Fallenbagel/jellyseerr) -- setup flow [LOW confidence -- verify current API]
- r/selfhosted community patterns -- common deployment mistakes [MEDIUM confidence -- well-established patterns]
- ARRmada project plan (local) -- architecture decisions, tier structure [HIGH confidence -- read directly]
- LinuxServer.io documentation -- PUID/PGID convention [MEDIUM confidence]

---
*Pitfalls research for: *ARR media automation stack installer (ARRmada)*
*Researched: 2026-02-22*
