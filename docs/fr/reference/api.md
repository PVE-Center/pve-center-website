# Référence API (REST, WebSocket & SSE)

PVE Command Center expose une API REST typée sous FastAPI ainsi que deux flux temps réel **WebSocket** et **Server-Sent Events (SSE)** sur le port `8000` (ou en HTTPS via Tailscale Serve).

## Authentification & En-têtes

| Mode | En-tête / Paramètre URL | Usage principal |
|---|---|---|
| **Tailscale Serve SSO** | `Tailscale-User-Login` (vérifié en loopback `127.0.0.1` / `::1` si `tailscale_sso_enabled: true`) | Navigateur / HTTPS Tailscale (`*.ts.net`) |
| **Jeton API Bearer** | `Authorization: Bearer pve_<token>` | iPhone, iPad, Mac, Apple TV, Windows, Linux, Android, scripts |
| **JWT Session** | `Authorization: Bearer <jwt>` (JWT HS256 24h délivré par `POST /api/auth/login`) | Navigateur / connexion par code PIN |
| **WebSocket / SSE** | `?token=<api_token_or_jwt>` ou `Authorization: Bearer <token>` | `/api/ws/events` et `/api/events/stream` |
| **Webhook entrant** | `Authorization: Bearer`, `auth_header_name` personnalisé, `X-Webhook-Token`, `X-API-Key` ou `?token=` | `POST /api/custom-services/{service_id}/webhook` |

### Contrôle d’accès par rôles (RBAC)

Chaque jeton API ou session porte un rôle (`admin`, `operator`, `viewer`, `device`) :

- **`admin`** : Accès complet en lecture et écriture (`/api/config/*`, `/api/auth/tokens`, `/api/setup/config`, `/api/discovery/qr`, utilisateurs, canaux d’alerte).
- **`operator` / `device`** : Lectures + écritures opérationnelles (`/api/pve/*`, `/api/ops/*`, `/api/incidents/*`, tests de sondes et `/api/notifications/devices`). Les écritures de configuration et lectures admin renvoient `403`.
- **`viewer`** : Télémétrie en lecture seule (`GET`/`HEAD`, dont `/metrics` et `/api/metrics/prometheus`) et enregistrement push ; les écritures renvoient `403`.

### Routes publiques cadrées

- `GET /api/health`
- `POST /api/auth/login`
- `/api/pairing/*` (protégé par PIN + anti brute-force)
- `GET /api/onboarding/status`
- `POST /api/custom-services/{service_id}/webhook` (vérifié selon `auth_type` & `auth_token` du service)
- Routes d’onboarding et setup (`/api/onboarding*`, `/api/setup/status`, `/api/setup/test-service`, `/api/setup/config`, `/api/discovery/lan`) **uniquement tant que `is_configured == false`**.

---

## Santé & Télémétrie temps réel

| Méthode | Chemin | Auth | Description |
|---|---|---|---|
| `GET` | `/api/health` | Public | Healthcheck léger retournant `status`, `version`, `server_id`, `server_name`, `is_configured` et `config_hash` (12 car.). |
| `GET` | `/api/events/stream` | DualAuth / `?token=` | Flux SSE (`text/event-stream`). Émet `: connected`, `: keepalive` toutes les `25`s et `data: <json>\n\n`. |
| `WS` | `/api/ws/events` | DualAuth / `?token=` | Flux WebSocket (fermeture `4401` si non authentifié). Envoie `ping` toutes les `25`s, accepte `"pong"` ou `{"type":"pong"}`. |

### Enveloppe & Événements temps réel

Chaque message WebSocket ou SSE suit l’enveloppe canonique en `snake_case` :

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

Valeurs de `type` émises : `ping`, `config_reloaded`, `service_status_changed`, `alert_triggered`, `alert_resolved`, `incident_acknowledged`, `pairing_completed`.

---

## Authentification, PIN & Jetons API

