# API Reference (REST, WebSocket & SSE)

PVE Command Center exposes a typed FastAPI REST API alongside real-time **WebSocket** and **Server-Sent Events (SSE)** streams on port `8000` (or HTTPS via Tailscale Serve).

## Authentication & Headers

| Mode | Header / Query Parameter | Where Used |
|---|---|---|
| **Tailscale Serve SSO** | `Tailscale-User-Login` (verified on loopback `127.0.0.1` / `::1` when `tailscale_sso_enabled: true`) | Browser / Tailscale HTTPS (`*.ts.net`) |
| **API Bearer Token** | `Authorization: Bearer pve_<token>` | iPhone, iPad, Mac, Apple TV, Windows, Linux, Android, scripts |
| **Session JWT** | `Authorization: Bearer <jwt>` (24h HS256 JWT issued by `POST /api/auth/login`) | Browser / PIN login |
| **WebSocket / SSE Auth** | `?token=<api_token_or_jwt>` or `Authorization: Bearer <token>` | `/api/ws/events` and `/api/events/stream` |
| **Webhook Ingest Auth** | `Authorization: Bearer`, custom `auth_header_name`, `X-Webhook-Token`, `X-API-Key`, or `?token=` | `POST /api/custom-services/{service_id}/webhook` |

### Role-Based Access Control (RBAC)

Authenticated API tokens and sessions carry a `role` (`admin`, `operator`, `viewer`, `device`):

- **`admin`**: Full read and write access, including `/api/config/*`, `/api/auth/tokens`, `/api/setup/config`, `/api/discovery/qr`, users, and notifications configuration.
- **`operator` / `device`**: Read access plus operational write routes (`/api/pve/*`, `/api/ops/*`, `/api/incidents/*`, service probes, and `/api/notifications/devices`). Configuration edits and admin-only reads return `403`.
- **`viewer`**: Read-only telemetry (`GET`/`HEAD`, including `/metrics` and `/api/metrics/prometheus`) and push device registration; write routes return `403`.

### Public Exempt Endpoints

- `GET /api/health`
- `POST /api/auth/login`
- `/api/pairing/*` (PIN-gated + brute-force rate-limited)
- `GET /api/onboarding/status`
- `POST /api/custom-services/{service_id}/webhook` (verified per service `auth_type` & `auth_token`)
- Onboarding & first-run setup endpoints (`/api/onboarding*`, `/api/setup/status`, `/api/setup/test-service`, `/api/setup/config`, `/api/discovery/lan`) **only while `is_configured == false`**.

---

## Health & Real-Time Telemetry

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/health` | Public | Lightweight server healthcheck returning `status`, `version`, `server_id`, `server_name`, `is_configured`, and `config_hash` (12 chars). |
| `GET` | `/api/events/stream` | DualAuth / `?token=` | SSE stream (`text/event-stream`). Emits `: connected`, `: keepalive` every `25`s, and `data: <json>\n\n` events. |
| `WS` | `/api/ws/events` | DualAuth / `?token=` | WebSocket stream (closes with `4401` if unauthorized). Sends `ping` every `25`s, accepts `"pong"` or `{"type":"pong"}`. |

### Real-Time Event Envelope & Types

Every WebSocket and SSE message shares the same snake_case envelope:

```json
{
  "type": "alert_triggered",
  "timestamp": "2026-11-18T10:15:30Z",
  "payload": {
    "incident_id": "a1b2c3d4...",
    "service_id": "vaultwarden",
    "service_name": "Vaultwarden",
    "severity": "critical",
    "status": "down",
    "message": "HTTP 502",
    "latency_ms": 142
  }
}
```

Supported event `type` values: `ping`, `config_reloaded`, `service_status_changed`, `alert_triggered`, `alert_resolved`, `incident_acknowledged`, `pairing_completed`.

---

## Authentication, PIN & API Tokens

| Method | Path | Role | Description |
|---|---|---|---|
| `GET` | `/api/me` | Any auth | Returns current user identity (`login`, `name`, `auth_mode`, `role`). |
| `POST` | `/api/auth/login` | Public | Accepts `{ "pin": "..." }` or `{ "api_token": "..." }` (`token` alias supported). Returns `access_token` (24h HS256 JWT). Rate-limited: 10 failures / 60s (`429`). |
| `GET` | `/api/auth/tokens` | Admin | Lists API token metadata (`id`, `name`, `role`, `created_at`) without hashes. |
| `POST` | `/api/auth/tokens` | Admin | Creates a named token (`role`: `admin \| operator \| viewer \| device`), stores SHA-256 hash in `config.yaml`, returns plaintext `pve_...` once. |
| `DELETE` | `/api/auth/tokens/{token_id}` | Admin | Revokes and deletes a Bearer token by ID. |
| `PUT` | `/api/auth/pin` | Admin | Sets or updates the Admin PIN (`min_length: 4`, `max_length: 32`) stored hashed in `security.admin_pin_hash`. |

---

## Declarative Configuration & Onboarding

| Method | Path | Role | Description |
|---|---|---|---|
| `GET` | `/api/onboarding/status` | Public | Non-secret state: `is_configured`, `needs_onboarding`, `server_id`, `server_name`, `public_url`, `tailscale_sso_enabled`, `pin_configured`, `api_tokens_count`, `auth_methods`. |
| `POST` | `/api/onboarding/init` | Public if unconfigured, else Admin | Initializes `config/config.yaml`, optional Admin PIN, Proxmox/integrations/notifications, and mints the initial `api_token`. |
| `GET` | `/api/config/yaml` | Admin | Returns masked `yaml` text (preserving `${VAR}` references), `hash`, `is_synthesized`, `config_exists`, `parsed_summary`. |
| `POST` | `/api/config/validate` | Admin | Validates candidate YAML (`{ "yaml": "..." }`) without saving; returns `valid`, `errors`, `warnings`, `parsed_summary`. |
| `PUT` | `/api/config/yaml` | Admin | Validates, atomically writes `config/config.yaml`, and hot-reloads `AlertEngine`, `MDNSPublisher`, and `pve_http`. Returns `422` on errors. |
| `GET` | `/api/config/export` | Admin | Downloads masked `config.yaml` with `Content-Disposition: attachment; filename="config.yaml"`. |

---

## Setup Wizard & Connectivity Probes

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/setup/status` | First-run exempt / DualAuth | Returns `is_configured`, detected `tailscale_url`, `pairing_deep_link` (`pvecenter://connect?...`), sanitized `env`, `categories`, and `storage_mounts`. |
| `POST` | `/api/setup/test-service` | First-run exempt / DualAuth | Runs a live probe for `proxmox`, `jellyfin`, `jellyseerr`, `plex`, `radarr`, `sonarr`, `prowlarr`, or `qbittorrent`. |
| `PUT` | `/api/setup/config` | First-run exempt / DualAuth | Updates `env`, `categories`, and `storage_mounts` in `config.yaml` (or `.env`/`services.yaml`) and triggers hot-reload. |

