# FreeTube — Project Analysis & Architecture

> Analysis of commit `c038450ed429af3bc8f165aa82c58b2995ae4179` (branch `development`, version `0.25.3`).
> FreeTube is an open-source, privacy-focused desktop YouTube client (AGPL-3.0-or-later),
> built with Electron and Vue. It watches YouTube **without** using official Google APIs,
> cookies, or JavaScript tracking — data is scraped directly from YouTube's InnerTube
> endpoint in the browser ("Local API") or via the optional third-party Invidious API.

---

## 1. Technology Stack

### Runtime / Desktop shell
| Technology | Version | Role |
|---|---|---|
| **Electron** | `^43.4.0` | Desktop shell; main / preload / renderer processes |
| **Shaka Player** | `^5.1.12` | HTML5 video player (DASH/HLS manifests, custom scheme plugins, PiP, theatre mode, screenshots) |
| **youtubei.js (YouTube.js)** | `^18.0.0` | "Local API" — InnerTube scraping/session management, innertube JS deciphering |
| **NeDB (`@seald-io/nedb`)** | `^4.1.2` | Embedded document database for all local, user-owned data |

### UI
| Technology | Version | Role |
|---|---|---|
| **Vue 3** | `^3.5.42` | Composition API, `<script setup>` |
| **Vue Router** | `^5.3.0` | `createWebHashHistory()` routes (`#/watch/:id`, etc.) |
| **Vuex** | `^4.1.0` | Global store (state shared across the whole app) |
| **vue-i18n** | `^11.4.10` | Localization; YAML sources in `static/locales/` (Weblate-managed) |
| **Font Awesome** (svg-core, vue-fontawesome) | `^7.3.1` | Icons — imported *manually* in `renderer/main.js` (explicit allowlist) |
| **swiper** | `^14.2.0` | Carousels (used in channel pages) |
| **vue-observe-visibility** | `^2.0.0-alpha.1` | Lazy-loading / viewport visibility directives |
| **autolinker**, **dompurify**, **marked** | — | Rendering & sanitizing user-generated / video description HTML |
| **electron-context-menu** | `^4.1.2` | Custom right-click menu in the main process |
| **googlevideo** | `^4.1.1` | googlevideo URL / playback helpers |

### Build / tooling
| Technology | Version | Role |
|---|---|---|
| **Webpack 5** | `^5.110.2` | 5 separate build configs (main, renderer, preload, web, botGuardScript) |
| **Babel** | `^8.x` | JS transpilation (babel-loader) |
| **electron-builder** | `^26.15.7` | Packaging (NSIS/zip/7z/portable, dmg, deb/rpm/AppImage/pacman) |
| **Sass / SCSS + PostCSS** | `sass ^1.103` | Styling (`.scss`, `.css`) |
| **ESLint + stylelint** | `^10.x` / `^17.x` | Linting (configs: `eslint.config.mjs`, `stylelint.config.mjs`) |
| **lefthook** | `^2.1.12` | Git hooks |
| pnpm | — | Package manager (`pnpm-lock.yaml`, `pnpm-workspace.yaml`) |

### Key runtime build flags (webpack `DefinePlugin`)
These are baked in at compile time and heavily branch the code paths:

- `process.env.IS_ELECTRON` — true in renderer when running inside Electron (vs. dev server / web build)
- `process.env.IS_ELECTRON_MAIN` — true in the main process build
- `process.env.SUPPORTS_LOCAL_API` — whether the youtubei.js "Local API" backend is compiled in
- `process.platform` — `win32` / `darwin` / `linux`
- `process.env.LOCALE_NAMES`, `GEOLOCATION_NAMES`, `SWIPER_VERSION`,
  `SHAKA_LOCALE_MAPPINGS`, `HOT_RELOAD_LOCALES`, etc.
---

## 2. High-Level Architecture

FreeTube is a classic **3-process Electron app** with a webpack-bundled main process,
a small preload bridge, and a Vue 3 renderer. Data is persisted by the **main process**
in NeDB files; the renderer never touches disk directly — it talks to the main process
over a strictly-typed IPC surface.

