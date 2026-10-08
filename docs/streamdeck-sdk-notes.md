# Stream Deck SDK notes (v3.0.0)

Source: https://docs.elgato.com/streamdeck/sdk/ and linked pages, fetched 2026-10-07.
These are working notes, not a copy of the docs. Items marked **(verify)** were inconsistent or unclear in the source.

## 1. Basics

- Plugins are Node.js backends (application layer) plus optional Chromium HTML views (property inspectors, "PI").
- Everything runs locally; the Stream Deck app owns hardware communication. Never ship secrets (private API keys) in a plugin.
- Prerequisites: Node.js 24+, Stream Deck app 7.1+, a device (Stream Deck Mobile works), VS Code recommended.
- CLI: `npm install -g @elgato/cli@latest`; alias `sd`. Scaffold with `streamdeck create`.
- Stream Deck 7.1+ auto-updates its bundled Node (20.20.0 and 24.13.1 as of 7.3). Chromium 130 as of 7.1-7.3.
- Plugin/action UUIDs: reverse-DNS, lowercase letters/digits/hyphens/periods. **Never change after publishing.** Action UUIDs are prefixed with the plugin UUID.

### Scaffolded layout

```
<uuid>.sdPlugin/
  bin/            compiled output
  imgs/
  logs/
  ui/             property inspector HTML
  manifest.json
src/
  plugin.ts       entry point
  actions/*.ts
package.json, rollup.config.mjs, tsconfig.json
```

### Dev loop

- `npm run build` (rollup), `npm run watch` (rebuild + `streamdeck restart <uuid>` on change).
- Manual: `streamdeck restart <uuid>` / `streamdeck stop <uuid>`.
- Debug: attach VS Code to the Node process (`Debug: Attach to Node Process`). Inspector port is auto-assigned unless `Nodejs.Debug` is set.
- PI debug: `streamdeck dev` (create enables it), open `http://localhost:23654/`. PI must be visible in the app.

## 2. Entry file and actions

```ts
import streamDeck, { action, SingletonAction, KeyDownEvent, WillAppearEvent } from "@elgato/streamdeck";

@action({ UUID: "com.example.plugin.counter" })
class Counter extends SingletonAction<{ count?: number }> {
	override async onKeyDown(ev: KeyDownEvent<{ count?: number }>) {
		const count = (ev.payload.settings.count ?? 0) + 1;
		await ev.action.setSettings({ count });
		await ev.action.setTitle(`${count}`);
	}
}

streamDeck.actions.registerAction(new Counter());
streamDeck.connect(); // always last
```

- Register all actions **before** `connect()`; `connect()` is the last call.
- One class instance per action type (`SingletonAction`); `this.actions` = visible instances of that type. `streamDeck.actions.forEach` iterates all visible ones. Use the loop variable, not `ev.action`, to update every instance.
- `Action` is a union of `KeyAction | DialAction | NeoInfobarAction`. Narrow with `isKey()` / `isDial()` / `isNeoInfobar()` before key/dial-only calls (shared events like `onWillAppear` cover all).
- Controllers in the manifest: `Keypad`, `Encoder`, `Neo`.
- Plugins cannot touch other plugins' actions.

### Events (override on SingletonAction)

`onWillAppear`, `onWillDisappear`, `onKeyDown`, `onKeyUp`, `onDialDown`, `onDialUp`, `onDialRotate`, `onTouchTap`, `onDidReceiveSettings`, `onDidReceiveResources`, `onSendToPlugin`, `onPropertyInspectorDidAppear`, `onPropertyInspectorDidDisappear`, `onTitleParametersDidChange`.

Global equivalents: `streamDeck.actions.onKeyDown(...)`, `streamDeck.ui.onDidAppear/onDidDisappear`, `streamDeck.ui.sendToPropertyInspector`.

`onWillAppear` also re-fires after system wake.

## 3. Keys

