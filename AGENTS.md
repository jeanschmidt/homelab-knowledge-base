# Homelab Knowledge Base - Agent Guide

This repository is a curated collection of homelab infrastructure, monitoring, and media stack open-source projects, organized as git submodules. It serves as a knowledge base for AI agents and developers working with the IoTBase homelab.

## Quick Start

```bash
# Sync all repositories (adds missing, updates existing)
uv run sync.py

# Preview changes without making them
uv run sync.py --dry-run
```

## Repository Structure

```
homelab-knowledge-base/
├── sync.py              # Repository sync tool
├── pyproject.toml       # Python project config
├── AGENTS.md            # This file
└── repos/               # Submodules directory
    │
    │── # Infrastructure & Tools
    ├── ansible/
    ├── compose/
    │
    │── # Reverse Proxy
    ├── traefik/
    │
    │── # Container Management
    ├── portainer/
    ├── watchtower/
    │
    │── # Home Automation
    ├── homeassistant/         # Home Assistant (home-assistant/core)
    │
    │── # Monitoring Stack
    ├── prometheus/
    ├── node_exporter/
    ├── cadvisor/
    ├── grafana/
    ├── loki/                  # Includes Promtail
    ├── ntfy/
    │
    │── # Firewall
    ├── opnsense-core/         # OPNsense core (opnsense/core)
    ├── opnsense-plugins/      # OPNsense plugins (opnsense/plugins)
    │
    │── # Media Stack - VPN & Downloads
    ├── gluetun/
    ├── qBittorrent/
    │
    │── # Media Stack - Automation (*arr)
    ├── Prowlarr/
    ├── Sonarr/
    ├── Radarr/
    ├── Lidarr/
    ├── bazarr/
    │
    │── # Media Stack - Transcoding & Playback
    ├── Tdarr/
    └── pms-docker/            # Plex Media Server Docker
```

**Note on naming:** Submodule directory names usually match the GitHub repository name. Where repos share names across orgs, custom local names are used (e.g., `homeassistant` for `home-assistant/core`, `opnsense-core` for `opnsense/core`). Check `.gitmodules` to see which org each belongs to.

## Repository Summaries

### Infrastructure & Tools

#### `repos/ansible/` - Ansible
**Language:** Python | **Version:** v2.20.2 | **Org:** ansible

IT automation engine for configuration management, application deployment, and orchestration. Used to manage the entire IoTBase server declaratively.

**Key paths:**
- `lib/ansible/modules/` - Built-in module implementations
- `lib/ansible/plugins/` - Plugin system (connection, callback, filter, etc.)
- `docs/` - Documentation source

**Relevant for:** Understanding module behavior, writing custom filters, debugging Ansible tasks.

---

#### `repos/compose/` - Docker Compose
**Language:** Go | **Version:** v5.0.2 | **Org:** docker

Multi-container Docker application orchestration. All IoTBase services are deployed as Docker Compose stacks.

**Key paths:**
- `cmd/compose/` - CLI command implementations
- `pkg/compose/` - Core compose logic (up, down, build, etc.)
- `docs/` - Reference documentation

**Relevant for:** Understanding compose file syntax, debugging container orchestration, service dependency behavior.

---

### Reverse Proxy

#### `repos/traefik/` - Traefik
**Language:** Go | **Version:** v3.6.8 | **Org:** traefik

Cloud-native reverse proxy and load balancer. Handles all HTTP/HTTPS routing for the homelab using file-based dynamic configuration.

**Key paths:**
- `pkg/provider/file/` - File provider (used by IoTBase)
- `pkg/middlewares/` - Middleware implementations (stripprefix, headers, etc.)
- `pkg/tls/` - TLS certificate handling
- `docs/content/` - Documentation (routing, middlewares, providers)

**Relevant for:** Configuring routing rules, understanding middleware chains, TLS setup, debugging request routing.

---

### Container Management

#### `repos/portainer/` - Portainer CE
**Language:** Go/TypeScript | **Version:** 2.33.7 | **Org:** portainer

Docker management UI for managing containers, images, networks, and volumes through a web interface.

**Key paths:**
- `api/` - Backend API (Go)
- `app/` - Frontend (TypeScript/React)

---

#### `repos/watchtower/` - Watchtower
**Language:** Go | **Version:** v1.7.1 | **Org:** containrrr

Automatically updates running Docker containers when new images are available. Runs daily at 4 AM on IoTBase.