```
┌────────────────────────────────────────────────────────────────────────┐
│                          Electron main process                          │
│                       src/main/index.js (2608 lines)                    │
│                                                                          │
│  • Window / tray / menu management        • Custom protocols            │
│  • IPC server (all DB + system services)    (app://bundle, imagecache://)│
│  • Network session fiddling               • Proxy (SOCKS/HTTP) support  │
│  • NeDB datastores (base handlers)        • Player cache, PO tokens     │
│  • electron-context-menu, external player, power-save blocker           │
└──────────────▲─────────────────────────────────────────────▲────────────┘
               │  IPC (ipcMain.handle / ipcMain.on)            │
┌──────────────┴─────────────────────────────────────────────┴────────────┐
│  Preload (src/preload/*)                                                 │
│   contextBridge.exposeInMainWorld('ftElectron', api)                     │
│   → renderer calls window.ftElectron.dbHistory(action, data), ...        │
│   → types declared in preload-interface.d.ts                             │
└──────────────▲─────────────────────────────────────────────▲────────────┘
│  Renderer (Vue 3 app)  — src/renderer/                                    │
│   App.vue (shell: TopNav, SideNav, prompts, RouterView)                   │
│   views/  (Subscriptions, Watch, Channel, SearchPage, Settings, ...)      │
│   components/ (~90 ft-* + feature components)                             │
│   store/    (Vuex modules → datastore handlers → IPC)                     │
│   router/index.js  (hash-based routes)                                    │
│   helpers/  (api/local.js, api/invidious.js, player/*, sponsorblock, ...) │
│   i18n/     (vue-i18n wiring)                                             │
└───────────────────────────────────────────────────────────────────────────┘
```

The renderer supports **two interchangeable data backends** (user-selectable setting
`backendPreference`): `'local'` (youtubei.js / InnerTube) and `'invidious'`.
A `backendFallback` setting allows falling back to the other backend on failure.
Both are consumed through structurally similar helpers (`helpers/api/local.js`,
`helpers/api/invidious.js`) that return **normalized, UI-ready objects**
(videos, channels, playlists, comments, formats, …).
---

## 3. Directory Structure