- States: at least one, two fully supported (toggle). Toggles automatically on press; set manifest `DisableAutomaticStates` to manage via `setState(0|1)`.
- Multi-actions: name each state; read `ev.payload.isInMultiAction` and `ev.payload.userDesiredState`.
- `setTitle(title?, {state, target})`, `setImage(image?, {state, target})`. Image: plugin-relative path, base64 data URL, or SVG string (wrap dynamic SVG in `encodeURIComponent` data URL). SVG/JPG/PNG/WEBP; **no animated GIF** via `setImage`.
- Precedence: user title/image > runtime `setTitle`/`setImage` > manifest defaults. `UserTitleEnabled: false` stops users overriding title.
- `showOk()` (green check), `showAlert()` (yellow triangle; pair with a log line).
- Limit programmatic updates to 10/sec.

## 4. Dials and touch strip (Stream Deck +, + XL)

- Encoder = dial + touch strip segment. Layout canvas 200x100.
- Built-in layouts: `$X1` (title, icon), `$A0` (title, full canvas pixmap), `$A1` (title, icon, value), `$B1` (+ bar), `$B2` (+ gradient bar), `$C1` (title, 2x icon+bar).
- Custom layout JSON: `$schema` https://schemas.elgato.com/streamdeck/plugins/layout.json, unique `id`, `controller: "Encoder"`, `items[]` with `key`, `type`, `rect [x,y,w,h]`.
- Items must be in bounds and not overlap at the same `zOrder` or the layout silently fails. `rect`, `type`, `key` can't change at runtime.
- Reserved item keys `title`, `icon`: user PI settings override plugin values.
- Manifest default: `Actions[].Encoder.layout`. Runtime: `setFeedbackLayout(idOrPath)`, `setFeedback({ key: value | {value, range:{min,max}, color, opacity, ...} })`. Text/pixmap take strings; bar/gbar take numbers.
- `setTriggerDescription({push, rotate, touch, longTouch})`; `undefined` resets to manifest.
- `onDialRotate` payload: `ticks` (signed, positive = clockwise), `pressed`. `onTouchTap`: `tapPos`, `hold`.
- Item types: `bar`, `gbar`, `pixmap`, `text`. Common props: `background`, `enabled`, `opacity` (0-1 step 0.1), `zOrder` (0-700). Bar props: `value`, `range`, `subtype` 0-4, `bar_bg_c`, `bar_border_c`, `bar_fill_c`, `border_w`; gbar adds `bar_h`. Text: `alignment`, `color`, `font.size/weight`, `text-overflow` (clip/ellipsis/fade). Colors: named, hex, or gradient `"0:#f00,0.5:yellow,1:#0f0"`.

## 5. Neo Infobar (Stream Deck Neo, needs app 7.6+)

- Display only, no input events. Manifest `Controllers: ["Neo"]`, `Software.MinimumVersion: "7.6"`.
- Canvas 232x50; layout JSON with `controller: "Neo"`.
- Call `setFeedbackLayout` on **every** `onWillAppear` (not persisted). `NeoInfobarAction` has no `setTitle`.

## 6. Settings

| | Action | Global |
|---|---|---|
| write | `ev.action.setSettings` | `streamDeck.settings.setGlobalSettings` |
| read | `ev.payload.settings` (preferred), `ev.action.getSettings()` (visible only) | `streamDeck.settings.getGlobalSettings<T>()` |
| change | `onDidReceiveSettings` | `streamDeck.settings.onDidReceiveGlobalSettings` |

- v3: change events fire only for PI-originated changes. `streamDeck.settings.useLegacySettingsBehavior = true` (before `connect()`) restores firing after reads.
- Settings are plain text and exported with profiles; no secrets. Global settings suit user-provided tokens/keys, not your own secrets.
- Types don't guarantee runtime shape: default on destructure or validate with Zod.
- Merge vs replace on write is undocumented **(verify)**.

## 7. Property inspector

