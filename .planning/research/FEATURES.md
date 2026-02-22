# Feature Research

**Domain:** *ARR media stack installer and auto-configurator
**Researched:** 2026-02-22
**Confidence:** MEDIUM-HIGH (based on local competitor research docs from Feb 2026 + training data)

## Competitor Landscape Summary

Six competitors analyzed across two categories:

**Deployers** (deploy containers, stop there):
- **Ezarr** (~1027 stars): Bash + docker-compose, TRaSH-compliant paths, zero auto-configuration
- **Deployrr** (~719 stars): Bash, 140+ apps, freemium model, includes Traefik/auth, no auto-wiring
- **Cloud Seeder** (~188 stars): Partial auto-wiring (qBittorrent only), small scope, small community

**Configurators** (configure existing services, don't deploy):
- **Buildarr** (Python): Config-as-code, idempotent, TRaSH integration, plugin architecture, *ARR-only scope
- **Flemmarr** (~336 stars): Raw API push via YAML, no abstraction, poor docs, not idempotent
- **Recyclarr**: Official TRaSH Guides tool, quality profiles only, excellent but narrow scope

**Hybrid (NixOS-only):**
- **Nixflix** (Feb 2026, alpha): Declarative API auto-wiring + TRaSH + deployment, but NixOS-only

**Key gap:** No open-source Docker-based tool does deploy + configure + auto-wire in one flow.

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete or broken.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Docker Compose generation** | Every deployer does this (Ezarr, Deployrr). Users expect a compose file they can inspect. | MEDIUM | Jinja2 templating with conditionals per service. Must support optional services cleanly. |
| **.env generation** | Standard practice. PUID, PGID, TZ, paths, ports must be auto-detected or prompted. | LOW | Detect UID/GID from current user, TZ from system, prompt for paths only. |
| **Directory structure creation** | Ezarr does this. Wrong permissions = services fail silently. | LOW | Must follow TRaSH Guides single `/data` root. Set ownership correctly (PUID:PGID). |
| **Service selection** | Deployrr has 140+ apps, Ezarr has ~16. Users expect to choose what to install. | LOW | Tier-based (Essential/Extended/Advanced) is better than a la carte for beginners. |
| **Deploy command** | Literally `docker compose up -d`. Every tool does this. | LOW | Wrap compose commands, handle errors, show progress. |
| **Status/health check** | Users need to know if containers are running. Basic monitoring. | LOW | Poll container status via Docker SDK or compose ps. Show port accessibility. |
| **Clean teardown** | Users need to undo. Ezarr lacks this, which is a pain point. | LOW | `docker compose down -v` with confirmation. Option to preserve data. |
| **Hardlink-friendly paths** | TRaSH Guides standard. Ezarr follows this. Users in 2026 expect it. | LOW | Single `/data` root mount with `downloads/` and `media/` subdirs. Non-negotiable. |
| **VPN support (optional)** | Gluetun is the standard. Most guides include it. Users expect a VPN option. | MEDIUM | Gluetun integration with qBit network_mode. Must prompt for provider + credentials. 60+ providers. |
| **Recyclarr/TRaSH Guides** | Community gold standard for quality profiles. Recyclarr is the official tool. | LOW | Bundle Recyclarr as a service, generate its config with correct API keys and sensible template defaults. |
| **Idempotent operations** | Buildarr's key strength. Users re-run commands and expect no breakage. | MEDIUM | All API calls must check current state before modifying. Skip if already configured. |
| **Linux support (Debian/Ubuntu)** | Primary target for every homelab tool. | LOW | Primary platform. Must test on Debian 12 and Ubuntu 22.04+. |

### Differentiators (Competitive Advantage)

Features that set ARRmada apart. No competitor offers all of these together.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Full auto-wiring via APIs** | THE killer feature. After `docker compose up`, ARRmada configures all service interconnections automatically. Ezarr stops at deployment, Buildarr requires manual YAML writing. Only Cloud Seeder does partial wiring (qBit only). Only Nixflix does full wiring but is NixOS-only. | HIGH | Must implement API clients for every service: Prowlarr apps, Sonarr/Radarr download clients + root folders, Jellyseerr connections, Bazarr connections. This is the hardest and most valuable feature. |
| **API key auto-extraction** | Users never touch an API key. ARRmada reads them from config.xml after first boot. No other Docker-based tool does this. | MEDIUM | Parse XML config files from mounted volumes. Requires waiting for service first-boot to generate keys. Timing-sensitive. |
| **Dual interface (TUI + Web wizard)** | No competitor has both. Deployrr has CLI-only, corelab.tech has web-only. Having both serves power users (SSH into server, run TUI) and beginners (open browser, follow wizard). | HIGH | Two frontends sharing the same core engine. TUI via Textual, Web via FastAPI. Must keep feature parity between them. |
| **One-liner install** | `curl ... \| bash` that handles Docker installation if absent, then launches wizard. Reduces friction to absolute minimum. Deployrr has something similar but it's freemium. | MEDIUM | install.sh must detect OS, install Docker if missing, pull ARRmada image, launch. Must be robust across distros. |
| **Tiered service profiles** | Not a la carte chaos, not all-or-nothing. Essential (6 services) / Extended (11) / Advanced (14). Reduces decision fatigue for beginners while giving power users options. | LOW | Map services to tiers. Allow tier selection + individual service toggles within tiers. |
| **Service health polling with readiness wait** | Before auto-wiring, wait for all services to be healthy and API-responsive. No competitor handles the timing gap between container start and API readiness. | MEDIUM | Poll health endpoints with exponential backoff. Timeout after N minutes with clear error. Critical for auto-wiring reliability. |
| **Zero-config deployment** | Sensible defaults for everything. A user can accept all defaults and get a working stack. Ezarr requires manual .env editing. Buildarr requires YAML authoring. | MEDIUM | Auto-detect PUID/PGID/TZ, use standard ports, default paths. Only truly required input: where to store data. |
| **Self-removing web wizard** | Web wizard container deploys the stack then removes itself. Clean, no lingering management container. | LOW | After successful deployment, `docker rm` self. Leave only the media stack running. |
| **Post-install summary with URLs** | After deployment, show a clear summary: service names, URLs, default credentials. No competitor does this well. | LOW | Print/display table of `service -> http://host:port -> default login`. Massive UX win. |
| **Config export (arrmada.yml)** | Save all user choices to a file. Enables reproducible deployments, sharing configs, and re-running on new hardware. Inspired by Buildarr's config-as-code approach. | LOW | Pydantic model serialized to YAML. Can be used for `arrmada deploy --config arrmada.yml`. |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems. Deliberately NOT building these.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Reverse proxy (Traefik/Caddy)** | Remote access, SSL, custom domains. Deployrr includes this. | Massively increases complexity. DNS, certificates, auth, port forwarding -- each introduces failure modes. V1 targets local network. Deployrr is proof: it becomes a full homelab framework, not an ARR installer. | Document how to add Traefik manually post-install. Consider V2 addon. |
| **Authentication layer (Authelia/Authentik)** | Security for remote access. | Requires reverse proxy first. Overkill for local network. Turns ARRmada into Deployrr. | Individual service passwords. Document auth setup in docs. |
| **Built-in media player/streaming** | "Can ARRmada play my media?" | Jellyfin IS the media player. ARRmada deploys it. Building another player is insane scope creep. | Jellyfin deployment + auto-config. Point users to Jellyfin apps. |
| **Natural language downloads** | "Download The Batman 2022 in 4K" -- cool demo feature. | Requires LLM/NLP integration, TMDB search, ARR API orchestration. Massive scope for a V1 installer tool. | Defer to V2. Jellyseerr already provides a search-and-request UI. |
| **Package manager / app store** | "Let me add any Docker app through ARRmada." | Scope creep into Portainer/CasaOS territory. ARRmada is for the ARR stack, not a general homelab manager. | Fixed service catalog with tiers. Add new services via config, not a marketplace. |
| **Auto-updates with rollback** | "Keep my stack always up to date." | Unattended updates break things. ARR services sometimes have breaking changes between versions. Automatic rollback requires snapshot infrastructure. | `arrmada update` as a manual command. Pin image versions. Show available updates but don't auto-apply. |
| **Multi-server / distributed deployment** | "Deploy Sonarr on server A, Jellyfin on server B." | Networking between servers, remote Docker management, service discovery. Massive complexity increase for a niche use case. | V1 is single-host only. Document manual multi-server setup. |
| **Plex support** | Plex has a larger user base than Jellyfin. | Plex requires a Plex account (external dependency), has a freemium model, closed source. Jellyfin is fully open source and aligns with self-hosting philosophy. Supporting both doubles testing surface for media server integration. | Jellyfin-only for V1. Plex as V2 addition if demand is high. The project plan already uses Jellyseerr which supports both. |
| **Indexer management** | "Add indexers automatically to Prowlarr." | Indexers require user-specific accounts, API keys, invites. Can't be automated ethically or reliably. Each user has different indexer access. | Deploy Prowlarr with auto-wiring to *ARR apps. User adds their own indexers. Document recommended public indexers. |
| **Backup/restore in V1** | Data safety is important. | Adds significant complexity (what to backup, where to store, scheduling, restore logic). V1 scope is install, not operate. | Document manual backup (copy config/ dir). Add `arrmada backup` in V1.x. |

## Feature Dependencies

```
[Docker Compose generation]
    |-- requires --> [.env generation]
    |-- requires --> [Directory structure creation]
    |-- requires --> [Service selection / tier choice]

[Deploy command]
    |-- requires --> [Docker Compose generation]

[Service health polling]
    |-- requires --> [Deploy command]

[API key extraction]
    |-- requires --> [Service health polling] (services must be running + ready)

[Prowlarr auto-wiring]
    |-- requires --> [API key extraction]
    |-- requires --> [Service health polling]

[Download client auto-config (qBit -> *ARRs)]
    |-- requires --> [API key extraction]

[Root folder auto-setup]
    |-- requires --> [API key extraction]
    |-- requires --> [Directory structure creation]

[Jellyseerr auto-config]
    |-- requires --> [API key extraction] (Jellyfin, Sonarr, Radarr keys)
    |-- requires --> [Service health polling]

[Bazarr auto-config]
    |-- requires --> [API key extraction] (Sonarr, Radarr keys)

[Recyclarr integration]
    |-- requires --> [API key extraction] (Sonarr, Radarr keys)

[VPN integration (Gluetun)]
    |-- requires --> [Docker Compose generation] (network_mode on qBit)
    |-- independent of --> [Auto-wiring] (compose-level, not API-level)

[Web wizard]
    |-- requires --> [Core engine] (shares all logic with CLI)
    |-- enhances --> [Service selection]
    |-- enhances --> [Deploy command]

[TUI (Textual)]
    |-- requires --> [Core engine]
    |-- parallel with --> [Web wizard] (both are frontends)

[Status command]
    |-- requires --> [Deploy command]
    |-- enhanced by --> [Service health polling]

[Teardown command]
    |-- independent --> (just wraps docker compose down)

[Config export (arrmada.yml)]
    |-- requires --> [Service selection]
    |-- enhances --> [Deploy command] (deploy from config)

[Post-install summary]
    |-- requires --> [Auto-wiring complete]
    |-- requires --> [Service health polling]
```

### Dependency Notes

- **API key extraction requires health polling:** Services must be fully booted and have generated their config.xml before keys can be read. This is the most timing-sensitive dependency.
- **All auto-wiring requires API keys:** The entire auto-configuration chain starts with extracting API keys. If this fails, everything downstream fails. Must be rock-solid.
- **VPN is compose-level, not API-level:** Gluetun integration happens in the docker-compose template (network_mode), not via API calls. It's independent of the auto-wiring chain.
- **Web wizard and TUI are parallel frontends:** They share the core engine. Neither depends on the other. Build TUI first (simpler, no frontend build step), then web wizard.

## MVP Definition

### Launch With (v1.0)

Minimum viable product -- what proves the concept of "deploy + auto-wire."

- [x] **Docker Compose generation** (Tier 1 services: Prowlarr, Radarr, Sonarr, qBittorrent, Jellyfin, Jellyseerr) -- the baseline every competitor has
- [x] **.env generation** with auto-detected PUID/PGID/TZ -- removes the "edit .env manually" step from Ezarr
- [x] **Directory structure creation** with correct permissions and TRaSH-compliant paths -- table stakes
- [x] **Service selection** (Tier 1 default, optional Bazarr/Recyclarr/Gluetun) -- not all tiers, just enough choice
- [x] **Deploy command** (`arrmada deploy`) -- wraps docker compose
- [x] **Service health polling** with readiness wait -- prerequisite for auto-wiring
- [x] **API key auto-extraction** from config.xml -- the enabler
- [x] **Prowlarr auto-wiring** (add Sonarr + Radarr as applications) -- the differentiator starts here
- [x] **qBittorrent auto-config** as download client in Sonarr + Radarr -- core wiring
- [x] **Root folder auto-setup** (media paths in each *ARR) -- core wiring
- [x] **Jellyseerr auto-config** (connect Jellyfin + Sonarr + Radarr) -- completes the request flow
- [x] **Bazarr auto-config** (connect Sonarr + Radarr) -- simple wiring, high value
- [x] **Recyclarr config generation** (TRaSH Guides quality profiles) -- community expectation
- [x] **Gluetun VPN integration** (optional, compose-level) -- users expect VPN support
- [x] **Status command** (`arrmada status`) -- basic health check
- [x] **Teardown command** (`arrmada destroy`) -- clean removal
- [x] **Textual TUI** as primary interface -- ship one interface first
- [x] **Post-install summary** with service URLs and default credentials -- UX polish

### Add After Validation (v1.x)

Features to add once core deployment + auto-wiring is proven stable.

- [ ] **Web wizard (FastAPI)** -- add when TUI-based flow is proven; requires frontend work
- [ ] **One-liner install script** (install.sh with Docker auto-install) -- add when the tool itself is stable
- [ ] **Config export/import (arrmada.yml)** -- reproducible deployments, sharing configs
- [ ] **Update command** (`arrmada update`) -- pull new images, re-run auto-config
- [ ] **Tier 2 services** (Lidarr, Readarr, Huntarr) -- extend once Tier 1 wiring is solid
- [ ] **Docker image** (for web wizard deployment method) -- after FastAPI wizard is built
- [ ] **qBittorrent default config** (download paths, categories per *ARR) -- requires qBit API
- [ ] **Idempotent re-configuration** (`arrmada reconfigure`) -- re-wire without redeploying

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] **Tier 3 services** (Tdarr, Byparr) -- niche, can be added manually
- [ ] **Monitoring tier** (Homepage, Jellystat) -- nice-to-have dashboards
- [ ] **Reverse proxy addon** (Traefik/Caddy with SSL) -- remote access
- [ ] **Backup/restore** -- operational tooling beyond install scope
- [ ] **Multi-platform support** (macOS, Windows WSL2) -- after Linux is rock-solid
- [ ] **Plex support** -- alternate media server path
- [ ] **Natural language downloads** -- LLM-powered content requests
- [ ] **Plugin system** for community-contributed service definitions

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Docker Compose generation | HIGH | MEDIUM | P1 |
| .env generation | HIGH | LOW | P1 |
| Directory structure creation | HIGH | LOW | P1 |
| Service selection (tiers) | HIGH | LOW | P1 |
| Deploy command | HIGH | LOW | P1 |
| Service health polling | HIGH | MEDIUM | P1 |
| API key extraction | HIGH | MEDIUM | P1 |
| Prowlarr auto-wiring | HIGH | MEDIUM | P1 |
| qBittorrent auto-config | HIGH | MEDIUM | P1 |
| Root folder auto-setup | HIGH | LOW | P1 |
| Jellyseerr auto-config | HIGH | MEDIUM | P1 |
| Bazarr auto-config | MEDIUM | LOW | P1 |
| Recyclarr config generation | HIGH | LOW | P1 |
| Gluetun VPN integration | HIGH | MEDIUM | P1 |
| Status command | MEDIUM | LOW | P1 |
| Teardown command | MEDIUM | LOW | P1 |
| Textual TUI | HIGH | HIGH | P1 |
| Post-install summary | MEDIUM | LOW | P1 |
| Web wizard (FastAPI) | HIGH | HIGH | P2 |
| One-liner install script | HIGH | MEDIUM | P2 |
| Config export/import | MEDIUM | LOW | P2 |
| Update command | MEDIUM | MEDIUM | P2 |
| Tier 2 services wiring | MEDIUM | MEDIUM | P2 |
| Idempotent reconfiguration | MEDIUM | MEDIUM | P2 |
| qBittorrent advanced config | LOW | MEDIUM | P2 |
| Tier 3 services | LOW | MEDIUM | P3 |
| Monitoring tier | LOW | LOW | P3 |
| Reverse proxy addon | MEDIUM | HIGH | P3 |
| Backup/restore | MEDIUM | HIGH | P3 |
| Plex support | MEDIUM | HIGH | P3 |

