# Home Lights — Implementation Plan

## Context

`E:\HomeLights` is an empty npm-workspaces skeleton (`Application`, `Documentation`, `Package/*`, `Script/*`) scaffolded from `E:\Jotfolio` (same ESLint config, `.vscode`, beads). Goal:

- **Home Lights** (Expo, TypeScript, Android + iOS): after a one-time 6-digit passcode, shows a single column of Hue-style controls — one per **Hue room** and one per **Govee light** (2 Govee lights, each alone in its room). Each control has a power toggle and brightness slider; tapping elsewhere opens a one-to-one recreation of the Hue room screen (scenes, per-light on/off + brightness, color / white picker with per-light pins) **minus** the room/light settings entry points and the Automations section. Hue-quality **optimistic rendering** throughout.
- **`Package/UI`**: components that look and behave identically to the Hue app's, derived from studying the real Hue app on Gage's Android phone via `agent-device` + wireless ADB.
- **`Package/LocalBackend`**: runs on Gage's Raspberry Pi; talks to the Hue bridge and Govee lights; serves the app over the LAN. Configured manually on the Pi (Govee device IDs given by Gage; Hue pairing via a subcommand).
- **No Supabase, no accounts.** Security is deliberately simple and weak: a 6-digit passcode.
- **Effect everywhere**, functional/effectful style across the monorepo. Effect platform services (`FileSystem`, `Path`, `HttpClient`, `DatagramSocket`, `Terminal`, …) instead of `node:*` modules wherever a service exists.

### Decisions (confirmed with Gage)
- Platforms: Android + iOS (Android first; iOS via EAS — no Mac).
- Hue scenes: **included** on the room screen (apply existing scenes only; no create/edit). Govee screens have no scenes section.
- Pi address: **baked into the app** at build time (Gage gives the Pi a DHCP reservation).
- Passcode: stored in the app as a build-time constant; entered once per device; never expires.

### Effect 4.1.2 surface (verified from the published tarballs)
- `effect/cli` (`Command`, `Flag`, `Argument`, `Prompt`), `effect/rpc` (`RpcGroup`, `RpcServer`, `RpcClient`, `RpcSerialization`, `RpcMiddleware`, `RpcTest`), `effect/socket` (`Socket`, `SocketServer`, `DatagramSocket`), `effect/http`, `effect/reactivity` (`Atom`, `AtomRpc`, `AsyncResult`).
- `@effect/platform-node@4.1.2`: `NodeRuntime`, `NodeServices`, `NodeFileSystem`, `NodePath`, `NodeHttpClient`/`Undici`, `NodeHttpServer`, `NodeSocketServer`, `NodeDatagramSocket`, `NodeTerminal`.
- `@effect/atom-react@4.1.2` for React bindings. Pin all three exactly at `4.1.2`.