**Key paths:**
- `pkg/container/` - Container update logic
- `docs/` - Configuration reference

**Relevant for:** Understanding update strategies, notification configuration, container filtering.

---

### Home Automation

#### `repos/homeassistant/` - Home Assistant Core
**Language:** Python | **Version:** 2026.2.2 | **Org:** home-assistant

Smart home automation platform. Runs as the primary IoT controller on IoTBase, accessed directly on port 8123 (no subpath proxy support).

**Key paths:**
- `homeassistant/components/` - All integrations (1000+)
- `homeassistant/components/prometheus/` - Prometheus metrics exporter
- `homeassistant/components/climate/` - Climate/HVAC entities
- `homeassistant/components/rest/` - REST API integration

**Relevant for:** Understanding entity state models, Prometheus metric naming conventions (`homeassistant_climate_*`), automation trigger/action schemas, REST API endpoints.

---

### Monitoring Stack

#### `repos/prometheus/` - Prometheus
**Language:** Go | **Version:** v3.9.1 | **Org:** prometheus

Time-series metrics database. Scrapes metrics from Node Exporter, cAdvisor, Traefik, Home Assistant, and OPNsense.

**Key paths:**
- `config/` - Configuration parsing and validation
- `scrape/` - Scrape manager and target discovery
- `promql/` - PromQL query engine
- `web/` - HTTP API (`/api/v1/query`, `/api/v1/series`, etc.)
- `documentation/` - Metric types, configuration reference

**Relevant for:** Writing PromQL queries, understanding scrape behavior, configuring scrape targets, API usage for debugging.

---

#### `repos/node_exporter/` - Prometheus Node Exporter
**Language:** Go | **Version:** v1.10.2 | **Org:** prometheus

Exposes host-level metrics (CPU, memory, disk, network, filesystem). Runs on IoTBase (`:9100`) and OPNsense (`192.168.30.1:9100`).

**Key paths:**
- `collector/` - Individual metric collectors (cpu, meminfo, diskstats, etc.)

**Relevant for:** Understanding available host metrics and their label schemas for Grafana dashboards.

---

#### `repos/cadvisor/` - cAdvisor
**Language:** Go | **Version:** v0.56.2 | **Org:** google

Container-level metrics (CPU, memory, network, filesystem per container). Scraped by Prometheus.

**Key paths:**
- `metrics/prometheus/` - Prometheus metric definitions
- `container/` - Container runtime integrations

**Relevant for:** Understanding `container_*` metric naming for Grafana dashboards.

---

#### `repos/grafana/` - Grafana
**Language:** Go/TypeScript | **Version:** v12.3.3 | **Org:** grafana

Dashboard and visualization platform. Accessed at `/grafana/` via Traefik. Connects to Prometheus and Loki as datasources.

**Key paths:**
- `pkg/api/` - Backend API
- `public/app/plugins/` - Built-in panel plugins (timeseries, gauge, stat, table, etc.)
- `pkg/services/alerting/` - Alerting engine
- `docs/sources/` - Documentation (dashboards, panels, alerting)
- `pkg/services/provisioning/` - Provisioning system (datasources, dashboards)

**Relevant for:** Dashboard JSON model schema, panel configuration options, alerting rule syntax, provisioning file format.

---

#### `repos/loki/` - Grafana Loki
**Language:** Go | **Version:** v3.6.6 | **Org:** grafana

Log aggregation system (like Prometheus but for logs). Includes Promtail (log shipping agent).

**Key paths:**
- `pkg/logql/` - LogQL query engine
- `clients/pkg/promtail/` - Promtail source code
- `docs/sources/` - Documentation (LogQL, configuration)

**Relevant for:** Writing LogQL queries, configuring Promtail scrape targets, understanding log label extraction.

---

#### `repos/ntfy/` - ntfy
**Language:** Go | **Version:** v2.17.0 | **Org:** binwiederhier

Push notification server. Used as the notification endpoint for Grafana alerts, accessed at `/ntfy/` via Traefik.

**Key paths:**
- `server/` - Server implementation
- `docs/` - API reference, configuration

**Relevant for:** Notification payload format, topic configuration, Grafana webhook integration.

---

### Firewall

#### `repos/opnsense-core/` - OPNsense Core
**Language:** PHP/Python | **Tracking:** Latest | **Org:** opnsense