```
FreeTube/
├── _scripts/                    # All build/dev tooling (webpack configs, dev runner)
│   ├── webpack.main.config.js   #   → dist/main.js        (electron-main)
│   ├── webpack.renderer.config.js # → dist/renderer.js    (web; Vue, env flags)
│   ├── webpack.preload.config.js  # → dist/preload.js     (electron-preload)
│   ├── webpack.web.config.js      # → dist/web/*          (PWA web build)
│   ├── webpack.botGuardScript.config.js # → dist/botGuardScript.js
│   ├── dev-runner.js            # dev: webpack-dev-server :9080 + Electron
│   ├── injectAllowedPaths.mjs   # inlines app://bundle allowlist into main.js
│   ├── ProcessLocalesPlugin.js  # YAML locales → (brotli) JSON
│   ├── getShakaLocales.js / getInstances.js / getRegions.mjs / ...
│   └── ebuilder.config.mjs      # electron-builder packaging config
├── src/
│   ├── constants.js             # IpcChannels, DBActions, SyncEvents,
│   │                            #   KeyboardShortcuts, themes, limits…
│   ├── index.ejs                # HTML template (HtmlWebpackPlugin)
│   ├── botGuardScript.js        # YouTube bot-challenge solver script
│   ├── data/                    # (reserved; .gitkeep)
│   ├── datastores/
│   │   ├── index.js             # NeDB datastore definitions (6 DBs)
│   │   └── handlers/
│   │       ├── base.js          # real logic (runs in main process)
│   │       ├── electron.js      # renderer IPC proxy (webpack alias target)
│   │       ├── web.js           # web build (direct use of base)
│   │       └── index.js         # re-export via alias
│   │                            #   DB_HANDLERS_ELECTRON_RENDERER_OR_WEB
│   ├── main/                    # Electron main process
│   │   ├── index.js             # app entry: windows, IPC, network, protocols
│   │   ├── ImageCache.js        # imagecache:// protocol backing cache
│   │   ├── externalPlayer.js    # mpv/VLC/external player handling
│   │   ├── poTokenGenerator.js  # YouTube PO token generation
│   │   └── utils.js             # isFreeTubeUrl() etc.
│   ├── preload/
│   │   ├── main.js              # contextBridge exposure
│   │   ├── interface.js         # window.ftElectron API (IPC client)
│   │   └── preload-interface.d.ts # global types for renderer
│   └── renderer/                # Vue 3 app
│       ├── main.js              # entry: icon library, router/store/i18n
│       ├── App.vue / App.css    # app shell
│       ├── themes.css
│       ├── fontawesome-minimal.js
│       ├── sigFrameScript.js    # n/sig decipher iframe payload
│       ├── assets/  scss-partials/
│       ├── components/          # ~90 reusable components (ft-* prefix)
│       │   └── ft-shaka-video-player/
│       │       ├── ft-shaka-video-player.vue / .js / .css
│       │       └── player-components/  # custom Shaka UI extensions
│       ├── composables/  directives/(vSaferHtml)  helpers/
│       ├── i18n/                # vue-i18n setup + locale loader
│       ├── router/index.js      # all routes
│       ├── store/               # Vuex modules (see §6)
│       └── views/               # page-level views (Watch, Channel, …)
├── static/                      # shipped static assets
│   ├── locales/*.yaml           # Weblate translation sources
│   ├── geolocations/ invidious-instances.json external-player-map.json
│   ├── manifest.json  pwabuilder-sw.js      # PWA web app
│   └── dashFiles/ storyboards/  # (referenced, usually ignored in webpack copy)
├── dist/                        # webpack output (main.js, preload.js, …)
├── _icons/  .github/workflows/  # CI/CD (build, release, linter, codeql…)
├── package.json  pnpm-lock.yaml pnpm-workspace.yaml
├── eslint.config.mjs  stylelint.config.mjs  lefthook.yml
└── Docs.md                      # this file
```
---

## 4. Main Process (`src/main/index.js`)

