# Roadmap: ARRmada

## Overview

ARRmada goes from zero to a fully wired media stack in six phases. We start by getting containers running with correct paths and permissions (Foundation), then wire the core *ARR services via their APIs (Core Auto-Config), extend wiring to Jellyseerr/Bazarr/Recyclarr (Extended Wiring), add optional VPN support (VPN Integration), build the Textual TUI as the primary user interface (CLI TUI), and finish with packaging and polish (Ship It).

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation** - Project scaffold, compose generation, deploy/status/destroy commands
- [ ] **Phase 2: Core Auto-Configuration** - Health polling, API key extraction, Prowlarr/qBit/root folder wiring
- [ ] **Phase 3: Extended Wiring** - Jellyseerr + Bazarr + Recyclarr auto-config, post-install summary
- [ ] **Phase 4: VPN Integration** - Gluetun VPN with conditional hostname logic
- [ ] **Phase 5: CLI TUI** - Textual interactive interface for service selection, config, and progress
- [ ] **Phase 6: Ship It** - One-liner install, README, license, zero-config defaults

## Phase Details

### Phase 1: Foundation
**Goal**: User can generate, deploy, and manage a containerized media stack with correct directory structure
**Depends on**: Nothing (first phase)
**Requirements**: DEPL-01, DEPL-02, DEPL-03, DEPL-04, DEPL-05, DEPL-06, QUAL-02, QUAL-03, QUAL-05
**Success Criteria** (what must be TRUE):
  1. User can run `arrmada deploy` and all selected containers start with healthy status
  2. User can run `arrmada status` and see health state of every container
  3. User can run `arrmada destroy` and all containers/networks are removed (with option to preserve data)
  4. Generated directory structure follows TRaSH Guides hardlink-friendly layout with correct PUID/PGID ownership
  5. Project is installable via pip/uv with typed Pydantic config model and arrmada.yml persistence
**Plans**: TBD

Plans:
- [ ] 01-01: Project scaffold (pyproject.toml, package structure, Pydantic config model, arrmada.yml)
- [ ] 01-02: Compose generation engine (Jinja2 templates, .env generation, directory structure creation)
- [ ] 01-03: Stack lifecycle commands (deploy, status, destroy)

### Phase 2: Core Auto-Configuration
**Goal**: ARRmada automatically wires Prowlarr, qBittorrent, and root folders into Sonarr/Radarr after deployment
**Depends on**: Phase 1
**Requirements**: CONF-01, CONF-02, CONF-03, CONF-04, CONF-05, CONF-09, CONF-10
**Success Criteria** (what must be TRUE):
  1. After `arrmada configure`, Prowlarr shows Sonarr and Radarr as connected applications
  2. After `arrmada configure`, Sonarr and Radarr each have qBittorrent configured as download client
  3. After `arrmada configure`, Sonarr and Radarr each have correct root folders (media library paths)
  4. Running `arrmada configure` a second time produces no duplicates or errors (idempotent)
**Plans**: TBD

Plans:
- [ ] 02-01: Health polling engine (exponential backoff, service readiness detection)
- [ ] 02-02: API key extraction and service abstraction layer
- [ ] 02-03: Core wiring (Prowlarr apps, qBit download client, root folders in Sonarr/Radarr)

### Phase 3: Extended Wiring
**Goal**: ARRmada completes the "zero to streaming" promise by wiring Jellyseerr, Bazarr, and Recyclarr
**Depends on**: Phase 2
**Requirements**: CONF-06, CONF-07, CONF-08, DEPL-09
**Success Criteria** (what must be TRUE):
  1. After configuration, Jellyseerr is connected to Jellyfin and shows Sonarr + Radarr as configured services (first-run wizard fully automated)
  2. After configuration, Bazarr shows Sonarr and Radarr as connected providers for subtitle management
  3. After configuration, Recyclarr has applied TRaSH Guides quality profiles to Sonarr and Radarr
  4. User sees a post-install summary with all service URLs and default credentials
**Plans**: TBD

Plans:
- [ ] 03-01: Jellyfin + Jellyseerr first-run automation (highest risk -- needs live API verification)
- [ ] 03-02: Bazarr + Recyclarr wiring and post-install summary

### Phase 4: VPN Integration
**Goal**: User can optionally route download traffic through a VPN with zero manual network configuration
**Depends on**: Phase 2
**Requirements**: VPN-01, VPN-02, VPN-03
**Success Criteria** (what must be TRUE):
  1. User can enable VPN during setup by choosing a Gluetun-supported provider and entering credentials
  2. When VPN is enabled, qBittorrent container uses Gluetun's network (network_mode: service:gluetun)
  3. All download client configurations use the correct hostname (gluetun when VPN on, qbittorrent when VPN off)
**Plans**: TBD

Plans:
- [ ] 04-01: Gluetun compose integration and conditional hostname logic

### Phase 5: CLI TUI
**Goal**: User interacts with ARRmada through an interactive Textual TUI for all setup and monitoring tasks
**Depends on**: Phase 3, Phase 4
**Requirements**: CLI-01, CLI-02, CLI-03, CLI-04, CLI-05
**Success Criteria** (what must be TRUE):
  1. User can select/deselect services via an interactive TUI with tier-based grouping and individual toggles
  2. User can configure data paths and VPN settings through the TUI without editing files manually
  3. User sees real-time deployment and configuration progress (container status, wiring steps) in the TUI
  4. Every TUI operation can also be performed non-interactively via CLI flags (scriptable)
**Plans**: TBD

Plans:
- [ ] 05-01: Textual TUI screens (service selection, path config, VPN config)
- [ ] 05-02: Progress display and non-interactive CLI flag support

### Phase 6: Ship It
**Goal**: ARRmada is installable by anyone with one command and works out of the box with zero configuration
**Depends on**: Phase 5
**Requirements**: DEPL-07, DEPL-08, QUAL-01, QUAL-04
**Success Criteria** (what must be TRUE):
  1. User can install ARRmada via `curl | bash` one-liner that handles Docker installation if absent
  2. User can accept all defaults and get a working, fully wired media stack (zero decisions required)
  3. GitHub repo has a clean README with installation instructions, architecture overview, and MIT license
**Plans**: TBD

Plans:
- [ ] 06-01: One-liner install script and zero-config defaults
- [ ] 06-02: README, license, and repo polish

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6
Note: Phase 4 (VPN) depends on Phase 2 only, so it could execute in parallel with Phase 3 if needed.

| Phase | Plans Complete | Status | Completed |
|-------|---------------|--------|-----------|
| 1. Foundation | 0/3 | Not started | - |
| 2. Core Auto-Configuration | 0/3 | Not started | - |
| 3. Extended Wiring | 0/2 | Not started | - |
| 4. VPN Integration | 0/1 | Not started | - |
| 5. CLI TUI | 0/2 | Not started | - |
| 6. Ship It | 0/2 | Not started | - |