- HTML under `ui/`, referenced by manifest `PropertyInspectorPath` (action-level overrides plugin-level).
- Use `sdpi-components` (v4), bundled locally, not the CDN, for production. Elements: `sdpi-item`, `-button`, `-checkbox`, `-checkbox-list`, `-color`, `-calendar`, `-delegate`, `-file`, `-password`, `-radio`, `-range`, `-select`, `-textarea`, `-textfield`. Bind with `setting="key"`.
- `const { streamDeckClient } = SDPIComponents; streamDeckClient.setSettings({...})`.
- Plugin to PI: `streamDeck.ui.sendToPropertyInspector`; PI to plugin: `onSendToPlugin`. Docs are "in development".
- Guidelines: save automatically (no Save button), checkboxes for booleans, validation feedback, no donation links.
- `@elgato/streamdeck` can't be imported in the browser (v2+).

## 8. Devices, profiles, resources

- `streamDeck.devices` array: `id, isConnected, name, size, type`. Events `onDeviceDidConnect/DidChange (7.0+)/DidDisconnect`.
- DeviceType: 0 Stream Deck/Scissor, 1 Mini, 2 XL, 3 Mobile, 4 Corsair GKeys, 5 Pedal, 6 Corsair Voyager, 7 Stream Deck +, 8 SCUF, 9 Neo, 10 Studio, 11 Virtual, 12 Galleon 100 SD, 13 + XL.
- Profiles: export `.streamDeckProfile` from the app, place in `.sdPlugin`, list in manifest `Profiles` (`Name` is a path without extension, `DeviceType`, `Readonly`, `DontAutoSwitchWhenInstalled`, `AutoInstall`). Switch with `streamDeck.profiles.switchToProfile(deviceId, profileName)`. Only bundled profiles, never user profiles.
- Resources (7.1+): `setResources({key: path})` / `getResources()` / `ev.payload.resources`. Key-to-path map only; paths are rewritten on import, so never persist them yourself.

## 9. System

- `streamDeck.system.openUrl(url)` (default browser, http(s) only).
- `onSystemDidWakeUp` (plugin only). `onWillAppear` also fires again on wake; avoid duplicate reconnect logic.
- App monitoring: manifest `ApplicationsToMonitor` (`mac`: bundle IDs, `windows`: exe names); `onApplicationDidLaunch/Terminate` give `ev.application`. Mac ID lookup: `osascript -e 'id of app "Name"'`.
- Deep links: `streamdeck://plugins/message/<PLUGIN_UUID>[/path][?query][#fragment]`; handler `streamDeck.system.onDidReceiveDeepLink` (`ev.url.path`, `.fragment`). Active (6.5+, foregrounds app) vs passive (7.0+, `?streamdeck=hidden`). Max 2000 chars, local only. OAuth proxy: `https://oauth2-redirect.elgato.com/streamdeck/plugins/message/<UUID>`; forwards only `code, state, scope, error`.
- `streamDeck.system.getSecrets` was removed in v3.

## 10. Logging and i18n

- `streamDeck.logger.info/warn/error/debug/trace`, `setLevel("warn")`, `createScope("Main")`. Defaults: DEBUG dev, INFO prod. Files: `<sdPlugin>/logs/<uuid>.<n>.log`, 10 files x 10 MiB, deleted on uninstall. App logs: macOS `~/Library/Logs/ElgatoStreamDeck/`.
- Localization: `en.json, de.json, es.json, fr.json, ja.json, ko.json, zh_CN.json, zh_TW.json (6.8+)` at the `.sdPlugin` root. Override manifest `Name`, `Description`; per action (keyed by action UUID) `Name`, `Tooltip`, `States[].Name`, `Encoder.TriggerDescription`. Custom strings under `Localization`; read with `streamDeck.i18n.translate(key, lang?)` (returns key if missing). PI localization is separate (sdpi-components).

## 11. Manifest essentials