All system-level work happens here. It is the **only** process with OS access
(renderer reaches it through the preload's IPC surface).

### 4.1 Startup flow
1. Parse `--version` / `--help` / `-h`.
2. Single-instance lock (`app.requestSingleInstanceLock()`) except in dev.
3. `baseHandlers.loadDatastores()` — load all 6 NeDB DBs, then `runApp()`.
4. Register privileged schemes: `imagecache://` (always), `app://` (production).
5. `app.whenReady()` → configure session security/permissions, restore settings
   (proxy, backend preference, tray behavior, base theme), create main window.

### 4.2 Custom URL protocols
- **`app://bundle/index.html`** — production entry point for the renderer.
  Served by a `protocol.handle('app', …)` handler that only serves files listed in
  `ALLOWED_RENDERER_FILES` (a `Set<string>` injected at build time by
  `_scripts/injectAllowedPaths.mjs`). Any other path → 400. Brotli JSON is
  transparently decompressed.
- **`imagecache://<encoded-url>#<webContentsId>`** — serves cached images
  (in-memory `ImageCache`) and proxies remote thumbnails through the main process
  so the renderer never directly loads images from arbitrary hosts.

### 4.3 Network session layer (anti-tracking / anti-bot)
- Sets a fixed User-Agent (Electron identifiers stripped).
- Consent cookies (`CONSENT=YES+`, `SOCS=CAI`) on youtube.com domains.
- `onBeforeSendHeaders` globally rewrites request headers per origin:
  - `youtubei/*` → `Referer/Origin = https://www.youtube.com/`, Sec-Fetch-* headers
  - `/watch` pages → no Referer/Origin, Sec-Fetch-Dest: document, PREF timezone cookie
  - `*.googlevideo.com/videoplayback` → media-like headers, no Content-Type
  - Invidious auth header injected when a per-window authorization exists
- `onHeadersReceived` strips `set-cookie` / CSP / report headers on YouTube pages.
- Optional proxy support (`ENABLE_PROXY`/`DISABLE_PROXY` IPC, SOCKS5 default) and
  proxy auth on the `app.on('login')` event.
### 4.4 Security posture (very relevant for the future local-video feature)
- `setPermissionCheckHandler` / `setPermissionRequestHandler`: only
  `fullscreen`, `clipboard-sanitized-write`, and non-directory `fileSystem` requests
  are allowed, **and only from FreeTube URLs**.
- `file-system-access-restricted` event: non-directory file access allowed only from
  FreeTube origins (this is how Import/Export data uses the Web File System API).
- Every IPC handler starts with `if (!isFreeTubeUrl(event.senderFrame.url)) return`.
- The `app://bundle` allowlist means arbitrary local files cannot be served to the
  renderer today.
  → **Serving local video files will require an explicit, new mechanism** (a custom
  protocol like `imagecache://`, a dedicated IPC that returns blob bytes, or a
  deliberate, narrowly-scoped expansion of file-system permissions).

### 4.5 IPC surface (all channels enumerated in `src/constants.js`)
Examples: `GET_SYSTEM_LOCALE`, `OPEN_URL`, `CREATE_NEW_WINDOW`, `CHANGE_VIEW`,
`SET_WINDOW_TITLE`, `NATIVE_THEME_UPDATE`, `ENABLE/DISABLE_PROXY`,
`SET_INVIDIOUS_AUTHORIZATION`, `START/STOP_POWER_SAVE_BLOCKER`,
`GET/TOGGLE_REPLACE_HTTP_CACHE`, `PLAYER_CACHE_GET/SET`, `GENERATE_PO_TOKEN`,
`CHOOSE_DEFAULT_FOLDER`, `WRITE_TO_DEFAULT_FOLDER`, `OPEN_IN_EXTERNAL_PLAYER`,
plus the `DB_*` CRUD channels and `SYNC_*` cross-window broadcast channels.

---

## 5. Preload Bridge (`src/preload/`)

Tiny and stable:
- `main.js`: `contextBridge.exposeInMainWorld('ftElectron', api)`
- `interface.js`: a plain object of methods, each calling
  `ipcRenderer.send`/`ipcRenderer.invoke` on defined channels, or
  `ipcRenderer.on` to register renderer-side handlers (e.g.
  `handleChangeView`, `handleOpenUrl`, `handleUpdateSearchInputText`,
  `handleSyncSettings`, `handleSyncHistory`, …).
- `preload-interface.d.ts`: declares `window.ftElectron` globally so JS type-checking
  (jsconfig) knows the API.

**Pattern for adding system features:** 1) add channel(s) to `constants.js`,
2) implement `ipcMain` handler in `src/main/index.js`,
3) add a typed wrapper method in `interface.js`,
4) call `window.ftElectron.<method>()` from renderer.
---

## 6. Renderer (Vue 3 app, `src/renderer/`)

### 6.1 Entry & shell
- `main.js` builds the app with `router`, `store`, `i18n`, registers Font Awesome
  (explicit icon list) and Swiper, mounts `#app`, and wires a couple of
  Electron-only handlers.
- `App.vue` is the shell: `TopNav`, `SideNav`, `FtFlexBox > RouterView`, plus
  global overlays (prompts, toasts, progress bar, shortcut prompt, playlists
  add-video prompt, update banner).

### 6.2 Routing (`router/index.js`)
Hash-based routes — each maps to a **view** folder under `views/`:
`/subscriptions`, `/subscribedchannels`, `/trending` (conditional on
`SUPPORTS_LOCAL_API`), `/popular`, `/userplaylists`, `/history`, `/settings`,
`/settings/profile`, `/about`, `/search/:query`, `/playlist/:id`,
`/channel/:id/:currentTab?`, `/watch/:id`, `/hashtag/:hashtag`, `/post/:id`.
→ New top-level pages (e.g. a "Local Videos" library) = new view + route +
  SideNav/TopNav entry.