Firewall and routing platform GUI, API, and backend. Runs on the network gateway at `192.168.30.1`.

**Key paths:**
- `src/opnsense/mvc/` - MVC framework and controllers
- `src/opnsense/www/` - Web interface
- `src/etc/inc/` - Core system includes

**Relevant for:** Understanding OPNsense API for automation, firewall rule management.

---

#### `repos/opnsense-plugins/` - OPNsense Plugins
**Language:** PHP/Shell | **Tracking:** Latest | **Org:** opnsense

Plugin collection for OPNsense. Includes the node_exporter plugin used for Prometheus scraping.

**Key paths:**
- `net-mgmt/node_exporter/` - Prometheus node_exporter plugin
- `net/haproxy/` - HAProxy plugin
- `security/acme-client/` - ACME/Let's Encrypt plugin

**Relevant for:** Understanding the node_exporter deployment on OPNsense, available plugins for network monitoring.

---

### Media Stack - VPN & Downloads

#### `repos/gluetun/` - Gluetun
**Language:** Go | **Version:** v3.41.1 | **Org:** qdm12

VPN client in a thin Docker container supporting multiple VPN providers (Mullvad, ProtonVPN, AirVPN, etc.) with OpenVPN or WireGuard. Other containers route traffic through it via `network_mode: "service:gluetun"`.

**Key paths:**
- `internal/provider/` - VPN provider implementations
- `internal/configuration/` - Configuration parsing
- `internal/firewall/` - Built-in kill switch
- `internal/dns/` - DNS over TLS

**Relevant for:** VPN provider configuration, port forwarding setup, understanding the kill switch mechanism, DNS leak prevention.

---

#### `repos/qBittorrent/` - qBittorrent
**Language:** C++ | **Version:** release-5.1.4 | **Org:** qbittorrent

BitTorrent client with web UI. Routes all traffic through Gluetun's VPN tunnel.

**Key paths:**
- `src/webui/` - Web UI implementation
- `src/base/bittorrent/` - BitTorrent engine
- `src/base/rss/` - RSS feed support

**Relevant for:** Web API endpoints for *arr integration, configuration options, download path management.

---

### Media Stack - Automation (*arr)

#### `repos/Prowlarr/` - Prowlarr
**Language:** C# | **Version:** v2.3.0.5236 | **Org:** Prowlarr

Indexer manager/proxy that integrates with Sonarr, Radarr, and Lidarr. Manages torrent tracker and Usenet indexer configurations in one place.

**Key paths:**
- `src/NzbDrone.Core/Indexers/` - Indexer implementations
- `src/Prowlarr.Api.V1/` - API endpoints
- `frontend/` - React web UI

**Relevant for:** Indexer configuration, API for syncing indexers to other *arr apps.

---

#### `repos/Sonarr/` - Sonarr
**Language:** C# | **Version:** v4.0.16.2944 | **Org:** Sonarr

TV show automation — monitors, downloads, and organizes TV series.

**Key paths:**
- `src/NzbDrone.Core/MediaFiles/` - File management and importing
- `src/NzbDrone.Core/Download/` - Download client integration
- `src/Sonarr.Api.V3/` - API endpoints
- `frontend/` - React web UI

**Relevant for:** Understanding import/move behavior (critical for NAS path mapping), quality profiles, download client configuration, API for automation.

---

#### `repos/Radarr/` - Radarr
**Language:** C# | **Version:** v6.0.4.10291 | **Org:** Radarr

Movie automation — monitors, downloads, and organizes movies. Same architecture as Sonarr.

**Key paths:**
- `src/NzbDrone.Core/MediaFiles/` - File management and importing
- `src/NzbDrone.Core/Download/` - Download client integration
- `src/Radarr.Api.V3/` - API endpoints

**Relevant for:** Same as Sonarr but for movies. Import behavior, quality profiles, path mapping.

---

#### `repos/Lidarr/` - Lidarr
**Language:** C# | **Version:** v3.1.0.4875 | **Org:** Lidarr

Music automation — monitors, downloads, and organizes music. Same architecture as Sonarr/Radarr.

**Key paths:**
- `src/NzbDrone.Core/MediaFiles/` - File management
- `src/Lidarr.Api.V1/` - API endpoints

**Relevant for:** Music library organization, metadata handling, import behavior.

---

#### `repos/bazarr/` - Bazarr
**Language:** Python | **Version:** v1.5.5 | **Org:** morpheus65535

