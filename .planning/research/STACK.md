# Stack Research

**Domain:** Docker-based installer + auto-configurator for *ARR media ecosystem
**Researched:** 2026-02-22
**Confidence:** MEDIUM (versions from training data May 2025 -- verify with `pip index versions <pkg>` before locking)

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended | Confidence |
|------------|---------|---------|-----------------|------------|
| **Python** | 3.12+ | Core engine language | Universal on Linux, async-native (asyncio), rich ecosystem for HTTP/templating/CLI. 3.12 has better error messages, performance improvements, and is the current stable branch. 3.11 as floor for `tomllib` stdlib and `ExceptionGroup` support. | HIGH |
| **Pydantic** | ^2.9 | Config model, validation, serialization | V2 is a rewrite in Rust (pydantic-core) -- 5-50x faster than V1. `model_validate`, `model_dump`, discriminated unions for service configs. JSON Schema generation for web wizard validation. Never use Pydantic V1 API. | HIGH |
| **Jinja2** | ^3.1 | Docker Compose + .env template generation | Battle-tested, supports conditionals/loops for optional services (tiers), auto-escaping. No serious competitor for Python templating. | HIGH |
| **httpx** | ^0.28 | Async HTTP client for *ARR API wiring | Native async/await, HTTP/2 support, timeout controls, connection pooling. Used to configure all services in parallel via their REST APIs. Drop-in replacement for requests with async. | HIGH |
| **Textual** | ^1.0 | CLI TUI framework | Rich terminal UIs with CSS-like styling, widgets (selectors, progress bars, data tables). Built on Rich. Same Python codebase as core engine -- no context switch. Actively maintained by Textualize. | MEDIUM |
| **FastAPI** | ^0.115 | Web wizard backend | Auto-generates OpenAPI docs, native Pydantic integration (same models for CLI and web), WebSocket support for live deployment progress, async by default. Lightweight enough to self-remove after install. | HIGH |
| **uvicorn** | ^0.32 | ASGI server for FastAPI | The standard ASGI server for FastAPI. Use `uvicorn[standard]` for uvloop + httptools performance. | HIGH |
| **Rich** | ^13.9 | Terminal formatting, logging, progress bars | Dependency of Textual anyway. Use for CLI-only mode fallback (non-TUI), structured logging, and deployment progress display. | HIGH |
| **PyYAML** | ^6.0 | YAML config file read/write | ARRmada config (`arrmada.yml`), Recyclarr config generation, Homepage config. stdlib has no YAML support. | HIGH |
| **docker** (py) | ^7.1 | Docker Engine API client | Programmatic Docker control: check Docker is running, inspect containers, read health status, exec into containers. Prefer over shelling out to `docker` CLI for reliability and structured output. | MEDIUM |

### Supporting Libraries

| Library | Version | Purpose | When to Use | Confidence |
|---------|---------|---------|-------------|------------|
| **anyio** | ^4.7 | Async runtime abstraction | Used by httpx and FastAPI internally. Pin to avoid version conflicts. Provides structured concurrency (task groups) for parallel API wiring. | HIGH |
| **pydantic-settings** | ^2.7 | Environment variable + .env loading | Load user config from env vars, `.env` files, and `arrmada.yml` with validation. Separates settings from data models cleanly. | HIGH |
| **websockets** | ^14.0 | WebSocket support for FastAPI | Real-time deployment progress in web wizard. FastAPI uses this under the hood for WS endpoints. | MEDIUM |
| **typer** | ^0.15 | CLI argument parsing | Wraps Click with type hints. Use for `arrmada deploy`, `arrmada status`, `arrmada configure` subcommands. Textual TUI launches from typer commands. | MEDIUM |
| **platformdirs** | ^4.3 | Cross-platform config paths | Resolve `~/.config/arrmada/`, `/etc/arrmada/` correctly across Linux/macOS. Don't hardcode paths. | HIGH |
| **tenacity** | ^9.0 | Retry logic | Service health polling (wait for containers to be ready), API key extraction retries (config.xml may not exist yet). Exponential backoff, configurable stop conditions. | HIGH |
| **python-dotenv** | ^1.0 | .env file parsing | Read existing .env files if user has a prior setup. Pydantic-settings uses this as backend. | HIGH |
| **xmltodict** | ^0.14 | XML parsing for *ARR config files | *ARR services store API keys in `config.xml`. xmltodict is simpler than ElementTree for extracting single values. Alternatively, use stdlib `xml.etree.ElementTree` to avoid the dep. | LOW |

