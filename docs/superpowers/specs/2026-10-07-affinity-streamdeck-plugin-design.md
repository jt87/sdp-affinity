# Affinity 2 Stream Deck Plugin: Design

Date: 2026-10-07. Status: approved by owner 2026-10-07.
Background: `docs/streamdeck-sdk-notes.md` (Stream Deck SDK v3), `docs/research/` (Affinity menu shortcuts).

## Goal

A Stream Deck plugin with curated shortcut buttons for Affinity Designer 2, Affinity Photo 2 and Affinity Publisher 2 on macOS.

## Decisions

| Decision | Choice |
|---|---|
| Control mechanism | Keyboard shortcuts only. No menu-item invocation. |
| Platform | macOS only. |
| Controllers | Keys only (`Keypad`). No dials, no Neo Infobar. |
| Action layout | Three actions (Designer, Photo, Publisher), each with a command dropdown. |
| App not running | The press launches the app and sends nothing. |
| App running, not frontmost | Activate, wait until frontmost, settle, then send. |
| Remapped shortcuts | Catalog defaults plus a per-key override. |
| Command set | Commands whose shortcut includes ⌘, ⌥ or ⌃ (⇧ alone does not count). Single-key and ⇧-only shortcuts (tools such as V, P, B; ⇧X) are excluded, because they type text while a text field is being edited. |
| Icons | One SVG per category; the command name is the key title. |

## Research findings

Affinity 2 (installed 2.6.5, Mac App Store builds):
- No AppleScript dictionary, CLI or command URL scheme. The scripting SDK/MCP connector is Affinity 3 only. Keystrokes are the only control surface.
- Bundle IDs: `com.seriflabs.affinitydesigner2`, `com.seriflabs.affinityphoto2`, `com.seriflabs.affinitypublisher2`. App Store process names end in ` MAS`, so the plugin always targets by bundle ID.
- Default menu shortcuts were read from the live apps into `docs/research/affinity-{designer,photo,publisher}2-menu-shortcuts.md`; these are the catalog's source of truth. Menus were read in one persona each, so persona-specific items may be missing.
- Shortcuts include non-character keys (arrows, ⌫, ↩, F5/F6, Page Up/Down, Space, Tab).
- The same command can live under different menus per app (Group: `Layer` in Designer/Publisher, `Arrange` in Photo). Catalogs are per app; shared commands share id and shortcut only.
- User shortcut customizations are in sandboxed binary `preferences.dat`, which the plugin cannot read.
- Recorded macros (`.afmacro`) exist in v2; triggering them is a possible later feature.

Stream Deck (installed 7.6.0; official samples and `streamdeck create` template reviewed):
- Use `@elgato/streamdeck` ^3.0.1 (npm latest; the samples still use v2). v3 needs Stream Deck 7.1: manifest `SDKVersion: 3`, `Software.MinimumVersion: "7.1"`, `Nodejs.Version: "24"`, `OS: [{ Platform: "mac", MinimumVersion: "12" }]`.
- Build with the template's rollup setup.
- `CategoryIcon` is required in practice: Stream Deck 7.6 rejects a plugin without it ("category icon not defined"), even though the docs call it optional and `streamdeck validate` passes.
- The CLI's `restart`/`stop` silently do nothing until developer mode is on (`streamdeck dev`).
- Property inspector dropdowns are populated with the sdpi-components data-source pattern (official `data-sources` sample): `<sdpi-select setting="commandId" datasource="getCommands" hot-reload>`; the action answers in `onSendToPlugin` with `streamDeck.ui.sendToPropertyInspector({ event: "getCommands", items })`, where `items` can be groups with `children`.
- `sdpi-textfield` `pattern` only flags invalid input; it does not block saving.

Feasibility spike (2026-10-07, throwaway plugin, Designer 2 on a two-monitor setup):
- `open -b <bundleId>` from the plugin activates the app in about 70 ms and exits 1 if the app is not installed.
- System Events `keystroke "0" using {command down}` from the plugin reaches Affinity and runs the command (about 110 ms).
- Sending keys about 180 ms after activation failed: the app was not yet receiving keys even though System Events reported it frontmost. Waiting until both System Events and `lsappinfo front` agree, then a 300 ms settle, succeeded.
- First use shows a macOS Automation prompt attributed to **Elgato Stream Deck** ("A Stream Deck plugin wants to use AppleScripts"). While it is on screen, osascript calls block until they time out. Accessibility was already granted to Stream Deck on the test machine; new users may also be asked for it.