**Priority key:**
- P1: Must have for launch -- proves the "deploy + auto-wire" concept
- P2: Should have, add after v1.0 validation
- P3: Nice to have, future consideration

## Competitor Feature Analysis

| Feature | Ezarr | Deployrr | Cloud Seeder | Buildarr | Flemmarr | ARRmada |
|---------|-------|----------|-------------|----------|----------|---------|
| **Deploys containers** | Bash + compose | Bash script | Script | No | No | Compose generation + deploy |
| **Service count** | ~16 | 140+ | ~6 | N/A (config only) | N/A (config only) | 9-14 (tiered) |
| **Auto-wires services** | No | No | qBit only | Partial (manual YAML) | No | Full auto (all services) |
| **API key handling** | Manual | Manual | Unknown | Manual (in YAML) | Manual | Auto-extracted from config.xml |
| **TRaSH Guides** | Paths only | No | No | Built-in | No | Recyclarr bundled + auto-configured |
| **VPN support** | No | Gluetun | No | No | No | Gluetun with one-click setup |
| **Interactive UI** | No | CLI menu | No | No | No | Textual TUI + Web wizard |
| **Idempotent** | N/A | No | Unknown | Yes | No | Yes (planned) |
| **Config-as-code** | No | No | No | Yes (YAML) | Yes (YAML) | Yes (arrmada.yml) |
| **Hardlink paths** | Yes | No | Unknown | N/A | N/A | Yes (default) |
| **Health checks** | No | Basic | Unknown | No | No | Full readiness polling |
| **Open source** | Yes | Freemium | Yes | Yes | Yes | Yes (MIT) |
| **OS support** | Linux | Ubuntu mainly | Linux | Any (Python) | Any (Docker) | Linux (Debian/Ubuntu primary) |
| **Directory setup** | Yes (root) | Yes | Unknown | No | No | Yes (auto, correct perms) |
| **Post-install guidance** | README | Blog posts | No | No | No | Auto-generated summary |

