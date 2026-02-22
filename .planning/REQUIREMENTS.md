# Requirements: ARRmada

**Defined:** 2026-02-22
**Core Value:** One command deploys a fully connected, optimally configured media stack — zero manual configuration required.

## v1 Requirements

Requirements for initial release (CLI-only). Each maps to roadmap phases.

### Deployment

- [ ] **DEPL-01**: User can generate a Docker Compose file for selected services via CLI wizard
- [ ] **DEPL-02**: User can generate a .env file with auto-detected PUID, PGID, TZ, and prompted paths/ports
- [ ] **DEPL-03**: User can create the full directory structure with correct permissions and hardlink-friendly layout (/data root)
- [ ] **DEPL-04**: User can deploy the stack with a single `arrmada deploy` command
- [ ] **DEPL-05**: User can check all container health with `arrmada status`
- [ ] **DEPL-06**: User can tear down the stack cleanly with `arrmada destroy` (with data preservation option)
- [ ] **DEPL-07**: User can install ARRmada via one-liner script (`curl | bash`) that installs Docker if absent
- [ ] **DEPL-08**: User can deploy with zero-config defaults (accept all defaults, get a working stack)
- [ ] **DEPL-09**: User sees a post-install summary with all service URLs and default credentials

### Auto-Configuration

- [ ] **CONF-01**: ARRmada waits for all services to be healthy before starting configuration (health polling with exponential backoff)
- [ ] **CONF-02**: ARRmada extracts API keys automatically from service config.xml files
- [ ] **CONF-03**: ARRmada adds Sonarr and Radarr as applications in Prowlarr (indexer sync)
- [ ] **CONF-04**: ARRmada adds qBittorrent as download client in Sonarr and Radarr
- [ ] **CONF-05**: ARRmada sets up root folders (media paths) in Sonarr and Radarr
- [ ] **CONF-06**: ARRmada connects Jellyseerr to Jellyfin, Sonarr, and Radarr (complete first-run setup)
- [ ] **CONF-07**: ARRmada connects Bazarr to Sonarr and Radarr for subtitle management
- [ ] **CONF-08**: ARRmada generates Recyclarr config and runs TRaSH Guides quality profile sync
- [ ] **CONF-09**: All auto-configuration steps are idempotent (safe to re-run without creating duplicates)
- [ ] **CONF-10**: User can run auto-configuration standalone with `arrmada configure`

### VPN

- [ ] **VPN-01**: User can optionally enable Gluetun VPN during setup (choose provider + enter credentials)
- [ ] **VPN-02**: When VPN is enabled, qBittorrent traffic is routed through Gluetun (network_mode)
- [ ] **VPN-03**: All download client configurations use correct hostname (gluetun vs qbittorrent) based on VPN setting

### CLI Interface

- [ ] **CLI-01**: User can select services via interactive Textual TUI (tier-based with individual toggles)
- [ ] **CLI-02**: User can configure data paths via TUI
- [ ] **CLI-03**: User can configure VPN provider and credentials via TUI
- [ ] **CLI-04**: User sees real-time deployment and configuration progress in TUI
- [ ] **CLI-05**: User can run all commands non-interactively with flags (for scripting/automation)

### Project Quality

- [ ] **QUAL-01**: Project has clean README with installation instructions and architecture overview
- [ ] **QUAL-02**: Project has pyproject.toml with proper metadata and dependencies
- [ ] **QUAL-03**: Project uses typed Python (Pydantic models, type hints throughout)
- [ ] **QUAL-04**: Project has MIT license
- [ ] **QUAL-05**: Config is stored as YAML (arrmada.yml) for reproducible deployments

## v1.1 Requirements

Deferred to post-V1 release. Tracked but not in current roadmap.

### Web Wizard

- **WEB-01**: User can access a web wizard via `docker run` on port 8888
- **WEB-02**: Web wizard provides step-by-step guided setup (service selection, paths, VPN)
- **WEB-03**: Web wizard shows real-time deployment progress via WebSocket
- **WEB-04**: Web wizard container self-removes after successful deployment

### Extended Services

- **EXT-01**: User can add Lidarr (music) to the stack
- **EXT-02**: User can add Readarr (ebooks) to the stack
- **EXT-03**: User can add Huntarr (missing content hunter) to the stack
- **EXT-04**: Auto-wiring for Tier 2 services (Prowlarr + qBit + Huntarr connections)

## v2 Requirements

### Advanced Features

- **ADV-01**: Natural language download ("download The Batman 2022 in 4K")
- **ADV-02**: Reverse proxy addon (Traefik/Caddy with auto-SSL)
- **ADV-03**: Backup/restore functionality
- **ADV-04**: Auto-update mechanism (`arrmada update`)
- **ADV-05**: Tdarr transcoding integration
- **ADV-06**: Monitoring dashboard (Homepage/Jellystat)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Plex support | Jellyfin is open-source, no account needed. Plex doubles testing surface. |
| Multi-server deployment | Massive complexity (networking, service discovery). V1 is single-host. |
| App store / marketplace | ARRmada is for the ARR stack, not a general homelab manager. |
| Authentication layer (Authelia) | Requires reverse proxy. V1 targets local network. |
| Indexer auto-add | Indexers require user-specific accounts/invites. Can't be automated. |
| Windows native | V1 targets Linux. WSL2 is possible but not primary. |
| Auto-updates with rollback | Unattended updates break things. Manual `arrmada update` in V2. |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| DEPL-01 | Phase 1 | Pending |
| DEPL-02 | Phase 1 | Pending |
| DEPL-03 | Phase 1 | Pending |
| DEPL-04 | Phase 1 | Pending |
| DEPL-05 | Phase 1 | Pending |
| DEPL-06 | Phase 1 | Pending |
| DEPL-07 | Phase 6 | Pending |
| DEPL-08 | Phase 6 | Pending |
| DEPL-09 | Phase 3 | Pending |
| CONF-01 | Phase 2 | Pending |
| CONF-02 | Phase 2 | Pending |
| CONF-03 | Phase 2 | Pending |
| CONF-04 | Phase 2 | Pending |
| CONF-05 | Phase 2 | Pending |
| CONF-06 | Phase 3 | Pending |
| CONF-07 | Phase 3 | Pending |
| CONF-08 | Phase 3 | Pending |
| CONF-09 | Phase 2 | Pending |
| CONF-10 | Phase 2 | Pending |
| VPN-01 | Phase 4 | Pending |
| VPN-02 | Phase 4 | Pending |
| VPN-03 | Phase 4 | Pending |
| CLI-01 | Phase 5 | Pending |
| CLI-02 | Phase 5 | Pending |
| CLI-03 | Phase 5 | Pending |
| CLI-04 | Phase 5 | Pending |
| CLI-05 | Phase 5 | Pending |
| QUAL-01 | Phase 6 | Pending |
| QUAL-02 | Phase 1 | Pending |
| QUAL-03 | Phase 1 | Pending |
| QUAL-04 | Phase 6 | Pending |
| QUAL-05 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 32 total
- Mapped to phases: 32
- Unmapped: 0

---
*Requirements defined: 2026-02-22*
*Last updated: 2026-02-22 after roadmap creation*
