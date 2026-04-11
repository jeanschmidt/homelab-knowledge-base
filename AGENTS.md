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
    │── # Web Frameworks
    ├── tornado/
    │
    │── # Reverse Proxy
    ├── nginx/
    ├── traefik/
    │
    │── # Container Management
    ├── portainer/
    ├── watchtower/
    │
    │── # Home Automation
    ├── homeassistant/         # Home Assistant (home-assistant/core)
    ├── mosquitto/             # Eclipse Mosquitto MQTT broker
    ├── winix/                 # Winix air purifier HA integration
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
    ├── sabnzbd/
    │
    │── # Media Stack - Automation (*arr)
    ├── Prowlarr/
    ├── Sonarr/
    ├── Radarr/
    ├── Lidarr/
    ├── bazarr/
    │
    │── # Media Stack - Requests & Discovery
    ├── seerr/
    │
    │── # Media Stack - Transcoding & Playback
    ├── Tdarr/
    ├── pms-docker/            # Plex Media Server Docker
    │
    │── # Graph Visualization
    ├── pydot/
    ├── graphviz/
    │
    │── # Media Stack - Jellyfin
    ├── jellyfin-web/
    ├── jellyfin-ffmpeg/
    ├── jellyfin-meta/
    ├── jellyfin-chromecast/
    ├── jellyfin-androidtv/
    ├── jellyfin-ios/
    ├── Swiftfin/
    ├── jellyfin-sdk-typescript/
    └── jellyfin-android/
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

### Web Frameworks

#### `repos/tornado/` - Tornado
**Language:** Python | **Version:** v6.5.5 | **Org:** tornadoweb

Python web framework and asynchronous networking library. Provides non-blocking I/O, coroutines, WebSockets, and a built-in HTTP server/client.

**Key paths:**
- `tornado/web.py` - Request handlers, application routing, and web framework core
- `tornado/ioloop.py` - Core I/O event loop (integrates with asyncio)
- `tornado/httpserver.py` - Non-blocking HTTP server
- `tornado/httpclient.py` - Async/sync HTTP client
- `tornado/websocket.py` - WebSocket server and client implementation
- `tornado/template.py` - Template engine
- `tornado/gen.py` - Generator-based coroutine utilities
- `tornado/iostream.py` - Non-blocking socket I/O wrappers
- `tornado/tcpserver.py` - Non-blocking TCP server base class
- `tornado/auth.py` - Third-party authentication (OAuth, OpenID)
- `tornado/routing.py` - URL routing
- `demos/` - Example applications (blog, chat, WebSocket, etc.)
- `docs/` - Sphinx documentation source

**Relevant for:** Async web services, WebSocket servers, non-blocking HTTP clients, event-driven networking, building lightweight API servers.

---

### Reverse Proxy

#### `repos/nginx/` - Nginx
**Language:** C | **Version:** release-1.29.5 | **Org:** nginx

High-performance HTTP server, reverse proxy, and load balancer. The reference web server used across countless deployments.

**Key paths:**
- `src/core/` - Core event loop and process management
- `src/http/` - HTTP protocol implementation and modules
- `src/http/modules/` - Built-in modules (proxy, upstream, ssl, rewrite, etc.)
- `src/stream/` - TCP/UDP stream proxy modules
- `conf/` - Default configuration files (`nginx.conf`, `mime.types`)
- `contrib/` - Vim syntax highlighting and other contrib tools
- `docs/` - XML documentation source

**Relevant for:** Reverse proxy configuration, upstream load balancing, SSL/TLS termination, location block routing, rate limiting, caching, stream (TCP/UDP) proxying.

---

#### `repos/traefik/` - Traefik
**Language:** Go | **Version:** v3.6.13 | **Org:** traefik

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

#### `repos/mosquitto/` - Eclipse Mosquitto
**Language:** C | **Version:** v2.1.2 | **Org:** eclipse-mosquitto

Lightweight open-source MQTT message broker. Core messaging backbone for IoT devices communicating with Home Assistant.

**Key paths:**
- `src/` - Broker daemon source code
- `lib/` - Client library (libmosquitto)
- `plugins/` - Authentication and access control plugins
- `config.h` - Compile-time configuration
- `mosquitto.conf` - Default configuration file with all options documented
- `man/` - Man pages (mosquitto.conf.5 is the config reference)
- `examples/` - Example client implementations
- `docker/` - Dockerfile and Docker entrypoint