### Key Competitive Insights

1. **The deployment gap is real.** Ezarr has 1000+ stars precisely because "just getting containers running" is painful. But it stops at `docker compose up`. Users then spend hours reading wikis to wire services together.

2. **The configuration gap is real too.** Buildarr exists because manually configuring *ARR services via their UIs is tedious and error-prone. But Buildarr requires you to write YAML and already have services running.

3. **Nobody bridges both gaps.** This is ARRmada's core opportunity. The closest is Nixflix, but it's NixOS-only (excludes 95%+ of users).

4. **Cloud Seeder proves partial auto-wiring has demand** (188 stars for a small project), but its scope is too limited (no Jellyfin, no Jellyseerr, no TRaSH).

5. **Deployrr proves the market wants a guided experience**, but its freemium model and homelab-wide scope make it overkill for ARR-only users.

6. **Flemmarr's low adoption** (~336 stars despite being older) shows that raw API configuration without abstraction isn't what users want. Users want *magic*, not a different syntax for the same manual work.

## Sources

- `ARR_Ecosysteme_Alternatives_Detaillees.md` (local project file, Feb 2026 research) -- PRIMARY source, MEDIUM-HIGH confidence
- `ARRmada_PROJECT_PLAN.md` (local project file) -- project context and architectural decisions
- `.planning/PROJECT.md` (local project file) -- validated requirements and constraints
- Training data knowledge of *ARR ecosystem, Docker patterns, homelab tooling -- LOW-MEDIUM confidence (used to supplement, not as primary)

### Confidence Notes

- Competitor feature analysis is based primarily on the user's own research document dated Feb 2026, which cites GitHub repos, stars, and direct feature inspection. **MEDIUM-HIGH confidence.**
- Feature prioritization and dependency analysis is based on domain knowledge and the project's stated goals. **MEDIUM confidence** -- should be validated against actual user feedback.
- Anti-features list is opinionated based on scope analysis. **HIGH confidence** for V1 scoping decisions; the "why not" reasoning is sound.
- Star counts and activity status may have changed since the research doc was written (Feb 2026). **LOW confidence** on exact numbers.

---
*Feature research for: *ARR media stack installer and auto-configurator*
*Researched: 2026-02-22*