| Méthode | Chemin | Rôle | Description |
|---|---|---|---|
| `GET` | `/api/me` | Authentifié | Retourne l’identité courante (`login`, `name`, `auth_mode`, `role`). |
| `POST` | `/api/auth/login` | Public | Accepte `{ "pin": "..." }` ou `{ "api_token": "..." }` (alias `token`). Retourne `access_token` (JWT HS256 24h). Limité à 10 échecs / 60s (`429`). |
| `GET` | `/api/auth/tokens` | Admin | Liste les métadonnées des jetons (`id`, `name`, `role`, `created_at`) sans exposer les hash. |
| `POST` | `/api/auth/tokens` | Admin | Génère un jeton nommé (`role`: `admin \| operator \| viewer \| device`), enregistre son hash SHA-256 et renvoie `pve_...` une seule fois. |
| `DELETE` | `/api/auth/tokens/{token_id}` | Admin | Révoque et supprime un jeton Bearer par ID. |
| `PUT` | `/api/auth/pin` | Admin | Définit ou modifie le code PIN administrateur (`4..32` car.) dans `security.admin_pin_hash`. |

---

## Configuration déclarative & Onboarding

| Méthode | Chemin | Rôle | Description |
|---|---|---|---|
| `GET` | `/api/onboarding/status` | Public | État sans secret : `is_configured`, `needs_onboarding`, `server_id`, `server_name`, `public_url`, `tailscale_sso_enabled`, `pin_configured`, `api_tokens_count`, `auth_methods`. |
| `POST` | `/api/onboarding/init` | Public si non configuré, sinon Admin | Initialise `config/config.yaml`, PIN admin optionnel, Proxmox/intégrations/notifications et génère le premier `api_token`. |
| `GET` | `/api/config/yaml` | Admin | Retourne le texte `yaml` masqué (en préservant les `${VAR}`), `hash`, `is_synthesized`, `config_exists`, `parsed_summary`. |
| `POST` | `/api/config/validate` | Admin | Valide un YAML candidat (`{ "yaml": "..." }`) sans écrire sur disque (`valid`, `errors`, `warnings`, `parsed_summary`). |
| `PUT` | `/api/config/yaml` | Admin | Valide, écrit de manière atomique `config/config.yaml` et recharge à chaud `AlertEngine`, `MDNSPublisher` et `pve_http` (`422` si invalide). |
| `GET` | `/api/config/export` | Admin | Télécharge une copie masquée de `config.yaml` (`attachment; filename="config.yaml"`). |

---

## Setup & Tests de connectivité

| Méthode | Chemin | Auth | Description |
|---|---|---|---|
| `GET` | `/api/setup/status` | Exempt 1er démarrage / DualAuth | Retourne `is_configured`, `tailscale_url`, `pairing_deep_link` (`pvecenter://connect?...`), `env` nettoyé, `categories` et `storage_mounts`. |
| `POST` | `/api/setup/test-service` | Exempt 1er démarrage / DualAuth | Teste en direct `proxmox`, `jellyfin`, `jellyseerr`, `plex`, `radarr`, `sonarr`, `prowlarr` ou `qbittorrent`. |
| `PUT` | `/api/setup/config` | Exempt 1er démarrage / DualAuth | Met à jour `env`, `categories` et `storage_mounts` dans `config.yaml` (ou `.env`/`services.yaml`) avec rechargement à chaud. |

---

## Découverte Bonjour & Appairage Apple TV

| Méthode | Chemin | Auth | Description |
|---|---|---|---|
| `GET` | `/api/discovery/lan` | Exempt 1er démarrage / DualAuth | Scan mDNS local (`timeout` `0.5..4.0`s, défaut `2.0`s) sur `_pvecenter._tcp`, `_http._tcp`, `_ssh._tcp`, `_smb._tcp`. |
| `GET` | `/api/discovery/qr` | DualAuth | Retourne `deep_link` (`pvecenter://connect?url=...&name=...&id=...`), `onboarding_link`, `public_url`, `lan_ip`, `port`, `mdns_service_type`. |
| `POST` | `/api/pairing/sessions` | Public | L’Apple TV enregistre sa clé publique P-256 éphémère + `device_name` ; retourne le `pin` à 6 chiffres, `session_id` et `expires_at` (TTL `300`s). |
| `GET` | `/api/pairing/sessions/{pin}` | Public (rate-limit) | L’iPhone récupère la `public_key` et le `device_name` du récepteur. |
| `POST` | `/api/pairing/sessions/{pin}/complete` | Public (rate-limit) | L’iPhone dépose le paquet chiffré AES-256-GCM (`sender_public_key`, `nonce`, `ciphertext`, `tag`) et publie `pairing_completed` sur `RealtimeBus`. |
| `GET` | `/api/pairing/sessions/{pin}/payload` | Public (rate-limit) | L’Apple TV consomme le paquet chiffré en lecture unique (`425` si en attente) et supprime aussitôt la session. |