### Development Tools

| Tool | Purpose | Notes | Confidence |
|------|---------|-------|------------|
| **uv** | Package manager + venv + build | 10-100x faster than pip. Replaces pip, pip-tools, virtualenv. Use `uv sync` for lockfile, `uv run` for scripts. The 2025+ standard for Python projects. | HIGH |
| **ruff** | Linter + formatter | Replaces flake8 + black + isort in one tool (Rust-based, instant). Configure in `pyproject.toml`. | HIGH |
| **pytest** | Test framework | With `pytest-asyncio` for async test support (httpx calls, API wiring tests). | HIGH |
| **pytest-asyncio** | Async test support | Required for testing async httpx calls to *ARR APIs. Use `mode = "auto"` in pytest config. | HIGH |
| **mypy** | Type checking | Strict mode. Pydantic V2 has native mypy plugin. Catch config model errors at dev time. | MEDIUM |
| **pre-commit** | Git hook manager | Run ruff + mypy on commit. Enforce code quality from day one. | MEDIUM |
| **hatch** / **uv build** | Build backend | Use hatchling as build backend in `pyproject.toml`. Produces wheels for `pip install arrmada`. uv can build too. | MEDIUM |

### Frontend (Web Wizard)

| Technology | Purpose | Why | Confidence |
|------------|---------|-----|------------|
| **Alpine.js** ~3.14 | Reactive frontend | No build step, 15KB gzipped, perfect for a wizard UI. Sprinkle reactivity onto HTML. No npm/webpack/vite needed -- ship as static files inside the Python package. | MEDIUM |
| **htmx** ~2.0 | Server-driven UI updates | Alternative to Alpine.js. Let FastAPI render HTML fragments, htmx swaps them. Even less JS. Perfect for wizard step transitions and live status. | MEDIUM |
| **Pico CSS** ~2.1 | Classless CSS framework | Clean defaults, dark mode, responsive. No class names needed for basic elements. Tiny (~10KB). Pairs well with Alpine.js/htmx. | LOW |

**Recommendation:** Use **htmx + Alpine.js + Pico CSS**. htmx handles wizard step navigation and live deployment updates (SSE). Alpine.js handles interactive form elements (tier selector, VPN provider dropdown, path inputs). Pico CSS provides a clean look with zero effort. **No build step, no npm, no bundler.** Everything ships as static files in `src/arrmada/web/static/`.

## Installation

```bash
# Project setup with uv
uv init arrmada
cd arrmada

# Core dependencies
uv add pydantic pydantic-settings jinja2 httpx[http2] \
       textual rich typer pyyaml docker tenacity \
       platformdirs python-dotenv anyio

# Web wizard
uv add fastapi "uvicorn[standard]" websockets

# Dev dependencies
uv add --dev pytest pytest-asyncio ruff mypy pre-commit
```

```toml
# pyproject.toml essentials
[project]
name = "arrmada"
requires-python = ">=3.11"

[project.scripts]
arrmada = "arrmada.cli:app"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "A", "SIM", "TCH"]

[tool.pytest.ini_options]
asyncio_mode = "auto"

[tool.mypy]
python_version = "3.11"
strict = true
plugins = ["pydantic.mypy"]
```

## Alternatives Considered