**Relevant for:** MQTT broker configuration, TLS/SSL setup, authentication (password file, plugin-based), ACL rules, bridge configuration for connecting multiple brokers, WebSocket support, client library usage.

---

#### `repos/winix/` - Winix Air Purifier Integration
**Language:** Python | **Version:** v1.3.1 | **Org:** iprak

Home Assistant custom integration for controlling and monitoring Winix air purifiers (C545, C610, AM90, HR1000, C909, T800, etc.). Exposes fan entities with speed/preset modes, air quality sensors (AQI, PM2.5), filter life sensors, and PlasmaWave toggle.

**Key paths:**
- `custom_components/winix/` - Integration source code
- `custom_components/winix/fan.py` - Fan entity (speed, preset modes, PlasmaWave)
- `custom_components/winix/sensor.py` - Air quality and filter life sensors
- `custom_components/winix/driver.py` - Winix API client driver
- `custom_components/winix/manager.py` - Device management and polling
- `tests/` - Test suite

**Relevant for:** Winix air purifier control via Home Assistant, air quality monitoring (AQI/PM2.5), custom HA integration patterns, fan entity configuration.

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

#### `repos/sabnzbd/` - SABnzbd
**Language:** Python | **Version:** 4.5.5 | **Org:** sabnzbd

Usenet binary newsreader/downloader. Automates downloading from Usenet via NZB files, integrates with Sonarr, Radarr, and Lidarr as a download client.

**Key paths:**
- `sabnzbd/` - Main application code
- `interfaces/` - Web UI templates
- `SABnzbd.py` - Application entry point
- `tests/` - Test suite

**Relevant for:** Usenet download client configuration, NZB processing, *arr download client integration, category/folder mapping.

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

### Media Stack - Requests & Discovery

#### `repos/seerr/` - Seerr
**Language:** TypeScript (Next.js) | **Version:** v3.1.0 | **Org:** seerr-team

Media request and discovery tool (successor to Overseerr). Lets users browse, request, and manage movies/TV shows with automatic routing to Sonarr and Radarr.

**Key paths:**
- `server/` - Backend server logic
- `src/` - Frontend (Next.js/React)
- `seerr-api.yml` - OpenAPI spec
- `docs/` - Documentation
- `config/` - Configuration files

**Relevant for:** Media request workflows, Sonarr/Radarr integration, user access control, notification configuration.

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

### Media Stack - Jellyfin

#### `repos/jellyfin-web/` - Jellyfin Web Client
**Language:** TypeScript | **Version:** v10.11.6 | **Org:** jellyfin

The main web UI for Jellyfin media server. Single-page app built with Vite/Webpack.

**Key paths:**
- `src/` - Application source (components, controllers, plugins)

**Relevant for:** Customizing the web client, understanding playback UI, plugin integration.

---

#### `repos/jellyfin-ffmpeg/` - Jellyfin FFmpeg
**Language:** C | **Version:** v7.1.3-3 | **Org:** jellyfin

Custom FFmpeg fork with Jellyfin-specific patches for hardware-accelerated transcoding (VAAPI, QSV, NVENC, V4L2, AMF).

**Key paths:**
- `debian/` - Debian packaging
- `libavcodec/` - Codec implementations
- `libavfilter/` - Filter implementations (tone-mapping, subtitles, etc.)
- `docker-build.sh` - Docker-based build system

**Relevant for:** Understanding hardware transcoding capabilities, codec support, build configuration for specific hardware.

---

#### `repos/jellyfin-meta/` - Jellyfin Meta
**Language:** Markdown | **Tracking:** Latest | **Org:** jellyfin

Organizational meta-repository containing team info, policies, procedures, and API design docs.

**Key paths:**
- `api-design/` - API design documents
- `policies-and-procedures/` - Project governance

**Relevant for:** Understanding Jellyfin API conventions and project policies.

---

#### `repos/jellyfin-chromecast/` - Jellyfin Chromecast
**Language:** TypeScript | **Version:** v1.2.0 | **Org:** jellyfin

Chromecast receiver app for casting media from Jellyfin clients.

**Key paths:**
- `src/` - Receiver application source

**Relevant for:** Chromecast playback behavior, cast protocol integration.

---

#### `repos/jellyfin-androidtv/` - Jellyfin Android TV
**Language:** Kotlin | **Version:** v0.19.7 | **Org:** jellyfin

