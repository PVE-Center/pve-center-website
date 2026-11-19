# Multi-Platform Clients & Build Guides

PVE Command Center provides dedicated client applications across Apple ecosystems, Windows/Linux desktops, Android devices, and web browsers. All native clients share the same **Liquid Glass** & **Cider Studio** aesthetics, support **Demo Mode**, **multi-node Proxmox VE cluster switching**, and connect over **LAN HTTP**, **Bonjour mDNS**, or **Tailscale HTTPS**.

---

## Multi-Node Cluster Telemetry & Failover {#multi-node-cluster}

All client apps (Apple SwiftUI, Desktop Tauri v2, Android Capacitor 8, and Web PWA) support multi-node Proxmox VE clusters via the server's `?node=` query contract:

- **Cluster & Per-Node Views**: Switch seamlessly between individual cluster nodes (`GET /api/pve/overview?node=<name>`) and aggregated **Cluster (`*`)** mode (`GET /api/pve/overview?node=*` / `all`) using the Proxmox section pill selector showing live online status and CPU utilization per node.
- **Automatic Failover Banner**: When the primary configured node is unreachable and the backend automatically fails over (`failover_active: true`), clients surface a warning banner identifying the active fallback node.
- **Host Node Badges & Targeted Power Actions**: Each LXC container and QEMU VM row displays its hosting `node` badge, and power commands (`start`, `shutdown`, `reboot`, `stop`) automatically pass `?node=<guest.node>` to `POST /api/pve/{vm_type}/{vmid}/{action}` so actions always reach the right hypervisor host.
- **Two-Node Demo Mode**: Built-in Demo Mode simulates two cluster nodes (`pve-node-01` and `pve-node-02`) with node filtering and cluster-wide aggregation.

---

## Apple Native (`iOS / iPadOS / macOS / tvOS`) {#apple-native}