### 6.3 Vuex store (`store/`)
Modules (each `{ state, getters, mutations, actions }`):
`history`, `invidious`, `playlists`, `profiles`, `settings`, `search-history`,
`subscription-cache`, `utils`, `player`.

**Settings module is notable:** a `state` object of simple settings auto-generates
getter `getX`, mutation `setX`, action `updateX`
(persist → DB → commit). Settings needing side effects (e.g. backend change,
theme change, proxy toggle) are listed in `settingsWithSideEffects` with a
`sideEffectHandlers` entry (fires DB write + dispatch of the side-effect action).
Complex/custom settings go in `customState`/`customGetters`/`customMutations`/`customActions`.
→ Adding a setting for the local-video feature is mostly declarative.
### 6.4 Datastores & handler indirection
Six NeDB databases, kept in Electron's `userData` dir (`<name>.db`):

| DB | Purpose |
|---|---|
| `settings` | flat key/value (`_id` = setting name) |
| `profiles` | channel filters / profiles (default profile `allChannels`) |
| `playlists` | user playlists (videos + progress) |
| `history` | watch history incl. watch progress, last-viewed playlist |
| `search-history` | search box history |
| `subscription-cache` | per-channel cached videos/live/shorts/posts |

Handlers: `base.js` (implementation, used by main process and web),
`electron.js` (IPC proxy used by Electron renderer — selected via webpack alias
`DB_HANDLERS_ELECTRON_RENDERER_OR_WEB`), `web.js` (thin wrapper over base for web).
`DBActions` (create/find/upsert/delete/… + domain-specific ops) and `SyncEvents`
are centralized in `constants.js`. Write operations broadcast `SYNC_*` events to
other open windows, which the renderer applies to its store.

### 6.5 Data-fetching backends
- **`helpers/api/local.js`** (2500 lines): full InnerTube client on top of
  youtubei.js — sessions, `createInnertube()`, deciphering formats/player,
  DASH manifest assembly (`createLocalDashManifest`), search/trending/channel/
  playlist/comments/hashtag/community-post endpoints, and numerous `parseXxx`
  normalizers.
- **`helpers/api/invidious.js`** (~1100 lines): REST client for an Invidious
  instance (`/api/v1/...`), with image proxying, format mapping
  (`convertInvidiousToLocalFormat`, `mapInvidiousLegacyFormat`), and a local
  DASH manifest generator.
- **Both map YouTube/Invidious data to a common normalized shape** that views and
  list components consume (e.g. `FtListVideo` fields).
### 6.6 The Watch flow (playback pipeline)
`views/Watch/Watch.vue` + `Watch.js`:
1. Route `/watch/:id` → `getVideoInformationLocal()` or `getVideoInformationInvidious()`
   based on `backendPreference` (+ fallback logic).
2. Video info is destructured into dozens of `data` fields (title, channel,
   description, chapters, captions, live chat, recommendations…).
3. Streaming data is turned into one of:
   - **DASH manifest** (`manifestSrc` + mime `application/dash+xml`),
   - **HLS manifest** (`application/x-mpegurl`),
   - **SABR** live-stream metadata (`sabrData`),
   - **legacy formats** (`legacyFormats[]` — direct progressive MP4/WebM URLs).