---

## Bonjour Discovery & Apple TV Pairing

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/discovery/lan` | First-run exempt / DualAuth | Scans local mDNS (`timeout` `0.5..4.0`s, default `2.0`s) for `_pvecenter._tcp`, `_http._tcp`, `_ssh._tcp`, `_smb._tcp`. |
| `GET` | `/api/discovery/qr` | DualAuth | Returns `deep_link` (`pvecenter://connect?url=...&name=...&id=...`), `onboarding_link`, `public_url`, `lan_ip`, `port`, `mdns_service_type`. |
| `POST` | `/api/pairing/sessions` | Public | Apple TV registers ephemeral P-256 `public_key` + `device_name`; returns 6-digit `pin`, `session_id`, `expires_at` (`300`s TTL). |
| `GET` | `/api/pairing/sessions/{pin}` | Public (rate-limited) | iPhone retrieves receiver `public_key` and `device_name`. |
| `POST` | `/api/pairing/sessions/{pin}/complete` | Public (rate-limited) | iPhone uploads AES-256-GCM bundle (`sender_public_key`, `nonce`, `ciphertext`, `tag`); emits `pairing_completed` on `RealtimeBus`. |
| `GET` | `/api/pairing/sessions/{pin}/payload` | Public (rate-limited) | Apple TV consumes the encrypted bundle once (returns `425` while pending) and immediately deletes the session from memory. |

---

## Proxmox VE Cluster & Storage

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/pve/nodes` | Lists all Proxmox VE cluster nodes with `status`, `online`, `cpu_pct`, `mem_pct`, and `uptime`. |
| `GET` | `/api/pve/overview` | Optional `?node=<name> \| * \| all`: fetches Proxmox node or multi-node cluster metrics, automatic failover state (`failover_active`), storage pools, LXCs, QEMU VMs, SMART disks, and `/host_mnt` mounts. |
| `POST` | `/api/pve/{vm_type}/{vmid}/{action}` | Optional `?node=<name>`; executes `start`, `stop`, `reboot`, or `shutdown` on `vm_type` (`lxc` \| `qemu`). Blocks destructive actions when `vmid == panel_self_vmid`. |

---

## Categorized Services & Media Widgets

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/services` | Runs concurrent healthchecks across `categories` and `custom_services`. |
| `PUT` | `/api/services/items/{service_id}` | Updates `name`, `description`, `url`, `ping_url`, `icon`, `widget`, or `category`. |
| `POST` | `/api/services/items/{service_id}/move` | Moves a service `{ "direction": "up" \| "down" }` inside its category. |
| `POST` | `/api/services/items/{service_id}/test` | Runs an immediate healthcheck against saved or unsaved `url`/`ping_url`. |
| `DELETE` | `/api/services/items/{service_id}` | Removes a service card from its category. |
| `GET` | `/api/widgets/all` | Fetches all configured widgets (`jellyfin`, `plex`, `radarr`, `sonarr`, `prowlarr`, `qbittorrent`) concurrently. |
| `GET` | `/api/widgets/{widget_name}` | Fetches live telemetry for a single widget by name. |