Dedicated Android TV client for Jellyfin with leanback UI.

**Key paths:**
- `app/` - Main application module
- `playback/` - Playback engine module
- `preference/` - Settings/preferences module

**Relevant for:** Android TV playback configuration, ExoPlayer integration, leanback UI customization.

---

#### `repos/jellyfin-ios/` - Jellyfin iOS (React Native)
**Language:** TypeScript (React Native) | **Version:** v1.7.0.8 | **Org:** jellyfin

Legacy React Native iOS client for Jellyfin.

**Key paths:**
- `screens/` - Screen components
- `components/` - Reusable UI components
- `stores/` - State management
- `ios/` - Native iOS project files

**Relevant for:** Legacy iOS client behavior (see Swiftfin for the modern native client).

---

#### `repos/Swiftfin/` - Swiftfin
**Language:** Swift | **Version:** 1.4 | **Org:** jellyfin

Modern native Apple client for Jellyfin (iOS, iPadOS, tvOS). Replaces the older React Native client.

**Key paths:**
- `Shared/` - Shared code across platforms
- `Swiftfin/` - iOS/iPadOS target
- `Swiftfin tvOS/` - tvOS target
- `PreferencesView/` - Settings UI

**Relevant for:** Native Apple platform playback, VLCKit integration, multi-platform Swift architecture.

---

#### `repos/jellyfin-sdk-typescript/` - Jellyfin TypeScript SDK
**Language:** TypeScript | **Version:** v0.13.0 | **Org:** jellyfin

Auto-generated TypeScript/JavaScript SDK for the Jellyfin API. Used by jellyfin-web and third-party integrations.

**Key paths:**
- `src/` - Generated API client and models
- `openapi.json` - OpenAPI spec the SDK is generated from

**Relevant for:** Jellyfin API reference (the OpenAPI spec is the authoritative source), building custom integrations.

---

#### `repos/jellyfin-android/` - Jellyfin Android
**Language:** Kotlin | **Version:** v2.6.3 | **Org:** jellyfin

Native Android mobile client for Jellyfin.

**Key paths:**
- `app/` - Main application module

**Relevant for:** Android playback configuration, ExoPlayer integration, WebView-based UI with native playback.

---

### Graph Visualization

#### `repos/pydot/` - pydot
**Language:** Python | **Version:** v4.0.1 | **Org:** pydot

Python interface to Graphviz's DOT language. Allows creating, reading, and writing DOT graph descriptions and rendering them to images via Graphviz.

**Key paths:**
- `src/pydot/core.py` - Core graph, node, and edge classes
- `src/pydot/dot_parser.py` - DOT language parser
- `src/pydot/classes.py` - Graph class hierarchy
- `src/pydot/exceptions.py` - Exception definitions
- `test/` - Test suite

**Relevant for:** Programmatically generating graph diagrams, parsing DOT files, rendering infrastructure topology or dependency visualizations.

---

#### `repos/graphviz/` - graphp/graphviz
**Language:** PHP | **Version:** v0.2.2 | **Org:** graphp

PHP library for reading and writing Graphviz DOT graph files and rendering them to images.

**Key paths:**
- `src/GraphViz.php` - Main GraphViz renderer (invokes `dot` binary)
- `src/Dot.php` - DOT file format writer
- `src/Image.php` - Image file handling
- `examples/` - Usage examples (simple graphs, UML, HTML labels, records)
- `tests/` - Test suite

**Relevant for:** PHP-based graph visualization, generating DOT files, rendering graphs to PNG/SVG from PHP applications.

---

## Managing Repositories

See `CLAUDE.md` for the full step-by-step procedure. Summary below.

### Adding a New Repository

All of these steps are required — do not skip any:

1. **Look up latest release tag** — `gh api repos/ORG/REPO/releases/latest --jq '.tag_name'`
2. **Add to `sync.py`** — Add entry to `ALLOWED_REPOS` in the appropriate section. Pin to latest release; use bare string if no releases exist.
3. **Add git submodule** — `git submodule add <url> repos/<name>`
4. **Run `uv run sync.py`** — Verifies allowlist, checks out pinned versions.
5. **Update this file (`AGENTS.md`)** — Three places:
   - Directory tree (under `repos/`)
   - Repository summary (language, version, key paths, relevance)
   - Common Tasks table (if applicable)
6. **Verify** — `uv run sync.py --dry-run` to confirm everything is in sync.