| Category | Recommended | Alternative | When to Use Alternative |
|----------|-------------|-------------|-------------------------|
| HTTP client | httpx | aiohttp | Never for this project -- httpx has cleaner API, better typing, requests-compatible interface |
| CLI framework | Typer + Textual | Click | Only if you want raw Click; Typer wraps Click with better DX |
| CLI TUI | Textual | curses / urwid | Never -- Textual is strictly superior for Python TUIs in 2025+ |
| Templating | Jinja2 | Mako / Chevron | Never -- Jinja2 is the Python standard, FastAPI/Starlette also use it |
| Config model | Pydantic V2 | dataclasses + marshmallow | Only if you hate fast validation -- Pydantic V2 is the standard |
| YAML | PyYAML | ruamel.yaml | Use ruamel.yaml IF you need to preserve comments in user-edited YAML. For generated configs (compose, recyclarr), PyYAML is fine. |
| ASGI server | uvicorn | hypercorn / granian | Granian if you need multi-process without gunicorn. Uvicorn is the standard for FastAPI. |
| Package manager | uv | pip + pip-tools | Only if uv is unavailable on target system (unlikely in 2026). Provide pip fallback in install.sh. |
| Frontend | htmx + Alpine.js | React / Vue / Svelte | Never -- the wizard is 5-6 screens max. A JS framework adds build complexity for zero benefit. |
| CSS | Pico CSS | Tailwind / Bootstrap | Tailwind requires build step. Bootstrap is heavy. Pico is classless and tiny. |
| Docker API | docker (py) | subprocess + docker CLI | Use subprocess as fallback if `docker` package causes issues. The package gives structured responses. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **Pydantic V1** | Deprecated, 50x slower, different API surface. V2 is stable since June 2023. | Pydantic V2 (`pydantic>=2.0`) |
| **requests** (sync HTTP) | Blocks the event loop. All *ARR API wiring should be async for parallel execution (10+ services). | httpx with async |
| **Flask** | No native async, no Pydantic integration, no auto-docs. Would need flask-cors, flask-socketio, marshmallow separately. | FastAPI |
| **argparse** | Verbose, no type inference, no TUI. Building a wizard with argparse would be painful. | Typer (wraps Click) |
| **docker-compose CLI** via subprocess | Brittle string parsing, version differences (V1 vs V2), no structured output. | `docker` Python package + `docker compose` as last resort |
| **React/Vue/Svelte** | Adds npm, node_modules, bundler, build step for a 5-screen wizard. Overkill. Bloats the Docker image. | htmx + Alpine.js |
| **SQLite / PostgreSQL** | ARRmada is a one-shot installer, not a long-running app. YAML config is sufficient. A database adds migration complexity for zero value. | YAML files (PyYAML) |
| **Ansible / Pulumi** | Infrastructure-as-code tools are overkill for generating one docker-compose.yml. They add massive dependencies. | Jinja2 templates |
| **pip** (as user-facing tool) | Slow, no lockfile, no venv management. Users may pollute system Python. | uv or Docker-based install |
| **Jackett** | Deprecated in favor of Prowlarr. Prowlarr syncs indexers to all *ARR apps automatically. Jackett requires manual per-service config. | Prowlarr (already in stack) |
| **FlareSolverr** | Broken/unmaintained. Cloudflare has outpaced it. | Byparr (SeleniumBase-based, active development) |
| **Overseerr** | Jellyfin support was always an afterthought. Jellyseerr is the maintained fork with native Jellyfin support. Projects are merging. | Jellyseerr |

## Stack Patterns by Variant

**If deploying as Docker container (web wizard method):**
- Bundle Python + static files in a slim Docker image (~150MB target)
- Mount Docker socket (`/var/run/docker.sock`) for container management
- Self-remove wizard container after successful deployment
- Use multi-stage build: build stage with uv, runtime with python:3.12-slim

**If deploying via pip install (CLI method):**
- Publish to PyPI as `arrmada`
- Entry point: `arrmada` CLI via typer
- Require Docker to be pre-installed (check on startup)
- Use `uv tool install arrmada` as recommended method

**If deploying via one-liner (curl | bash):**
- `install.sh` checks for Docker, installs if missing (apt/dnf)
- Downloads latest `arrmada` wheel or uses `uv tool install`
- Launches TUI wizard immediately
- Fallback: `docker run` if Python is not available

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| Pydantic ^2.9 | FastAPI ^0.115 | FastAPI requires Pydantic V2 since 0.100+ |
| httpx ^0.28 | anyio ^4.7 | httpx uses anyio internally for async |
| Textual ^1.0 | Rich ^13.0 | Textual depends on Rich, versions are co-released by Textualize |
| FastAPI ^0.115 | uvicorn ^0.32, Starlette ^0.40 | FastAPI pins Starlette internally |
| typer ^0.15 | Click ^8.1 | Typer wraps Click, both versions must be compatible |
| pytest-asyncio ^0.24 | pytest ^8.0 | Use `asyncio_mode = "auto"` to avoid per-test decorators |