---

## Custom Services, Probes & Webhook Ingest

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/custom-services` | Lists custom services enriched with live connector check status, latency, metrics, and persisted `StateStore` state. |
| `POST` | `/api/custom-services` | Creates a custom service (auto-slugs `id` if omitted) and persists to `config.yaml` (`201 Created`). |
| `POST` | `/api/custom-services/reorder` | Reorders custom services by `service_ids` (`ids` / `order` aliases supported). |
| `PUT` | `/api/custom-services/{service_id}` | Updates custom service fields while preserving existing `auth_token` if blank/masked (`********`). |
| `DELETE` | `/api/custom-services/{service_id}` | Deletes a custom service from `config.yaml`. |
| `POST` | `/api/custom-services/{service_id}/test` | Executes a live dry-run probe on saved config or unsaved override payload. |
| `POST` | `/api/custom-services/{service_id}/webhook` | Public heartbeat ingest (`status`: `up \| degraded \| down`, `message`, `ttl_seconds`, `metrics`), updates `StateStore` and triggers `AlertEngine.evaluate_service()`. |

---

## Incidents & Notifications

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/incidents` | Lists SQLite incidents (`?active_only=true&limit=50&service_id=...`), `active_count`, and per-service `service_states`. |
| `POST` | `/api/incidents/{incident_id}/ack` | Marks an incident acknowledged and publishes `incident_acknowledged`. |
| `POST` | `/api/incidents/services/{service_id}/mute` | Mutes alerts for `minutes` (`1..43200`, default `60`). |
| `DELETE` | `/api/incidents/services/{service_id}/mute` | Clears an active mute window for `service_id`. |
| `POST` | `/api/notifications/devices` | Registers an APNs device token (`device_token`, `platform`: `ios \| ipados \| tvos \| macos`, `device_name`, `app_version`, `environment`). |
| `DELETE` | `/api/notifications/devices/{token}` | Unregisters an APNs device token. |
| `GET` | `/api/notifications/channels` | Returns APNs status (`configured`, `private_key_exists`, `devices`) and webhooks with secrets masked. |
| `PUT` | `/api/notifications/channels` | Updates APNs config and/or webhook list in `config.yaml`, preserving untouched `bot_token` and sensitive `headers`. |
| `POST` | `/api/notifications/test` | Dispatches a live test notification to a specific `channel_id`, `channel_type` (`apns \| discord \| telegram \| generic \| all`), or all enabled channels. |

---

## Live Operations & Unified User Management

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/ops/overview` | Concurrently fetches Proxmox `pve_tasks`, pending Jellyseerr `seerr_requests`, and `qbit_control` (`alt_speed_enabled` + latest torrents). |
| `POST` | `/api/ops/seerr/{request_id}/{decision}` | Executes `approve` or `decline` on a Jellyseerr media request. |
| `POST` | `/api/ops/qbit/toggle-speed-mode` | Toggles qBittorrent alternative speed limits (Turtle Mode). |
| `POST` | `/api/ops/qbit/torrent/{torrent_hash}/{action}` | Executes `pause` or `resume` on a torrent (supports qBittorrent v4.x and v5.x endpoints). |
| `POST` | `/api/ops/media/scan/{target}` | Triggers a library refresh on `jellyfin` or `plex`. |
| `GET` | `/api/users` | Fetches Jellyfin users, virtual library folders, and synced Jellyseerr users/permissions concurrently. |
| `POST` | `/api/users` | Creates a Jellyfin user, applies folder/download/max-session policies, and optionally syncs to Jellyseerr. |
| `POST` | `/api/users/sync` | Imports missing (or specified) Jellyfin users into Jellyseerr. |
| `PATCH` | `/api/users/{jellyfin_id}` | Updates Jellyfin policy (`is_disabled`, `is_admin`, `enable_all_folders`, `enabled_folders`, `enable_downloading`, `max_active_sessions`) and Jellyseerr `auto_approve`. Protects `protected_owner_username`. |
| `POST` | `/api/users/{jellyfin_id}/password` | Resets and sets a new password on a Jellyfin user. |
| `DELETE` | `/api/users/{jellyfin_id}` | Deletes a user from Jellyseerr (if linked) and Jellyfin. Blocks deletion of `protected_owner_username`. |

---

## Prometheus Observability

| Method | Path | Role | Description |
|---|---|---|---|
| `GET` | `/metrics` | Any auth (`viewer`+) | Exposes Prometheus text metrics (`0.0.4`): `pve_center_up`, `pve_custom_service_up`, `pve_custom_service_latency_ms`, `pve_incidents_active_total`, `pve_node_*`, and `pve_guest_up`. |
| `GET` | `/api/metrics/prometheus` | Any auth (`viewer`+) | Alias under `/api` returning the same Prometheus scrape payload. |