### Allowlist entry formats

```python
"org/repo-name"                         # Track latest default branch
("org/repo-name", "v1.0.0")            # Pin to a tag/commit
("org/repo-name", "v1.0.0", "alias")   # Pin with custom local directory name
("org/repo-name", None, "alias")        # Track latest with custom local directory name
```

### Updating to Latest Versions

```bash
gh api repos/ORG/REPO/releases/latest --jq '.tag_name'
```

Update the version in `sync.py`, run `uv run sync.py`, and update the version in the AGENTS.md summary.

### Removing a Repository

Remove from `ALLOWED_REPOS` in `sync.py`, run `uv run sync.py` (handles submodule cleanup), and remove from all three places in this file (directory tree, summary, common tasks table).

---

## Common Tasks

| Task | Repository | Key File/Path |
|------|------------|---------------|
| Debug Ansible module behavior | `ansible` | `lib/ansible/modules/` |
| Understand compose file syntax | `compose` | `pkg/compose/` |
| Configure Nginx reverse proxy | `nginx` | `conf/`, `src/http/modules/` |
| Configure Traefik routing | `traefik` | `pkg/provider/file/`, `docs/content/routing/` |
| Write PromQL queries | `prometheus` | `promql/`, `documentation/` |
| Find available host metrics | `node_exporter` | `collector/` |
| Find container metric names | `cadvisor` | `metrics/prometheus/` |
| Build Grafana dashboards | `grafana` | `public/app/plugins/`, `docs/sources/dashboards/` |
| Write LogQL queries | `loki` | `pkg/logql/`, `docs/sources/` |
| Configure Promtail scraping | `loki` | `clients/pkg/promtail/` |
| Set up VPN provider | `gluetun` | `internal/provider/` |
| Configure Usenet downloads | `sabnzbd` | `sabnzbd/`, `interfaces/` |
| Configure *arr download paths | `Sonarr` | `src/NzbDrone.Core/MediaFiles/` |
| Understand import/move behavior | `Sonarr`/`Radarr` | `src/NzbDrone.Core/MediaFiles/EpisodeImport/` |
| Set up transcode rules | `Tdarr` | `Tdarr_Plugins/` |
| Configure Plex Docker | `pms-docker` | `README.md` |
| Configure MQTT broker | `mosquitto` | `mosquitto.conf`, `plugins/` |
| Control Winix air purifiers | `winix` | `custom_components/winix/fan.py`, `custom_components/winix/driver.py` |
| Monitor air quality (AQI/PM2.5) | `winix` | `custom_components/winix/sensor.py` |
| Check HA metric naming | `homeassistant` | `homeassistant/components/prometheus/` |
| OPNsense node_exporter setup | `opnsense-plugins` | `net-mgmt/node_exporter/` |
| Manage media requests | `seerr` | `server/`, `seerr-api.yml` |
| Jellyfin web UI customization | `jellyfin-web` | `src/` |
| Jellyfin transcoding/codec support | `jellyfin-ffmpeg` | `libavcodec/`, `libavfilter/` |
| Jellyfin API reference | `jellyfin-sdk-typescript` | `openapi.json` |
| Jellyfin Android TV playback | `jellyfin-androidtv` | `playback/`, `app/` |
| Jellyfin iOS/tvOS (Swiftfin) | `Swiftfin` | `Shared/`, `Swiftfin/` |
| Build async web services | `tornado` | `tornado/web.py`, `tornado/ioloop.py` |
| WebSocket server/client | `tornado` | `tornado/websocket.py` |
| Non-blocking HTTP client | `tornado` | `tornado/httpclient.py` |
| Generate DOT graphs (Python) | `pydot` | `src/pydot/core.py` |
| Parse DOT files (Python) | `pydot` | `src/pydot/dot_parser.py` |
| Render graphs to images (PHP) | `graphviz` | `src/GraphViz.php`, `examples/` |

## Tips for Navigating the Codebase

1. **Start with READMEs** - Each `repos/*/README.md` has setup instructions and configuration reference
2. **Check docs directories** - Most repos have `docs/` with comprehensive guides
3. **Search across repos** - `grep -r "pattern" repos/` to find implementations across all projects
4. **Check configuration schemas** - The *arr apps share a common architecture; patterns in one apply to others
5. **Look at Docker examples** - Many repos include `docker-compose.yml` examples in their docs