4. `ft-shaka-video-player` receives these props and drives Shaka Player.
   Custom Shaka bits: `SabrSchemePlugin` + `SabrManifestParser`
   (YouTube's live "SABR" format), segment parsers
   (`EbmlParser`, `WebmSegmentIndexParser`, `Mp4SegmentIndexParser`), custom
   UI controls (`player-components/*`), SponsorBlock integration, screenshots,
   PiP/full-window/theatre-mode, playback-rate ranges, error translation.
5. Watch progress → `history` DB; playlists auto-advance; recommendations feed
   the sidebar.
→ **Shaka can natively play a progressive MP4/WebM file** the same way it plays the
  "legacy format" URLs today; a local-file feature can reuse `ft-shaka-video-player`
  almost unchanged, given a URL the renderer is allowed to fetch.
---

## 7. i18n / Localization

- Sources: `static/locales/*.yaml` (Weblate). `activeLocales.json` whitelists.
- `_scripts/ProcessLocalesPlugin.js` compiles YAML → JSON at build time,
  brotli-compressed (`.json.br`) in production for size.
- `src/renderer/i18n/index.js` lazy-loads the active locale via `fetch`
  (`/static/locales/<locale>.json[.br]`), fallback chain → `en-US`.
- Every user-facing string goes through `$t('Key')`; when adding UI, keys must be
  added to **all** locale files (English source of truth; CI checks missing templates
  via `_scripts/findMissingTemplates.mjs`).

---

## 8. Build, Dev, and CI

### Dev
```bash
pnpm install
pnpm run dev          # webpack-dev-server :9080 + Electron (renderer hot-reload)
pnpm run dev:web      # web build only (PWA)
pnpm run debug        # + --remote-debug (port 9222 inspector / 9223 remote)
```
`dev-runner.js` orchestrates: starts webpack in watch mode for main/preload/renderer
(and botGuardScript), then Electron on `dist/main.js`; exit code `69` triggers an
app relaunch; locale hot reload over WebSocket.

### Production
```bash
pnpm run pack         # pack:main + pack:renderer + pack:preload + botGuardScript
                     # then injectAllowedPaths.mjs (inline app://bundle allowlist)
pnpm run build-release            # or build-release:arm64 / :arm32
```
`build.mjs` drives electron-builder per platform/arch (config `ebuilder.config.mjs`).
The `dist/` bundle is the only thing packaged (node_modules excluded).

### Lint / checks
```bash
pnpm run lint-all     # eslint + stylelint (+ json/yml)
pnpm run checkforbadtemplates  # locale template sanity
```
CI (`.github/workflows/build.yml`, `linter.yml`, `codeql.yml`, `release.yml`)
runs lint + build on the `development` branch, creates nightly builds on `nightly`.
---

## 9. Conventions & Patterns to Follow

1. **Always extend existing structures** rather than inventing parallel ones:
   new IPC channel + `constants.js` entry + `main/index.js` handler + preload
   `interface.js` method.
2. **Settings are declarative**: add to `state` in `store/modules/settings.js`
   (with `sideEffectHandlers` when Electron/system side effects are required).
3. **New datastores**: add NeDB instance in `src/datastores/index.js` +
   handler class in `base.js`/`electron.js` (+ `web.js` if web target) +
   `DBActions`/`SyncEvents` constants.
4. **Normalized API shape**: any new data source goes through
   `helpers/api/` and returns the same shapes as `local.js`/`invidious.js`
   so list components (`FtListVideo`, `FtListChannel`, …) work unchanged.
5. **Guard all IPC** with `isFreeTubeUrl(event.senderFrame.url)`.
6. **No direct filesystem access in the renderer** — route through IPC/custom
   protocols; keep the `app://bundle` allowlist tight.
7. **i18n every string** in all `static/locales/*.yaml`.
8. **Style**: SASS + Stylelint (a11y rules), ESLint with heavy plugin set
   (unicorn, jsdoc, promise, import-x, vue, yml), `.vue` files may use
   `<script src>` + `<style scoped lang="scss">` (see `Watch.vue`).

---

## 10. Feature Prep Notes — Viewing Local Videos (user's machine)

No local file playback exists today; the app is 100% YouTube/Invidious oriented.
Based on the architecture, the feature will likely touch:

| Concern | Existing hooks / integration points |
|---|---|
| **Picker UI** | Dialog via new IPC (like `CHOOSE_DEFAULT_FOLDER`/`WRITE_TO_DEFAULT_FOLDER` + `dialog.showOpenDialog` in main) or the Web File System API (already permitted: `fileSystem` non-directory permission + `file-system-access-restricted` allow) |
| **Serving the file to Shaka** | Shaka already plays progressive video from plain URLs (the `legacyFormats` path). The renderer cannot read `file://` today (permission handlers + `app://` allowlist). Options: (a) new custom protocol `localfile://<id>` handled in main with an allowed-id map; (b) IPC chunk/byte-range reader with a blob URL in renderer memory; (c) purposely extend file-system permission to a user-chosen file/folder only |
| **Playback UI** | Reuse `ft-shaka-video-player` with `format='legacy'` + a single progressive format entry and no manifest. Chapters/captions can be omitted or user-added |
| **Library/list UI** | New or extended view + route (e.g. `/local`) + SideNav entry; reuse `FtListVideo`-style cards; thumbnails would need local generation (frame grab via `<canvas>` on the `<video>`) since no remote thumbnails exist |
| **Persistence** | Watch progress/history is keyed by YouTube `videoId` strings; local files need their own id scheme (file hash or path) and likely a new datastore + Vuex module (or reuse `history` with a distinct id namespace) |
| **Settings** | Declarative additions in `settings.js` (e.g. default local media folder, autoplay next local file) |
| **Security** | Keep the "explicit user grant per file/folder" model; never turn on blanket file-system access |
| **Format support** | Shaka codecs: MP4 (H.264/AAC), WebM/VP8/VP9/Opus; the bundled segment-index parsers make some MKV/WebM variants possible; anything exotic needs the external-player fallback (already exists: `OPEN_IN_EXTERNAL_PLAYER`) |
| **i18n** | Every new label across all `static/locales/*.yaml` |

### 10.1 Implementation Roadmap

#### Step 1: Custom Protocol for Local Files (main process)
- Register a new privileged scheme (e.g., `localfile://`) in `src/main/index.js` alongside the existing `imagecache://` and `app://` schemes.
- Implement a `protocol.handle('localfile', ...)` handler that validates requests against an in-memory `Map<string, filePath>` of user-approved files.
- Only serve files that have been explicitly approved via the file picker (store the mapping with a UUID key).
- This mirrors the existing `imagecache://` pattern which already serves cached images through a custom protocol.

#### Step 2: IPC Channels for File Selection
- Add new IPC channels to `src/constants.js`:
  - `CHOOSE_LOCAL_FILE` — opens `dialog.showOpenDialog` with video file filters (`.mp4`, `.webm`, `.mkv`, etc.)
  - `GET_LOCAL_FILE_INFO` — returns metadata for a local file
  - `REVOKE_LOCAL_FILE_ACCESS` — removes a file from the approved map
- Implement handlers in `src/main/index.js` (following the existing `CHOOSE_DEFAULT_FOLDER` pattern).
- Add wrapper methods to `src/preload/interface.js`.

#### Step 3: Local Video Data Layer
- Create a new Vuex module `localVideos` in `src/renderer/store/modules/local-videos.js`:
  - State: `localVideosById` (Map), `localVideosSorted` (Array)
  - Actions: `addLocalVideo`, `removeLocalVideo`, `grabLocalVideos`, `updateLocalVideoWatchProgress`
- Either reuse the existing history datastore (with a distinct id prefix like `local_`) or create a new NeDB datastore.
- Reuse the `DBHistoryHandlers` pattern (with `videoId`, `watchProgress`, `title`, `thumbnail`, etc.).

#### Step 4: Local Video Watch View
- Create `src/renderer/views/LocalWatch/LocalWatch.vue` and `LocalWatch.js`:
  - Similar to `Watch.vue`/`Watch.js` but without YouTube-specific features (comments, live chat, recommendations, etc.)
  - Use `ft-shaka-video-player` with `format='legacy'` and a single progressive format entry.
  - Pass the `localfile://<uuid>` URL as the `legacyFormats` entry's `url`.
  - Implement local thumbnail generation using a `<canvas>` frame grab from the video element.

#### Step 5: Local Video Library View
- Create `src/renderer/views/LocalLibrary/LocalLibrary.vue`:
  - Grid/list view of locally indexed videos using `FtAutoGrid` + `FtListVideo` (or a custom `FtListLocalVideo` component).
  - Reuse `FtListVideoNumbered` or `FtListVideo` patterns for consistency.
  - Add thumbnail extraction/generation on file addition.

#### Step 6: Routing & Navigation
- Add routes to `src/renderer/router/index.js`:
  - `/local` → `LocalLibrary` view
  - `/watch/local/:uuid` → `LocalWatch` view
- Add SideNav entry in `src/renderer/components/SideNav/SideNav.vue` (with `faFilm` or `faVideo` icon).
- Add TopNav support if needed.

#### Step 7: Settings Integration
- Add settings to `src/renderer/store/modules/settings.js` state:
  - `defaultLocalMediaFolder: ''` — default folder for the file picker
  - `autoplayLocalVideos: false` — autoplay next local file in library
  - `localVideoThumbnailExtraction: true` — auto-generate thumbnails
- Add corresponding UI in `src/renderer/components/PlayerSettings.vue` or a new settings component.

#### Step 8: i18n
- Add all new labels to `static/locales/en-US.yaml` (and other locale files):
  - `Local Videos`, `Open Local File`, `Local Library`, etc.

### 10.2 Key Files to Modify (in order)

| Order | File | Change |
|---|---|---|
| 1 | `src/constants.js` | Add new IPC channels (`CHOOSE_LOCAL_FILE`, etc.) |
| 2 | `src/main/index.js` | Register `localfile://` protocol, implement IPC handlers |
| 3 | `src/preload/interface.js` | Add `chooseLocalFile()`, `revokeLocalFileAccess()` methods |
| 4 | `src/renderer/store/modules/local-videos.js` | New Vuex module for local video state |
| 5 | `src/renderer/store/index.js` | Register new module |
| 6 | `src/renderer/views/LocalLibrary/*` | New library view |
| 7 | `src/renderer/views/LocalWatch/*` | New watch view for local files |
| 8 | `src/renderer/router/index.js` | Add `/local` and `/watch/local/:uuid` routes |
| 9 | `src/renderer/components/SideNav/SideNav.vue` | Add nav entry |
| 10 | `src/renderer/store/modules/settings.js` | Add local video settings |
| 11 | `static/locales/en-US.yaml` | Add new i18n labels |

### 10.3 Security Considerations

- **Never enable blanket `file://` access** in the renderer. The current `app://` allowlist and permission handlers must remain strict.
- The `localfile://` protocol should only serve files from an explicit allowlist (Map<UUID, filePath>).
- File paths should never be exposed to the renderer directly — use UUIDs as identifiers.
- Revoke access when files are removed from the library or the app closes (unless persistence is desired).
- Follow the existing pattern: every IPC handler must validate with `isFreeTubeUrl(event.senderFrame.url)`.
---

## 11. Quick Reference — Key Files

| File | Why it matters |
|---|---|
| `src/constants.js` | Single source of truth: IPC channels, DB actions, sync events, shortcuts |
| `src/main/index.js` | All system services, IPC handlers, protocols, session/network layer |
| `src/preload/interface.js` | Every renderer→main call goes through here |
| `src/renderer/App.vue` | App shell + global overlays |
| `src/renderer/router/index.js` | All routes |
| `src/renderer/store/index.js` + `modules/*` | Global state (settings module = auto-gen pattern) |
| `src/datastores/handlers/*` | DB access indirection (base/electron/web) |
| `src/renderer/helpers/api/local.js` & `invidious.js` | The two data backends, normalized output |
| `src/renderer/helpers/player/*` | Shaka integration (SABR, EBML, MP4/WebM segment parsers) |
| `src/renderer/components/ft-shaka-video-player/*` | The video player component |
| `src/renderer/views/Watch/*` | The watch page pipeline (data → manifest → player) |
| `static/locales/*.yaml` | All user-facing strings |
| `_scripts/webpack.*.config.js` | Build flags, aliases, bundling |