Subtitle manager that integrates with Sonarr and Radarr. Automatically downloads subtitles based on configured preferences.

**Key paths:**
- `bazarr/` - Main application code
- `bazarr/api/` - API endpoints
- `frontend/` - React web UI

**Relevant for:** Subtitle provider configuration, Sonarr/Radarr integration, language preferences.

---

### Media Stack - Transcoding & Playback

#### `repos/Tdarr/` - Tdarr
**Language:** JavaScript/TypeScript | **Tracking:** Latest | **Org:** HaveAGitGat

Distributed transcode automation using FFmpeg/HandBrake. Watches library folders and batch-converts media to target formats for direct play compatibility.

**Key paths:**
- `Tdarr_Server/` - Server-side logic
- `Tdarr_Node/` - Worker node (can run on same or separate machines)
- `Tdarr_Plugins/` - Plugin system for transcode rules

**Relevant for:** Transcode plugin configuration, hardware acceleration setup (Intel Quick Sync via `/dev/dri`), worker/cache directory configuration, codec target profiles.

---

#### `repos/pms-docker/` - Plex Media Server (Docker)
**Language:** Dockerfile/Shell | **Tracking:** Latest | **Org:** plexinc

Official Docker image for Plex Media Server. Serves media to clients with automatic format negotiation.

**Key paths:**
- `root/` - Container filesystem overlay
- `README.md` - Environment variables and configuration reference

**Relevant for:** Docker volume mappings, environment variables (`PLEX_CLAIM`, `PLEX_UID`, `PLEX_GID`), hardware transcoding setup (`/dev/dri` passthrough), network mode configuration.

---

## Managing Repositories

### Adding a New Repository

Edit `sync.py` and add to the `ALLOWED_REPOS` list:

```python
ALLOWED_REPOS: list[str | tuple[str, str]] = [
    # Existing repos...

    # Track latest:
    "org/repo-name",

    # Pin to a specific version:
    ("org/repo-name", "v1.0.0"),
]
```

Then run `uv run sync.py` to sync.

### Updating to Latest Versions

Check the latest release:

```bash
gh api repos/ORG/REPO/releases/latest --jq '.tag_name'
```

Update the version in `sync.py` and run `uv run sync.py`.

### Removing a Repository

Remove it from `ALLOWED_REPOS` in `sync.py` and run `uv run sync.py`. The submodule will be automatically cleaned up.

---

## Common Tasks

| Task | Repository | Key File/Path |
|------|------------|---------------|
| Debug Ansible module behavior | `ansible` | `lib/ansible/modules/` |
| Understand compose file syntax | `compose` | `pkg/compose/` |
| Configure Traefik routing | `traefik` | `pkg/provider/file/`, `docs/content/routing/` |
| Write PromQL queries | `prometheus` | `promql/`, `documentation/` |
| Find available host metrics | `node_exporter` | `collector/` |
| Find container metric names | `cadvisor` | `metrics/prometheus/` |
| Build Grafana dashboards | `grafana` | `public/app/plugins/`, `docs/sources/dashboards/` |
| Write LogQL queries | `loki` | `pkg/logql/`, `docs/sources/` |
| Configure Promtail scraping | `loki` | `clients/pkg/promtail/` |
| Set up VPN provider | `gluetun` | `internal/provider/` |
| Configure *arr download paths | `Sonarr` | `src/NzbDrone.Core/MediaFiles/` |
| Understand import/move behavior | `Sonarr`/`Radarr` | `src/NzbDrone.Core/MediaFiles/EpisodeImport/` |
| Set up transcode rules | `Tdarr` | `Tdarr_Plugins/` |
| Configure Plex Docker | `pms-docker` | `README.md` |
| Check HA metric naming | `homeassistant` | `homeassistant/components/prometheus/` |
| OPNsense node_exporter setup | `opnsense-plugins` | `net-mgmt/node_exporter/` |

## Tips for Navigating the Codebase

1. **Start with READMEs** - Each `repos/*/README.md` has setup instructions and configuration reference
2. **Check docs directories** - Most repos have `docs/` with comprehensive guides
3. **Search across repos** - `grep -r "pattern" repos/` to find implementations across all projects
4. **Check configuration schemas** - The *arr apps share a common architecture; patterns in one apply to others
5. **Look at Docker examples** - Many repos include `docker-compose.yml` examples in their docs
