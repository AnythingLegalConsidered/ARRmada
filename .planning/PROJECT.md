# ARRmada

## What This Is

ARRmada is a batteries-included installer and auto-configurator for the *ARR media automation ecosystem. It deploys, wires, and configures an entire media stack (from indexers to media player) so users go from zero to streaming in minutes — not hours. It ships as a CLI + web wizard, targeting both power users who are tired of manual setup and complete beginners guided step-by-step.

## Core Value

One command deploys a fully connected, optimally configured media stack — zero manual configuration required.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] One-liner install script that sets up Docker if absent and launches the wizard
- [ ] Docker run method for users who already have Docker
- [ ] Interactive CLI with Textual TUI (service selection, paths, VPN config)
- [ ] Web wizard with step-by-step guided setup
- [ ] Docker Compose generation for selected services (Jinja2 templates)
- [ ] .env generation (PUID, PGID, TZ, paths, ports)
- [ ] Directory structure creation with correct permissions and hardlink-friendly layout
- [ ] Deploy command (docker compose up)
- [ ] Status command (container health check)
- [ ] Destroy command (clean teardown)
- [ ] Service health polling (wait for all services to be ready)
- [ ] API key extraction from service config files
- [ ] Prowlarr auto-wiring (add Sonarr, Radarr as applications)
- [ ] qBittorrent auto-config as download client in Sonarr, Radarr
- [ ] Root folder auto-setup (media paths in each *ARR)
- [ ] Jellyseerr auto-config (connect Jellyfin + Sonarr + Radarr)
- [ ] Bazarr auto-config (connect to Sonarr + Radarr)
- [ ] Recyclarr integration (apply TRaSH Guides quality profiles)
- [ ] Gluetun VPN integration (optional, user chooses provider + credentials)
- [ ] qBittorrent traffic routed through Gluetun when VPN enabled

### Out of Scope

- Natural language download ("download The Batman 2022 in 4K") — complex NLP/LLM feature, deferred to V2
- Reverse proxy (Traefik/Caddy) — V1 targets local network, remote access in V2
- Lidarr, Readarr, Huntarr — extended *ARR services deferred to V2
- Tdarr, Byparr — advanced infra deferred to V2
- Homepage/Jellystat — monitoring dashboard deferred to V2
- Backup/Restore — useful but not V1 scope
- Auto-update mechanism — V1 ships, updates are manual

## Context

- The *ARR ecosystem (Sonarr, Radarr, Prowlarr, etc.) is the standard for automated media management
- Existing installers (Ezarr, Deployarr, Cloud Seeder) stop at `docker compose up` — they don't auto-wire services together
- Config tools (Buildarr, Flemmarr) handle configuration but not deployment
- No tool does both deployment AND full auto-configuration in one flow
- TRaSH Guides are the community gold standard for quality profiles — Recyclarr is the official sync tool
- Hardlinks require a single `/data` root mount (TRaSH Guides best practice) to avoid duplicate disk usage
- *ARR services generate API keys on first boot in their `config.xml` files — these can be read programmatically
- Gluetun supports 60+ VPN providers via Docker env vars

## Constraints

- **Docker required**: All services run as containers. Docker must be present or installed by the one-liner script.
- **Linux primary**: V1 targets Linux (Debian/Ubuntu, mini-PCs, VPS). macOS best-effort, Windows WSL2 possible but not primary.
- **Python 3.11+**: Core engine in Python for ecosystem compatibility (httpx, Jinja2, Pydantic, Textual)
- **No external DB**: ARRmada config stored as YAML files, no database dependency
- **Portfolio quality**: Public GitHub repo, clean code, good README, MIT license

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Python + Textual for CLI | Same language as core engine, rich TUI without ncurses pain | — Pending |
| FastAPI for web wizard | Lightweight, auto-docs, same Python codebase | — Pending |
| Jinja2 for compose generation | Battle-tested templating, supports conditionals for optional services | — Pending |
| httpx async for API wiring | Fast parallel configuration of all services | — Pending |
| Pydantic for config model | Validation, serialization, good DX | — Pending |
| Single /data root (hardlinks) | TRaSH Guides best practice, instant moves, no disk waste | — Pending |
| Recyclarr bundled (not reimplemented) | Maintained upstream, idempotent, handles edge cases | — Pending |
| Gluetun for VPN | 60+ providers, well-maintained, Docker-native network isolation | — Pending |
| V1 = local network only | Simplifies scope, reverse proxy adds significant complexity | — Pending |

---
*Last updated: 2026-02-22 after initialization*