Required top level: `Actions`, `Author`, `CodePath`, `Description`, `Icon` (256/512 PNG), `Name`, `OS` (1-2 entries), `SDKVersion` (2|3, use 3), `Software.MinimumVersion` (6.4-7.6), `UUID`, `Version` (major.minor.patch.build).
Optional: `ApplicationsToMonitor`, `Category`, `CategoryIcon`, `CodePathMac/Win`, `DefaultWindowSize`, `Nodejs {Version "20"|"24", Debug "enabled"|"break", GenerateProfilerOutput}`, `Profiles`, `PropertyInspectorPath`, `SupportURL (6.9+)`, `URL`.
Action: `Name`, `UUID`, `Icon` (20/40, white mono), `States[]` required; `Controllers`, `Tooltip`, `DisableAutomaticStates`, `DisableCaching`, `Encoder`, `OS`, `SupportedInKeyLogicActions (7.0+)`, `SupportedInMultiActions`, `UserTitleEnabled`, `VisibleInActionsList` (use to retire an action instead of changing its UUID).
State: `Image` (72/144) required; `Name`, `Title`, `ShowTitle`, `TitleAlignment`, `TitleColor`, `Font*`, `MultiActionImage`.
Encoder: `Icon` (72/144), `background` (200x100), `layout`, `StackColor`, `TriggerDescription {Push, Rotate, Touch, LongTouch}`.
Schema: https://schemas.elgato.com/streamdeck/plugins/manifest.json. Node plugins run with `--enable-source-maps`.
Do not read the manifest at runtime (DRM; `streamDeck.manifest` removed in v2).

## 12. Packaging and distribution

- `streamdeck validate [path]`, `streamdeck pack <x.sdPlugin>` produces `.streamDeckPlugin`; exclude files via `.sdignore` (gitignore syntax).
- DRM (CLI 1.6+ default): SDK v2+, `SDKVersion: 3`, `Software.MinimumVersion >= 6.9`; files immutable after distribution. Test DRM build by uploading to Maker Console (maker.elgato.com) with "Publish after review" off, then download from the Versions tab.
- Marketplace flow: guidelines review, app icon, gallery images, submit in Maker Console.
- Marketplace guidelines highlights: plugin name unique, no org name; 2-30 actions, names ~30 chars max; action icons SVG/PNG white mono on transparent; key icons 72x72 (+144); max 10 programmatic updates/sec; touch targets >= 35x35; `showAlert` for failures; PI saves automatically.

## 13. Upgrade notes

- v3 (needs app 7.1+): settings events PI-only, settings generic fixed to the class type, `Action` is a union, settings type required on `KeyAction<T>` etc., `getSecrets` removed.
- v2: `streamDeck.ui.current?.x` becomes `streamDeck.ui.x`, `ui.current` becomes `ui.action`; JSON/utility types moved to `@elgato/utils`; `logger.setLevel("trace")` takes strings; manifest removed at runtime; no browser import.
- SDK 7.x change highlights: 7.6 Neo Infobar; 7.1 resources + Node 24; 7.0 `onDeviceDidChange`, passive deep links; 6.9 `SupportURL`; 6.5 `dialPress` removed (use `dialDown/dialUp`).

## 14. Linting

`npm i -D @elgato/eslint-config @elgato/prettier-config`; `eslint.config.js`: `import { config } from "@elgato/eslint-config"; export default config.recommended;` (or `.strict`). Script `"lint": "eslint --max-warnings 0"`, `"prettier": "@elgato/prettier-config"`, plus the repo's `.editorconfig`. Style: tabs, width 120, double quotes, semicolons, trailing commas, sorted imports.

## 15. Raw WebSocket protocol (non-Node plugins)

Launch args `-port -info -pluginUUID -registerEvent`; connect, send `{event: registerEvent, uuid: pluginUUID}`. Commands: `setTitle, setImage, setState, setSettings, getSettings, setGlobalSettings, getGlobalSettings, setFeedback, setFeedbackLayout, setTriggerDescription, showAlert, showOk, switchToProfile, openUrl, logMessage, sendToPropertyInspector, setResources, getResources`. Action `context` is not stable across app restarts.

## 16. Not fetched yet

CLI per-command pages (`/streamdeck/cli/commands/*`), SDK v1/v2 docs, Marketplace "Become a Maker", samples repo (https://github.com/elgatosf/streamdeck-plugin-samples), sdpi-components docs (https://sdpi-components.dev), and the HID API / icon / profile / OBS / iCUE sections.