**Repository:** [`PVE-Center/pve-center-apple`](https://github.com/PVE-Center/pve-center-apple)

A single **Swift 6** / **SwiftUI** codebase (`PVECenter.xcodeproj` & `Package.swift`) targeting:

- **iPhone & iPad (`iOS 27.0`)**: Liquid Glass TabView, `.sensoryFeedback`, `AVCaptureSession` QR scanner (`pvecenter://`), SwiftData cache, Keychain credential storage, and interactive local/APNs notifications (`Mute 1h`, `Test`).
- **macOS (`macOS 27.0`)**: `NavigationSplitView`, menu bar popover (`MenuBarExtra`) displaying live CPU/RAM/qBittorrent speeds/Jellyfin streams & Turtle Mode toggle, keyboard shortcuts (`⌘R`, `⌘L`, `⌘;`, `⌘1`–`⌘6`), and ad-hoc `/Applications/PVE Center.app` packaging.
- **Apple TV (`tvOS 27.0`)**: 10-foot dark glass Focus Engine UI (`TVGlassCardButtonStyle`), live WebSocket alert banners (`activeTVBanner`), and zero-typing encrypted PIN pairing. {#apple-tv}
- **WidgetKit Snapshot Telemetry Views**: `PVECenter/Views/Components/WidgetSnapshotViews.swift` provides `TelemetryWidgetSnapshot` (capturing `updatedAt`, `serverName`, `nodeName`, `online`, `cpuPct`, `memPct`, `runningGuests`, `totalGuests`, and active incident status), thread-safe `TelemetrySnapshotStore` (persisting ISO-8601 JSON atomically to Caches, `UserDefaults`, and optional App Group `group.io.ac.pvepanel.app` after `AppViewModel.refreshAll`), and the Cider Liquid Glass `GlanceableNodeWidgetView` across iOS, macOS, and tvOS.

### Apple Build & Run Commands

```bash
git clone https://github.com/PVE-Center/pve-center-apple.git
cd pve-center-apple

# Verify Swift 6 package build (macOS)
swift build

# Build & install native /Applications/PVE Center.app (Xcode)
./scripts/install-mac.sh

# Or package via SwiftPM + AppIcon.icns
./scripts/build-macos-app.sh

# Launch iOS or Apple TV Simulator
./scripts/run-iphone-sim.sh
./scripts/run-appletv-sim.sh

# Deploy to physical iPhone, iPad, or Apple TV (USB or Wi-Fi paired)
./scripts/deploy-device.sh --list
./scripts/deploy-device.sh -t <TEAM_ID> -p ios
./scripts/deploy-device.sh -t <TEAM_ID> -p tvos -d <APPLE_TV_UDID>
```

### App Store, APNs `.p8` & Sideloading

- **Free Personal Apple Account**: `aps-environment` is intentionally **not forced** in `project.pbxproj`, so any developer can build and run on personal iOS/tvOS devices immediately using Automatic Signing, AltStore, **SideStore** (Wi-Fi auto-refresh), or Sideloadly.
- **Paid Apple Developer Program ($99/year)**: Enable the **Push Notifications** capability in Xcode on target `PVECenter` (`io.ac.pvepanel.app`), create an APNs key (`AuthKey_<KEY_ID>.p8`), place it in `./config/AuthKey.p8` on your `pve-panel` server, and configure `notifications.apns` in `config/config.yaml`.

---

## Desktop (`Windows & Linux — Tauri v2`) {#desktop-tauri}

**Repository:** [`PVE-Center/pve-center-clients`](https://github.com/PVE-Center/pve-center-clients)

The desktop client wraps the React 19 + Tailwind CSS UI inside a **Tauri v2** Rust shell (`src-tauri/`, Bundle ID `io.github.pvepanel.desktop`, `1280×820` default window) with a native **System Tray** icon (Show/Hide window & Quit) and encrypted at-rest server profiles/tokens so you can point the desktop app to any LAN IP or `https://*.ts.net` URL.

### Prerequisites

- **Node.js 22+** & `npm`
- **Rust & Cargo** (`rustup`)
- **Linux**: WebKitGTK & libappindicator development headers (`libwebkit2gtk-4.1-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`, `patchelf` on Debian/Ubuntu)
- **Windows**: Visual Studio C++ Build Tools & WebView2 (bundled via `embedBootstrapper`)

### Desktop Build Commands

```bash
git clone https://github.com/PVE-Center/pve-center-clients.git
cd pve-center-clients
npm ci

# Run live Tauri v2 dev window
npm run tauri:dev

# Linux: build .AppImage and .deb packages
./scripts/build-desktop.sh
# Outputs under src-tauri/target/release/bundle/
```

On **Windows (PowerShell)**:

```powershell
git clone https://github.com/PVE-Center/pve-center-clients.git
cd pve-center-clients
.\scripts\build-windows.ps1
# Generates .exe (NSIS) and .msi installers under src-tauri\target\release\bundle\
```

---

## Android (`Capacitor 8`) {#android-capacitor}

**Repository:** [`PVE-Center/pve-center-clients`](https://github.com/PVE-Center/pve-center-clients)

The Android client packages the React 19 PWA with **Capacitor 8** (`@capacitor/android ^8.6.2`, App ID `io.github.pvepanel.app`, `CapacitorHttp` enabled) and pre-configures `AndroidManifest.xml` + `capacitor.config.ts` (`androidScheme: 'http'`, `cleartext: true`, `allowMixedContent: true`) so both local `http://192.168.x.x:8000` homelab IPs and `https://*.ts.net` Tailscale URLs work reliably.

### Prerequisites

- **Node.js 22+** & `npm`
- **Java JDK 17+** & **Android SDK** (or Android Studio)

### Android Build Commands

```bash
git clone https://github.com/PVE-Center/pve-center-clients.git
cd pve-center-clients
npm ci

# Sync web assets and build debug APK
./scripts/build-android.sh --debug

# Build release APK or Android App Bundle (.aab)
./scripts/build-android.sh --release
./scripts/build-android.sh --aab

# Or use npm scripts directly
npm run cap:sync
npm run android:build
```

Compiled `.apk` and `.aab` outputs are generated under `android/app/build/outputs/`.

---

## Web PWA (Safari, Chrome, Edge, Firefox)

No separate build is required for browser access: `pve-panel` serves the compiled React 19 PWA directly at `http://<LAN_IP>:8000` or `https://<hostname>.<tailnet>.ts.net`.

- **iPhone / iPad Safari**: Open your Tailscale HTTPS URL → **Share** → **Add to Home Screen** for a full-screen standalone Liquid Glass PWA.
- **Desktop Chromium / Edge**: Click **Install App** in the address bar.

---

## Encrypted Credential & Token Storage (`secureStorage.ts`) {#encrypted-storage}

The shared client runtime (`pve-center-clients/src/lib/secureStorage.ts`, integrated into `src/lib/apiClient.ts` and hydrated on boot via `ensureSecureStorageReady()`) protects persisted server URLs, Bearer API tokens (`pve_...`), and session JWTs at rest:

- **WebCrypto AES-256-GCM**: In secure contexts (`https://`, `localhost`, Tauri, Capacitor), values are encrypted using `window.crypto.subtle` (`AES-GCM`, 256-bit key with random 12-byte IV per write). A non-extractable `CryptoKey` is stored in IndexedDB (`pve_center_secure_keystore`), with an automatic PBKDF2-derived fallback key when IndexedDB is restricted.
- **Automatic Plaintext Migration**: Existing unencrypted `localStorage` entries (`pve_panel_server_url`, `pve_panel_auth_token`) are decrypted/migrated transparently on startup without logging the user out.
- **Plain HTTP LAN Fallback**: When `window.crypto.subtle` is unavailable (for example when opening the Web PWA directly over plain `http://192.168.x.x:8000` on a LAN), `secureStorage.ts` falls back synchronously and non-blockingly to an obfuscated payload format so local HTTP homelab access continues working reliably.

---

## Automated Tag-Based Release Publishing (`v*` Tags) {#release-publishing}

In [`PVE-Center/pve-center-clients`](https://github.com/PVE-Center/pve-center-clients), `.github/workflows/build-clients.yml` triggers on pushes/PRs to `main` and on version tags matching `v*` (for example `v1.0.0`):

1. **Matrix Build Jobs**: Builds the Web SPA (`web`), Linux `.AppImage` / `.deb` and Windows `.exe` / `.msi` Tauri v2 installers (`desktop`), and Android Capacitor 8 APK (`android`).
2. **Release Job (`if: startsWith(github.ref, 'refs/tags/v')`)**: Waits for `[web, desktop, android]`, merges all build artifacts via `actions/download-artifact@v4` (`merge-multiple: true`), packages the Web SPA into `pve-center-web-dist.tar.gz`, and publishes a GitHub Release with auto-generated notes via `softprops/action-gh-release@v2` (`generate_release_notes: true`, `permissions: { contents: write }`).
