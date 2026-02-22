# Research Summary: ARRmada

**Domain:** Docker-based installer + auto-configurator for *ARR media ecosystem
**Researched:** 2026-02-22
**Overall confidence:** MEDIUM (solid architectural patterns, verified library choices, but API endpoint details and exact library versions need live verification)

## Executive Summary

ARRmada fills a clearly documented gap in the *ARR ecosystem: no open-source Docker-based tool both deploys AND auto-configures a complete media stack. Ezarr (1000+ stars) deploys but doesn't wire. Buildarr configures but doesn't deploy. Cloud Seeder does partial wiring (qBit only). The closest competitor, Nixflix, does everything but is NixOS-only (excludes 95%+ of users). ARRmada's core value proposition -- "deploy + wire + configure in one flow" -- is validated by the ecosystem gap and proven demand from existing partial solutions.

The recommended stack is Python 3.12+ with Pydantic V2 (config model), Jinja2 (compose templating), httpx (async API wiring), Textual (CLI TUI), and FastAPI (web wizard). All libraries are mature, well-maintained, and part of the existing project decisions. The key frontend decision is to use htmx + Alpine.js + Pico CSS for the web wizard -- zero build step, no npm, everything ships as static files. This is correct for a 5-6 screen wizard that self-removes after use.

The architecture centers on a service abstraction layer where each deployable service implements a common interface (compose fragment, health check, API key extraction, API configuration). The auto-configuration engine is the most complex component: it must handle service health polling with exponential backoff, API key extraction from config.xml files, a precise wiring sequence (API keys -> qBit auth -> Sonarr/Radarr parallel -> Prowlarr -> Recyclarr -> Jellyseerr -> Bazarr), and idempotent operations. Every wiring step must check existing state before modifying.

The most dangerous pitfalls are: (1) broken hardlinks from separate volume mounts (must enforce single /data root), (2) PUID/PGID permission chaos (auto-detect and set consistently), (3) API wiring race conditions (robust health polling required), (4) Gluetun VPN breaking inter-container networking (hostname changes from qbittorrent to gluetun), and (5) Jellyfin/Jellyseerr first-run wizards blocking automation (must automate these programmatically via their Startup APIs). The Jellyseerr setup flow is the least documented and highest-risk integration point.

## Key Findings

**Stack:** Python 3.12+, Pydantic V2, Jinja2, httpx async, Textual TUI, FastAPI, htmx+Alpine.js for web. uv for packaging, ruff for linting. No npm, no database, no JS framework.

**Architecture:** Service abstraction layer with per-service classes. Async parallel wiring via anyio task groups. Pydantic config model as single source of truth shared between CLI and web frontends. Jinja2 template composition with per-service fragments.

**Critical pitfall:** Jellyseerr and Jellyfin first-run wizards must be automated via their APIs or the "zero manual config" promise is broken. This is the highest-risk, least-documented integration point. Needs phase-specific research with live Swagger docs.

## Implications for Roadmap

Based on research, suggested phase structure:

1. **Foundation (Compose + Deploy + Directory)** - Get containers running with correct paths and permissions
   - Addresses: Compose generation, .env generation, directory structure, deploy/status/destroy commands
   - Avoids: Hardlink breakage (enforce single /data root from day one), permission chaos (auto-detect PUID/PGID)

2. **Auto-Configuration Engine (the differentiator)** - Wire all services via their REST APIs
   - Addresses: API key extraction, health polling, Prowlarr wiring, qBit config, root folders, Sonarr/Radarr download clients
   - Avoids: Race conditions (health polling with backoff), non-idempotent operations (check-before-modify pattern)

3. **Extended Wiring (Jellyseerr + Recyclarr + Bazarr)** - Complete the "zero to streaming" promise
   - Addresses: Jellyfin first-run automation, Jellyseerr setup flow, Recyclarr config generation, Bazarr connections
   - Avoids: First-run wizard blocks (automate via Startup APIs), Recyclarr template mismatch (wizard asks quality preference)

4. **CLI TUI (Textual)** - Primary user interface
   - Addresses: Service selection, path config, VPN setup, deployment progress, post-install summary
   - Avoids: Wall-of-text UX (progress bars, status dashboard, clickable URLs)

5. **Web Wizard (FastAPI + htmx)** - Browser-based guided setup
   - Addresses: Beginner-friendly wizard, real-time progress via WebSocket/SSE, self-removing container
   - Avoids: Overengineering (no JS framework, no build step)

