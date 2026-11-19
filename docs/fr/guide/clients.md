# Clients Multi-Plateformes & Guides de Compilation

PVE Command Center propose des applications clientes dédiées pour l’écosystème Apple, les ordinateurs Windows et Linux, les appareils Android et tout navigateur Web. Tous les clients partagent l’esthétique **Liquid Glass** & **Cider Studio**, intègrent un **Mode Démo**, supportent le **pilotage multi-nœuds Proxmox VE** et se connectent indifféremment en **HTTP LAN**, via **Bonjour mDNS** ou en **HTTPS Tailscale**.

---

## Superviser un Cluster Multi-Nœuds & Failover {#multi-node-cluster}

Toutes les applications clientes (Apple SwiftUI, Desktop Tauri v2, Android Capacitor 8 et PWA Web) exploitent le contrat multi-nœuds `?node=` du serveur :

- **Vue Cluster (`*`) et par nœud** : basculez entre un nœud précis (`GET /api/pve/overview?node=<nom>`) et la vue agrégée **Cluster (`*`)** (`GET /api/pve/overview?node=*` / `all`) via les pilules de sélection affichant l’état en ligne et le CPU de chaque hôte.
- **Bannière de bascule automatique (Failover)** : lorsque le nœud principal configuré ne répond plus et que le serveur bascule automatiquement (`failover_active: true`), une bannière d’avertissement indique le nœud de secours actif.
- **Badges d’hôte & actions ciblées** : chaque ligne LXC/VM affiche le badge de son nœud hébergeur (`node`), et les commandes (`start`, `shutdown`, `reboot`, `stop`) transmettent `?node=<guest.node>` sur `POST /api/pve/{vm_type}/{vmid}/{action}` afin d’exécuter l’ordre sur le bon hyperviseur.
- **Mode Démo bi-nœuds** : le Mode Démo embarqué simule deux nœuds (`pve-node-01` et `pve-node-02`) avec filtrage et agrégation cluster.

---

## Apple Natif (`iOS / iPadOS / macOS / tvOS`) {#apple-native}