---

## Cluster Proxmox VE & Stockage

| Méthode | Chemin | Description |
|---|---|---|
| `GET` | `/api/pve/nodes` | Liste les nœuds du cluster Proxmox VE avec `status`, `online`, `cpu_pct`, `mem_pct` et `uptime`. |
| `GET` | `/api/pve/overview` | Optionnel `?node=<nom> \| * \| all` : agrège métriques d’un nœud ou du cluster multi-nœuds, bascule automatique (`failover_active`), stockages, LXC, VMs QEMU, disques SMART et points `/host_mnt`. |
| `POST` | `/api/pve/{vm_type}/{vmid}/{action}` | Optionnel `?node=<nom>` ; exécute `start`, `stop`, `reboot` ou `shutdown` sur `vm_type` (`lxc` \| `qemu`). Refuse l’arrêt si `vmid == panel_self_vmid`. |

---

## Services & Widgets Médias

| Méthode | Chemin | Description |
|---|---|---|
| `GET` | `/api/services` | Exécute les healthchecks concurrents sur `categories` et `custom_services`. |
| `PUT` | `/api/services/items/{service_id}` | Met à jour `name`, `description`, `url`, `ping_url`, `icon`, `widget` ou `category`. |
| `POST` | `/api/services/items/{service_id}/move` | Déplace une carte `{ "direction": "up" \| "down" }` dans sa catégorie. |
| `POST` | `/api/services/items/{service_id}/test` | Teste immédiatement une URL enregistrée ou en cours d’édition. |
| `DELETE` | `/api/services/items/{service_id}` | Supprime une carte de service de sa catégorie. |
| `GET` | `/api/widgets/all` | Récupère en parallèle tous les widgets configurés (`jellyfin`, `plex`, `radarr`, `sonarr`, `prowlarr`, `qbittorrent`). |
| `GET` | `/api/widgets/{widget_name}` | Récupère les métriques en direct d’un widget spécifique. |

---

## Services personnalisés, Sondes & Webhooks

| Méthode | Chemin | Description |
|---|---|---|
| `GET` | `/api/custom-services` | Liste les services personnalisés enrichis avec santé temps réel, latence, métriques et état `StateStore`. |
| `POST` | `/api/custom-services` | Crée un service personnalisé (`201 Created`, génère un `id` slug si omis) et l’enregistre dans `config.yaml`. |
| `POST` | `/api/custom-services/reorder` | Réordonne les services personnalisés selon `service_ids` (alias `ids` / `order`). |
| `PUT` | `/api/custom-services/{service_id}` | Met à jour un service personnalisé tout en préservant `auth_token` s’il est vide ou masqué (`********`). |
| `DELETE` | `/api/custom-services/{service_id}` | Supprime un service personnalisé de `config.yaml`. |
| `POST` | `/api/custom-services/{service_id}/test` | Exécute une sonde à blanc sur la configuration enregistrée ou un payload non sauvegardé. |
| `POST` | `/api/custom-services/{service_id}/webhook` | Point d’ingestion push (`status`: `up \| degraded \| down`, `message`, `ttl_seconds`, `metrics`), met à jour `StateStore` et déclenche `AlertEngine.evaluate_service()`. |

---

## Incidents & Notifications