### Conventions (from `E:\Jotfolio`)
- PascalCase identifiers/props, Allman braces, 4-space indent, JSDoc file headers (`@module`, `@file`, `@author Gage Sorrell <gage@sorrell.sh>`, `@copyright`, `@license MIT`), enforced by `@sorrell/eslint-config` + root [eslint.config.js](eslint.config.js).
- Packages build with `tsc` to `Distribution/`, tsbuildinfo in `Intermediate/`, `exports` maps; mirror `E:\Jotfolio\Package\UI\{package.json,tsconfig.json}` (extends `@sorrell/tsconfig/expo` for RN packages, `@sorrell/tsconfig` for Node).
- Styling via a `MakeStyles` helper like `E:\Jotfolio\Package\UI\Source\MakeStyles.ts`; Storybook in the app behind `STORYBOOK_ENABLED` (copy Jotfolio's `.rnstorybook` + `withStorybook` metro pattern).
- Use `@sorrell/effect` utilities where they fit. Style: no classes or mutable singletons; `Effect.gen`, services + `Layer`s, `Schema` for every boundary (config file, wire protocol, Hue/Govee payloads), `Data.TaggedError` errors, `Option` instead of null/undefined in domain code, pure functions for transforms.
- Task tracking with **`bd`**: one epic + an issue per phase. Durable decisions via `bd remember`. No commits/pushes unless asked.

---

## Repository layout (target)

```
Application/             Expo SDK 58 app "Home Lights" (expo-router, TS); id sh.sorrell.homelights
Package/Domain/          @homelights/domain — shared Schemas, RpcGroup, color math, optimistic reconcile (pure, platform-agnostic)
Package/UI/              @homelights/ui — Hue-look React Native components
Package/LocalBackend/    @homelights/local-backend — Pi service + CLI (Effect, @effect/platform-node)
Documentation/HueReference/  component/screen specs from the Hue app (raw captures gitignored)
```

---

## Phase 0 — Bootstrap

1. `bd` epic + phase issues.
2. Root `package.json`: devDeps (`eslint`, `@sorrell/eslint-config`, `eslint-config-expo`, `eslint-plugin-react-perf`, `typescript`, `@sorrell/tsconfig`, `vitest`, `@effect/vitest`), `engines` (node 24.19.0, npm ≥12), author/repo fields, react `overrides`, and `build:packages` / `eas-build-post-install` scripts as in Jotfolio.
3. Scaffold `Application` with `npx create-expo-app@latest` (SDK 58, router + TS); move routes to `Source/App`; set name "Home Lights", slug `home-lights`; add `expo-dev-client`.
4. Android builds use `JAVA_HOME="C:\Program Files\Android\Android Studio\jbr"` (no `java` on PATH); `ANDROID_HOME` is already `C:\Android\Sdk-Quail3`.
5. Trim Jotfolio leftovers in [eslint.config.js](eslint.config.js) (Next/shadcn/`Application/Mobile` globs → `Application/**`) and the `.vscode/tasks.json` task pointing at a nonexistent `Script/PatchReactNativeTools` (ask before removing the RN-Windows launch configs).
6. `.gitignore`: `node_modules`, `Distribution`, `Intermediate`, `.expo`, `android`, `ios`, `Documentation/HueReference/Capture/`, `.env*`.

## Phase 1 — Study the Hue app on Gage's phone (early; blocks UI work)

Gage enables wireless debugging → `adb pair` / `adb connect` → `agent-device devices` → `agent-device open` Hue.

For each target: `snapshot` (a11y tree + exact bounds → dp), `screenshot`, and `record` interaction videos (toggle, slider drags, pin drags incl. multi-pin group drag, scene tap, screen transitions, press states) to derive timing and spring curves. Capture rooms with lights on (color + white), off, mixed, and unreachable. Interacting changes real lights.

Targets:
- **Home screen**: header, room card (gradient from the room's light colors, icon, name, status line, toggle, brightness slider), column layout/spacing, pull-to-refresh, any bottom tab bar.
- **Room screen**: header (omit settings/edit/⋯ entries), scenes section (tiles, active state, "see all" if present), lights list rows (icon, name, toggle, slider), entry to the color picker. Note the **Automations** section and settings affordances to *exclude*.
- **Color picker**: Color / White tabs (plus any other mode tabs; capture, then decide), wheel with one pin per light, group/individual pin behavior, brightness bar, light chips/legend.
- Tokens: dark palette, type scale, spacing, radii, elevation, haptics, motion.

Output: `Documentation/HueReference/Tokens.md` + one spec per component/screen (dimensions, colors, states, gestures, animation params, screenshot refs). Raw captures → `Documentation/HueReference/Capture/` (gitignored). **Fidelity boundary:** match layout, color, sizing, motion and behavior exactly, but redraw icons as our own SVGs and use the closest open font; don't extract Philips' proprietary fonts/icons/assets from the APK.

## Phase 2 — `Package/Domain` (`@homelights/domain`)

Shared by the app and the Pi; depends only on `effect`.
- **Schemas:** `TargetId` (`HueRoom` | `GoveeLight`, tagged), `LightId`, `SceneId`; `LightState { On, Brightness (0–100), Color: Xy{X,Y} | Temperature{Mirek}, Reachable }`; `LightCapabilities { Gamut: Option, MirekRange: Option }`; `Target { Id, Name, Archetype/Icon, Lights[], Scenes[] }`; `HomeSnapshot`.
- **`HomeRpcs` (`RpcGroup`)**: `WatchHome` (stream of `HomeSnapshot`, sends a full snapshot then incremental updates), `SetTarget(Id, Patch{On?, Brightness?})`, `SetLight(Id, Patch{On?, Brightness?, Color?})`, `RecallScene(Id)`. Errors as tagged schemas (`Unreachable`, `BridgeError`, `Unauthorized`). Passcode auth via an `RpcMiddleware`.
- **`Color`**: xy↔sRGB (gamut A/B/C clamp), mirek↔kelvin, xy↔wheel coordinates, room gradient derivation — pure functions.
- **`Optimistic`**: pure reconcile — `Confirmed` + `Desired` + `LastIntentAt` + in-flight count per light/target → rendered state; server updates ignored while in flight or within a ~2 s grace window of the last intent; desired cleared once confirmed matches; revert on failure.
- Tests: `@effect/vitest` (color round-trips, reconcile edge cases with `TestClock`).

## Phase 3 — `Package/LocalBackend` (`@homelights/local-backend`, runs on the Pi)

Entry: `NodeRuntime.runMain(Command.run(...)).pipe(Effect.provide(NodeServices.layer))`. Services as `Context.Tag` + `Layer`.

**Config** (`ConfigStore` service): `Schema`-validated JSON at `$XDG_CONFIG_HOME/home-lights/Config.json` (default `~/.config/...`, override via `--config` flag), read/written atomically with `FileSystem` + `Path` (write temp, rename). Shape: `{ Passcode, Port, Hue: Option<{ BridgeId, Address, AppKey, ClientKey }>, Govee: Array<{ DeviceId, Name }> }`.

**CLI (`effect/cli`)**:
- `serve [--port]` — run the service.
- `hue pair [--address]` — discover via `https://discovery.meethue.com` (or `--address`), `Prompt` "press the link button", poll `POST /api {devicetype:"home_lights#pi", generateclientkey:true}` until success, verify the bridge ID, save.
- `govee scan` — multicast scan, prints DeviceId / SKU / IP (to find IDs). `govee add <DeviceId> --name <Room>`, `govee remove <DeviceId>`, `govee list`.
- `passcode set <6 digits>`.
- `status` — print the current home snapshot (handy for protocol smoke tests from the PC or the Pi).

**Hue (`HueBridge` service, CLIP v2 over HTTPS)**: `HttpClient` from `NodeHttpClient` with a custom `Undici` dispatcher that trusts the bundled Signify Hue root CA and checks the cert CN against the paired bridge ID (bridge certs aren't issued for its IP). Reads `room`, `device`, `light`, `grouped_light`, `scene`; streams `/eventstream/clip/v2` (SSE) into the state; reconnects with `Schedule` backoff. Commands: `PUT light/{id}` (`on`, `dimming`, `color.xy`, `color_temperature.mirek`), `PUT grouped_light/{id}` (room toggle/brightness), `PUT scene/{id} {recall:{action:"active"}}`.

**Govee (`GoveeLan` service)**: `DatagramSocket` via `NodeDatagramSocket` — bind `:4002`, multicast scan to `239.255.255.250:4001` on start and periodically to map configured DeviceIds → IPs; unicast to `:4003`: `turn`, `brightness`, `colorwc` (xy→RGB via Domain `Color`; mirek→kelvin), `devStatus` polled ~3 s. Schemas for each message. LAN Control must be enabled per device in Govee Home; the BLE-only desk strip simply isn't configured.

**State (`Home` service)**: `SubscriptionRef<HomeSnapshot>` merged from Hue events + Govee polls; `WatchHome` = `SubscriptionRef.changes`.

**Command pipeline**: per-target latest-wins coalescing (sliding `Queue` of capacity 1 per target, worker fiber). Rate limits: Hue lights ≈10 req/s shared, `grouped_light` ≈1/s (trailing value always sent), Govee ~10/s per device. Results feed back as acks / tagged errors.

**Server**: `RpcServer` over WebSocket (`NodeHttpServer` + `RpcServer.layerProtocolWebsocket`, JSON serialization); passcode middleware checks the `passcode` query param on upgrade against config.

**Deploy**: `Deploy/home-lights.service` (systemd, `Restart=always`, Node 24 arm64) + `Deploy/README.md` (install, `hue pair`, `govee scan/add`, `passcode set`, enable service).

Tests: `@effect/vitest` with test layers (fake `DatagramSocket`, fake `HttpClient`) for Govee codec, Hue payload decoding, coalescing/rate-limiting with `TestClock`, and config round-trip on an in-memory `FileSystem`.

## Phase 4 — `Package/UI` (`@homelights/ui`)

- Structure like Jotfolio: `Source/Token/` (from Phase 1), `Source/Primitive/` (Toggle, BrightnessSlider, Header, ListRow, Button, PasscodeKeypad/digit cells, Sheet, Toast, icon set), `Source/Block/` (RoomCard, SceneTile/SceneSection, LightRow, ColorWheel + TemperatureWheel with multi-pin support, ColorPickerTabs, RoomScreen layout).
- Built on `react-native-reanimated` + `react-native-gesture-handler` (UI-thread gestures), `react-native-svg`, `expo-linear-gradient`, `expo-haptics`; peer-dep pattern as in Jotfolio. Uses Domain's `Color` for gradients/wheel mapping.
- Components are **controlled, presentational, and backend-agnostic**: props like `On`, `Brightness`, `Pins: [{ Id, Color }]`, `OnChange` (continuous during drag) / `OnChangeEnd`. They render local drag state immediately.
- A story per component in `Application/.rnstorybook/stories`.
- Parity check: Storybook on the phone → `agent-device screenshot` per story → compare side-by-side with the Hue capture; iterate.

## Phase 5 — The app (`Application`)

- **Build-time config** (`app.config.ts` `extra`, or `EXPO_PUBLIC_*`): `HOME_LIGHTS_PASSCODE`, `LOCAL_BACKEND_URL` (`ws://<pi-ip>:<port>/rpc`).
- **Passcode gate**: Hue-styled 6-digit entry; compare to the constant; on success persist an "unlocked" flag in `expo-secure-store` (no expiry); the passcode also travels as a query param on the RPC socket so the Pi rejects other LAN clients.
- **RPC client**: `RpcClient` for `HomeRpcs` over `Socket.layerWebSocket` (RN global `WebSocket`) with reconnect `Schedule`; exposed through `AtomRpc` / `Atom.runtime` layers.
- **State**: `effect/reactivity` Atoms + `@effect/atom-react`. A `HomeAtom` subscribes to `WatchHome`; per-target/per-light derived atoms apply Domain `Optimistic` reconcile over the server snapshot. User actions write `Desired` synchronously (the UI updates that frame), then fire RPC calls throttled ~10/s per target during drags with a guaranteed trailing send; failures revert and show the Hue-style unreachable/error state.
- **Routes (expo-router)**: `Unlock`; `index` — single column of RoomCards (Hue rooms + Govee lights, ordered as the Hue app orders rooms, Govee appended/sorted by name); `Target/[id]` — room screen (scenes for Hue rooms, light rows; no settings, no Automations); `Target/[id]/Color` — color/white picker with per-light pins + brightness. Govee targets use the same screens with one light and no scenes.
- **Platform config**: Android `usesCleartextTraffic` (ws:// on LAN) via `expo-build-properties`; iOS `NSLocalNetworkUsageDescription` + `NSAppTransportSecurity.NSAllowsLocalNetworking` (no multicast entitlement needed — the Pi does discovery).

## Phase 6 — Builds

- Android dev build: `npx expo run:android` to the phone over wireless ADB.
- iOS: EAS build (needs Apple Developer account); install via internal distribution/TestFlight.

---

## Verification

1. `npm run lint` and `tsc --noEmit` in every workspace; `vitest` for Domain + LocalBackend.
2. From the PC first (same LAN, Node 24): `local-backend hue pair`, `govee scan/add`, `status` lists every Hue room/light/scene and both Govee lights; then on the Pi under systemd.
3. RPC: a small `RpcTest`-based test plus a manual `WatchHome` session; a wrong passcode is rejected.
4. Phone via `agent-device`: unlock once, relaunch → no prompt; toggle/slide each room card and Govee card; open room screens, apply a scene, toggle individual lights, drag single and group pins on the color and white wheels. Confirm instant response with no snap-back, external changes from the real Hue app showing up live (eventstream), and revert + error state when the Pi is stopped mid-command.
5. Visual parity: app screens screenshotted next to the matching Hue captures (room list, room screen, picker).
6. iOS EAS build controls the same lights.

## Prerequisites from Gage (at execution time)
- Wireless debugging enabled on the phone (pairing IP/port + code).
- Pi reachable (SSH) with Node 24 and a DHCP reservation; desired port and passcode.
- Press the Hue bridge link button during `hue pair`; LAN Control enabled for both Govee lights.
- Apple Developer account access for iOS builds (later).