## Behavior

1. **Three actions**, UUIDs prefixed with the plugin UUID and never changed after publishing.
2. **Property inspector** (one shared page, `sdpi-components` bundled locally):
   - command dropdown fed by the data source, grouped by category, showing the shortcut in each label (e.g. "Export… ⌥⇧⌘S");
   - optional shortcut override text field (e.g. `cmd+opt+shift+s`) with a `pattern` hint;
   - a short note about the macOS permissions.
   Settings save automatically; no Save button.
3. **On key press:**
   1. Resolve the shortcut: a valid override, otherwise the catalog default. An invalid override is logged and ignored.
   2. If the app is not running: launch it with `open -b`, and stop. No keys are sent.
   3. If it is running but not frontmost: activate with `open -b`, poll `lsappinfo front` until it matches (timeout 2 s), wait 300 ms, re-check, then send. If the check fails, `showAlert` and send nothing.
   4. If it is already frontmost: send immediately.
   5. On failure, `showAlert` and log the cause.
4. **Display:** key title is the command's short label; key image is its category icon. Refreshed on `onWillAppear` and `onDidReceiveSettings`. User-set titles/images still win, per SDK precedence.
5. **Serialization:** presses go through one queue, so keystrokes never interleave.

## Components

| Unit | Purpose | Depends on |
|---|---|---|
| `src/catalog/types.ts` | `Command` type: id, label, short title, category, default shortcut. | none |
| `src/catalog/{shared,designer,photo,publisher}.ts` | Command data from the research files. Shared commands (undo, redo, save, export, zoom, new, close…) reused across apps. About 20 to 30 per app to start. | types |
| `src/shortcut.ts` | Parse/validate/format shortcuts; map to an osascript statement. Printable characters use `keystroke "<char>"` (layout-aware); special keys use `key code`. Pure functions. | none |
| `src/mac/app-control.ts` | `isRunning`, `launch`, `activate`, `isFrontmost` via `open -b` and `lsappinfo`. | none |
| `src/mac/key-sender.ts` | `KeySender` interface plus the osascript implementation, with timeouts and error classification. The only module that sends keystrokes. | shortcut |
| `src/press-queue.ts` | Serializes presses. | none |
| `src/actions/affinity-action.ts` | Shared `SingletonAction` base: settings, display, data source, press flow. Subclasses bind bundle ID and catalog. | catalog, app-control, key-sender, press-queue |
| `src/actions/{designer,photo,publisher}.ts` | Thin subclasses with `@action` UUIDs. | affinity-action |
| `src/plugin.ts` | Register actions, then `streamDeck.connect()` last. | actions |
| `*.sdPlugin/ui/affinity.html` | Shared property inspector. | sdpi-components |
| `*.sdPlugin/imgs/` | Plugin icon, category icon, action icons, one key icon per command category. | none |

Settings per key: `{ commandId?: string; override?: string }`.

## Error handling

| Condition | Detection | Response |
|---|---|---|
| App not installed | `open -b` exits non-zero | `showAlert`, log |
| Permission prompt pending or denied | osascript timeout, error -1743 (Automation), 1002 or -25211 (Accessibility), matched by code and message | `showAlert`, log which permission; property inspector note explains how to grant it |
| App never becomes frontmost | 2 s poll timeout | `showAlert`, log; no keys sent |
| No command chosen, or unknown command id | settings lookup | `showAlert`, log; key title shows "Choose…" |
| Invalid override | parse failure | log warning, use the default |

## Testing

- Unit tests (Vitest): catalog validity (unique ids, parseable shortcuts, no excluded single-key shortcuts, categories present), shortcut parsing and osascript mapping including special keys, override precedence, error classification, press-queue ordering, the press flow's three branches with mocked app control and key sender.
- `streamdeck validate` must pass.
- Manual checklist on the installed apps: every catalog command, plus cold launch, background app, permission prompt and other-monitor cases.
- Development setup: `streamdeck dev` once, then `streamdeck link`.

## Out of scope

Menu-item invocation, single-key tool shortcuts, dials/touch strip, Neo Infobar, Windows, reading document/tool state, global shortcut-map file, one action per command, macro triggering, Marketplace submission assets.

## Open items

- Confirm persona-specific menu shortcuts not captured in the dump (Pixel, Export and Develop personas) before adding them to catalogs.
- Choose the final plugin UUID, plugin name, author name and icon artwork.