**Dépôt :** [`PVE-Center/pve-center-apple`](https://github.com/PVE-Center/pve-center-apple)

Une base de code unique **Swift 6** / **SwiftUI** (`PVECenter.xcodeproj` & `Package.swift`) ciblant :

- **iPhone & iPad (`iOS 27.0`)** : `TabView` Liquid Glass, retour haptique `.sensoryFeedback`, scanner QR `AVCaptureSession` (`pvecenter://`), cache local SwiftData, trousseau Keychain et notifications locales/APNs interactives (`Mute 1h`, `Tester`).
- **macOS (`macOS 27.0`)** : `NavigationSplitView`, menu barre (`MenuBarExtra`) affichant CPU/RAM/débits qBittorrent/flux Jellyfin & mode Tortue, raccourcis clavier (`⌘R`, `⌘L`, `⌘;`, `⌘1`–`⌘6`) et installation native dans `/Applications/PVE Center.app`.
- **Apple TV (`tvOS 27.0`)** : interface 10-foot sombre haute lisibilité (`TVGlassCardButtonStyle`), bannières d’alertes temps réel WebSocket (`activeTVBanner`) et appairage chiffré sans saisie à la télécommande. {#apple-tv}
- **Vues WidgetKit & Snapshots de Télémétrie** : `PVECenter/Views/Components/WidgetSnapshotViews.swift` fournit le modèle `TelemetryWidgetSnapshot` (`updatedAt`, `serverName`, `nodeName`, `online`, `cpuPct`, `memPct`, `runningGuests`, `totalGuests`, état des incidents), le magasin thread-safe `TelemetrySnapshotStore` (écriture JSON ISO-8601 atomique vers Caches, `UserDefaults` et App Group `group.io.ac.pvepanel.app` après `AppViewModel.refreshAll`) ainsi que la vue Liquid Glass `GlanceableNodeWidgetView` compatible iOS, macOS et tvOS.

### Commandes de compilation et déploiement Apple

```bash
git clone https://github.com/PVE-Center/pve-center-apple.git
cd pve-center-apple

# Vérifier le package Swift 6 (macOS)
swift build

# Compiler et installer /Applications/PVE Center.app (via Xcode)
./scripts/install-mac.sh

# Ou empaqueter via SwiftPM + AppIcon.icns
./scripts/build-macos-app.sh

# Lancer les simulateurs iPhone ou Apple TV
./scripts/run-iphone-sim.sh
./scripts/run-appletv-sim.sh

# Déployer sur iPhone, iPad ou Apple TV physique (USB ou Wi-Fi)
./scripts/deploy-device.sh --list
./scripts/deploy-device.sh -t <TEAM_ID> -p ios
./scripts/deploy-device.sh -t <TEAM_ID> -p tvos -d <UDID_APPLE_TV>
```

### App Store, APNs `.p8` & Sideloading

- **Compte Apple gratuit (Personal Team)** : `aps-environment` n’est **pas forcé** dans `project.pbxproj`, permettant à quiconque de compiler et installer l’application sur ses appareils personnels via Signature Automatique Xcode, AltStore, **SideStore** (renouvellement auto en Wi-Fi) ou Sideloadly.
- **Apple Developer Program (99 $/an)** : ajoutez la capability **Push Notifications** dans Xcode sur la cible `PVECenter` (`io.ac.pvepanel.app`), générez une clé APNs (`AuthKey_<KEY_ID>.p8`), copiez-la dans `./config/AuthKey.p8` sur le serveur `pve-panel` et activez `notifications.apns` dans `config/config.yaml`.

---

## Desktop (`Windows & Linux — Tauri v2`) {#desktop-tauri}

**Dépôt :** [`PVE-Center/pve-center-clients`](https://github.com/PVE-Center/pve-center-clients)

Le client bureau encapsule l’interface React 19 + Tailwind CSS dans une coquille Rust **Tauri v2** (`src-tauri/`, Bundle ID `io.github.pvepanel.desktop`, fenêtre `1280×820` par défaut) dotée d’une icône **System Tray** native (Afficher/Masquer & Quitter) et d’un stockage chiffré au repos pour les profils serveurs/jetons compatible IP LAN et `https://*.ts.net`.

### Prérequis

- **Node.js 22+** & `npm`
- **Rust & Cargo** (`rustup`)
- **Linux** : paquets de développement WebKitGTK & libappindicator (`libwebkit2gtk-4.1-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`, `patchelf` sous Debian/Ubuntu)
- **Windows** : outils C++ Visual Studio & WebView2 (bootstrapper embarqué)

### Commandes de compilation Desktop

```bash
git clone https://github.com/PVE-Center/pve-center-clients.git
cd pve-center-clients
npm ci

# Lancer en mode développement Tauri v2
npm run tauri:dev

# Linux : compiler les paquets .AppImage et .deb
./scripts/build-desktop.sh
# Artefacts générés dans src-tauri/target/release/bundle/
```

Sous **Windows (PowerShell)** :

```powershell
git clone https://github.com/PVE-Center/pve-center-clients.git
cd pve-center-clients
.\scripts\build-windows.ps1
# Génère les installeurs .exe (NSIS) et .msi dans src-tauri\target\release\bundle\
```

---

## Android (`Capacitor 8`) {#android-capacitor}

**Dépôt :** [`PVE-Center/pve-center-clients`](https://github.com/PVE-Center/pve-center-clients)

Le client Android empaquette la PWA React 19 avec **Capacitor 8** (`@capacitor/android ^8.6.2`, App ID `io.github.pvepanel.app`, `CapacitorHttp` activé). `AndroidManifest.xml` et `capacitor.config.ts` (`androidScheme: 'http'`, `cleartext: true`, `allowMixedContent: true`) sont préconfigurés pour supporter aussi bien les IP LAN locales (`http://192.168.x.x:8000`) que les domaines HTTPS Tailscale (`https://*.ts.net`).

### Prérequis

- **Node.js 22+** & `npm`
- **Java JDK 17+** & **Android SDK** (ou Android Studio)

### Commandes de compilation Android

```bash
git clone https://github.com/PVE-Center/pve-center-clients.git
cd pve-center-clients
npm ci

# Synchroniser les assets Web et compiler l'APK debug
./scripts/build-android.sh --debug

# Compiler l'APK release ou l'App Bundle (.aab)
./scripts/build-android.sh --release
./scripts/build-android.sh --aab

# Ou via les scripts npm directs
npm run cap:sync
npm run android:build
```

Les fichiers `.apk` et `.aab` générés se trouvent sous `android/app/build/outputs/`.

---

## Web PWA (Safari, Chrome, Edge, Firefox)

Aucune compilation séparée n’est requise : `pve-panel` sert directement la PWA React 19 sur `http://<IP_LAN>:8000` ou `https://<hostname>.<tailnet>.ts.net`.

- **Safari iPhone / iPad** : ouvrez votre URL HTTPS Tailscale → **Partager** → **Sur l’écran d’accueil** pour installer la PWA Liquid Glass en mode plein écran.
- **Chrome / Edge Desktop** : cliquez sur **Installer l’application** dans la barre d’adresse.

---

## Stockage Chiffré des Jetons (`secureStorage.ts`) {#encrypted-storage}

Le cœur client partagé (`pve-center-clients/src/lib/secureStorage.ts`, intégré dans `src/lib/apiClient.ts` et préchargé au démarrage via `ensureSecureStorageReady()`) protège au repos l’URL du serveur, les jetons API Bearer (`pve_...`) et les JWT de session :

- **WebCrypto AES-256-GCM** : en contexte sécurisé (`https://`, `localhost`, Tauri, Capacitor), les données sont chiffrées via `window.crypto.subtle` (`AES-GCM` 256 bits avec IV aléatoire de 12 octets par écriture). Une clé `CryptoKey` non extractible est stockée dans IndexedDB (`pve_center_secure_keystore`), avec repli automatique sur une clé dérivée PBKDF2 si IndexedDB est indisponible.
- **Migration transparente depuis le clair** : les anciennes entrées `localStorage` en clair (`pve_panel_server_url`, `pve_panel_auth_token`) sont automatiquement chiffrées au démarrage sans déconnecter l’utilisateur.
- **Repli non bloquant en HTTP LAN** : lorsque `window.crypto.subtle` est indisponible (accès direct à la PWA Web sur `http://192.168.x.x:8000` en réseau local), `secureStorage.ts` bascule sans blocage sur un format obfusqué synchrone afin de préserver l’accès LAN HTTP.

---

## Publication Automatisée des Releases sur Tags (`v*`) {#release-publishing}

Dans [`PVE-Center/pve-center-clients`](https://github.com/PVE-Center/pve-center-clients), le workflow `.github/workflows/build-clients.yml` s’exécute sur `main` ainsi que sur les tags `v*` (par exemple `v1.0.0`) :

1. **Matrice de compilation** : génère la PWA Web (`web`), les paquets Linux `.AppImage` / `.deb` et installeurs Windows `.exe` / `.msi` Tauri v2 (`desktop`), ainsi que l’APK Android Capacitor 8 (`android`).
2. **Job `release` (`if: startsWith(github.ref, 'refs/tags/v')`)** : attend `[web, desktop, android]`, rassemble tous les artefacts via `actions/download-artifact@v4` (`merge-multiple: true`), empaquette la PWA dans `pve-center-web-dist.tar.gz` et publie la GitHub Release avec notes automatiques via `softprops/action-gh-release@v2` (`generate_release_notes: true`, `permissions: { contents: write }`).