| Méthode | Chemin | Description |
|---|---|---|
| `GET` | `/api/incidents` | Retourne la timeline d’incidents SQLite (`?active_only=true&limit=50&service_id=...`), `active_count` et `service_states`. |
| `POST` | `/api/incidents/{incident_id}/ack` | Acquitte un incident et publie `incident_acknowledged`. |
| `POST` | `/api/incidents/services/{service_id}/mute` | Coupe les notifications du service pendant `minutes` (`1..43200`, défaut `60`). |
| `DELETE` | `/api/incidents/services/{service_id}/mute` | Lève le silence (`Mute`) sur `service_id`. |
| `POST` | `/api/notifications/devices` | Enregistre un jeton APNs (`device_token`, `platform`: `ios \| ipados \| tvos \| macos`, `device_name`, `app_version`, `environment`). |
| `DELETE` | `/api/notifications/devices/{token}` | Supprime un appareil APNs enregistré. |
| `GET` | `/api/notifications/channels` | Retourne l’état APNs (`configured`, `private_key_exists`, `devices`) et les webhooks avec secrets masqués. |
| `PUT` | `/api/notifications/channels` | Enregistre la config APNs et/ou les webhooks dans `config.yaml` en préservant `bot_token` et `headers` sensibles inchangés. |
| `POST` | `/api/notifications/test` | Envoie une notification de test vers `channel_id`, `channel_type` (`apns \| discord \| telegram \| generic \| all`) ou tous les canaux actifs. |

---

## Opérations & Gestion unifiée des utilisateurs

| Méthode | Chemin | Description |
|---|---|---|
| `GET` | `/api/ops/overview` | Agrège tâches Proxmox (`pve_tasks`), demandes Jellyseerr (`seerr_requests`) et contrôle qBittorrent (`qbit_control`). |
| `POST` | `/api/ops/seerr/{request_id}/{decision}` | Approuve (`approve`) ou refuse (`decline`) une demande Jellyseerr. |
| `POST` | `/api/ops/qbit/toggle-speed-mode` | Bascule les limites de vitesse alternatives (mode Tortue) sur qBittorrent. |
| `POST` | `/api/ops/qbit/torrent/{torrent_hash}/{action}` | Exécute `pause` ou `resume` sur un torrent (compatible qBittorrent v4.x et v5.x). |
| `POST` | `/api/ops/media/scan/{target}` | Lance un rafraîchissement des bibliothèques `jellyfin` ou `plex`. |
| `GET` | `/api/users` | Liste les comptes Jellyfin, les bibliothèques virtuelles et l’état de synchronisation/permissions Jellyseerr. |
| `POST` | `/api/users` | Crée un utilisateur Jellyfin, applique ses droits dossiers/téléchargements/sessions et le réplique optionnellement dans Jellyseerr. |
| `POST` | `/api/users/sync` | Importe les comptes Jellyfin manquants (ou ciblés) vers Jellyseerr. |
| `PATCH` | `/api/users/{jellyfin_id}` | Met à jour les droits Jellyfin (`is_disabled`, `is_admin`, `enable_all_folders`, `enabled_folders`, `enable_downloading`, `max_active_sessions`) et `auto_approve` Jellyseerr. Protège `protected_owner_username`. |
| `POST` | `/api/users/{jellyfin_id}/password` | Réinitialise et définit un nouveau mot de passe utilisateur Jellyfin. |
| `DELETE` | `/api/users/{jellyfin_id}` | Supprime un utilisateur de Jellyseerr (si présent) puis de Jellyfin. Interdit sur `protected_owner_username`. |

---

## Observabilité Prometheus

| Méthode | Chemin | Rôle | Description |
|---|---|---|---|
| `GET` | `/metrics` | Authentifié (`viewer`+) | Expose les métriques Prometheus (`0.0.4`) : `pve_center_up`, `pve_custom_service_up`, `pve_custom_service_latency_ms`, `pve_incidents_active_total`, `pve_node_*` et `pve_guest_up`. |
| `GET` | `/api/metrics/prometheus` | Authentifié (`viewer`+) | Alias sous `/api` retournant la même sortie texte Prometheus. |