**IMPORTANT: All versions above are from training data (May 2025). Before locking dependencies, run:**
```bash
uv add --dry-run <package>  # to see latest available version
```

## *ARR Services Docker Images

These are the container images ARRmada will deploy (not its own deps, but critical to document):

| Service | Image | Default Port | API Docs |
|---------|-------|-------------|----------|
| Sonarr | `lscr.io/linuxserver/sonarr:latest` | 8989 | `/api/v3` |
| Radarr | `lscr.io/linuxserver/radarr:latest` | 7878 | `/api/v3` |
| Prowlarr | `lscr.io/linuxserver/prowlarr:latest` | 9696 | `/api/v1` |
| qBittorrent | `lscr.io/linuxserver/qbittorrent:latest` | 8080 | WebUI API |
| Jellyfin | `lscr.io/linuxserver/jellyfin:latest` | 8096 | `/api` (custom) |
| Jellyseerr | `fallenbagel/jellyseerr:latest` | 5055 | `/api/v1` |
| Bazarr | `lscr.io/linuxserver/bazarr:latest` | 6767 | `/api` |
| Recyclarr | `ghcr.io/recyclarr/recyclarr:latest` | N/A (cron job) | CLI |
| Gluetun | `qmcgaw/gluetun:latest` | N/A (VPN) | REST control |
| Huntarr | `ghcr.io/plexguide/huntarr:latest` | 9705 | Web UI |
| Tdarr | `ghcr.io/haveagitgat/tdarr:latest` | 8265 | API |
| Byparr | `ghcr.io/ThePhaseless/Byparr:latest` | 8191 | FlareSolverr-compatible |

**Note on LinuxServer.io images:** Prefer `lscr.io/linuxserver/*` for Sonarr, Radarr, Prowlarr, Bazarr, qBittorrent, Jellyfin. These images have standardized PUID/PGID handling, consistent config paths (`/config`), and are the community standard. Hotio images are a valid alternative but less common.

## Key Integration Details

### API Key Extraction Pattern
```python
# *ARR services store API keys in /config/config.xml after first boot
import xml.etree.ElementTree as ET

async def get_api_key(service_name: str, config_path: str) -> str:
    """Read API key from *ARR service config.xml."""
    tree = ET.parse(f"{config_path}/{service_name}/config.xml")
    return tree.find(".//ApiKey").text
```

### qBittorrent Default Credentials
qBittorrent generates a random password on first boot (since v4.6.1). The temporary password is logged to stdout. Strategy:
1. Read password from container logs (`docker logs qbittorrent`)
2. Or set `WEBUI_PASSWORD` env var in compose to control initial password
3. Configure via qBittorrent WebUI API (no REST API like *ARRs -- it's form-based)

### Jellyseerr Initial Setup
Jellyseerr requires an interactive first-run wizard (select Jellyfin, provide URL + credentials). This cannot be fully automated via API on first boot -- it needs the initial setup endpoint to be called in sequence:
1. POST `/api/v1/auth/jellyfin` (authenticate)
2. POST `/api/v1/settings/jellyfin` (configure server)
3. POST `/api/v1/settings/sonarr` (add Sonarr)
4. POST `/api/v1/settings/radarr` (add Radarr)
5. POST `/api/v1/settings/initialize` (complete setup)

**This is the hardest service to auto-configure.** Needs phase-specific research.

## Sources

- Project plan: `ARRmada_PROJECT_PLAN.md` -- established tech decisions
- Ecosystem analysis: `ARR_Ecosysteme_Alternatives_Detaillees.md` -- competitor landscape (Feb 2026)
- Training data (May 2025) -- library versions, API patterns. **Versions flagged as MEDIUM confidence.**
- *ARR API documentation is available at `<service_url>/swagger` for Sonarr/Radarr/Prowlarr (Swagger UI built in).

---
*Stack research for: ARRmada -- *ARR media stack installer/auto-configurator*
*Researched: 2026-02-22*