6. **Advanced Features (VPN + Tier 2/3 + Polish)** - Extended services and hardening
   - Addresses: Gluetun VPN integration, Lidarr/Readarr/Huntarr, Tdarr/Byparr, one-liner install script
   - Avoids: Gluetun networking gotcha (conditional hostname logic), scope creep (no reverse proxy, no auth layer)

**Phase ordering rationale:**
- Phase 1 before Phase 2: Containers must run before they can be configured. Foundation decisions (paths, permissions, volume structure) cannot be changed later without data migration.
- Phase 2 before Phase 3: Core wiring (Prowlarr, qBit, root folders) is simpler and better-documented than Jellyseerr/Jellyfin first-run automation. Ship the easy wins first.
- Phase 3 before Phase 4: The auto-configuration engine must work before building a UI around it. CLI-only testing is faster.
- Phase 4 before Phase 5: TUI is simpler (no frontend build step) and serves power users. Web wizard requires more work and serves beginners (secondary audience for V1).
- Phase 6 last: VPN and Tier 2/3 are optional complexity. The core stack (Tier 1 + auto-wiring) must be solid first.

**Research flags for phases:**
- Phase 3: NEEDS deeper research. Jellyseerr setup API is poorly documented. Jellyfin Startup API needs live verification. Test with actual running instances.
- Phase 6: NEEDS deeper research for Gluetun provider-specific configuration (60+ providers, each with different env vars).
- Phase 1: Standard patterns, LOW risk. Docker Compose generation and directory creation are well-understood.
- Phase 2: MEDIUM risk. Servarr API endpoints are documented via Swagger but version-specific details may vary. Verify against live instances.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM-HIGH | Library choices are sound and well-established. Exact version numbers need verification (training data May 2025). |
| Features | HIGH | Based on detailed competitor analysis document (Feb 2026) with GitHub repo inspection. Feature gaps are clear and validated. |
| Architecture | MEDIUM | Patterns are standard Python (Pydantic, httpx async, Jinja2 templating). API endpoint shapes from training data -- MUST verify against live Swagger docs. |
| Pitfalls | MEDIUM | Based on well-known community patterns (TRaSH Guides, Servarr wiki, r/selfhosted). Live verification was not possible (WebSearch/WebFetch unavailable). |

## Gaps to Address

- **Jellyseerr API verification:** The first-run setup flow is the least documented integration. Must test with a live Jellyseerr instance during Phase 3 implementation. Swagger docs at `http://jellyseerr:5055/api-docs`.
- **Jellyfin Startup API verification:** Endpoints for completing the first-run wizard need live testing. Documentation at `https://api.jellyfin.org/`.
- **Bazarr API/config format:** Bazarr is not a Servarr app -- its config storage has changed between versions. Verify current format during Phase 3.
- **qBittorrent default password behavior:** Since v4.6.1, random password is logged. Verify LinuxServer.io image supports `WEBUI_PASSWORD` env var.
- **Exact library versions:** All version numbers from training data (May 2025). Run `uv add --dry-run <package>` to verify latest before locking `pyproject.toml`.
- **Gluetun provider-specific env vars:** 60+ VPN providers, each with different configuration. Need a mapping table or dynamic detection. Research during Phase 6.
- **Recyclarr template names:** Template names may change between Recyclarr releases. Verify current template list with `recyclarr list templates`.
- **Sonarr v4 vs v3 API differences:** Sonarr v4 is current; some community docs still reference v3 patterns. Verify Swagger endpoint shapes.

## Files Created

| File | Purpose | Size |
|------|---------|------|
| `STACK.md` | Technology recommendations with versions, rationale, alternatives, and what NOT to use | Complete |
| `FEATURES.md` | Feature landscape: table stakes, differentiators, anti-features, dependency graph, MVP definition, competitor matrix | Complete |
| `ARCHITECTURE.md` | System architecture, API endpoint details for every service, wiring sequence, code patterns, anti-patterns | Complete |
| `PITFALLS.md` | Critical pitfalls (hardlinks, permissions, race conditions, VPN, first-run wizards), integration gotchas, platform issues | Complete |
| `SUMMARY.md` | This file -- executive summary with roadmap implications | Complete |

---
*Research summary for: ARRmada -- *ARR media stack installer/auto-configurator*
*Researched: 2026-02-22*
