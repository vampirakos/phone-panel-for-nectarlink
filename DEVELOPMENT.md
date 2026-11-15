# Phone panel for NectarLink (Windhawk mod)

An **unofficial** Windhawk mod by vampirakos that shows a NectarLink-paired Android phone
in the Windows 11 Start menu, in the side panel Phone Link normally uses.

- Mod file: `phone-panel-for-nectarlink.wh.cpp` (in a `windhawk-mods` fork, it goes in `mods/`)
- Mod id: `phone-panel-for-nectarlink`, current version 1.0.0
- NectarLink (the app it works with, not ours): https://github.com/thtbee/nectarlink, by Bee
- Publishing: pull request to `ramensoftware/windhawk-mods`; maintainers review it

## How it works

- Injected into `StartMenuExperienceHost.exe` (x64 only).
- A worker thread finds Start's `Windows.UI.Core.CoreWindow` and runs code on its XAML
  thread with `RunFromWindowThread` (a `WH_CALLWNDPROC` hook plus `SendMessage`).
- The UI is XAML strings loaded with `XamlReader::Load`. Rows in Recent and Clipboard are
  built per item, and their text is set in code, never interpolated into XAML.
- A `DispatcherTimer` (500 ms) on the XAML thread attaches the panel, applies status and
  shows short "flash" messages.
- **Companion mode (default):** it finds `RightCompanion` (`StartMenu.StartMenuCompanion`,
  or `LeftCompanion`), adds our root to its `CompanionRoot` Grid (RowSpan and ColumnSpan 99,
  margin 1), and collapses Phone Link's `MainContent`. It re-collapses it if Phone Link shows
  it again, and restores it on uninit.
  - Needs Windows Settings → Personalization → Start → "Show mobile device in Start" on.
  - Replacing Phone Link's panel only happens while NectarLink is installed and not known to
    be unpaired (`g_takeOverAllowed`), unless the setting `takeOver` = `always`.
- **Floating mode:** an unparented `Popup` placed next to the `MainMenu` element.
- If it can't find an element, it logs the named-element tree of Start once (`DumpNames`).
  Use that log to fix element names after Windows updates.

### Start's XAML tree (new Start menu, Nov 2026, from a real log)

`FrameRoot > AnimationRoot > { MainMenu (834x866), RightCompanionContainerGrid >
RightCompanion > CompanionRoot > { AcrylicBorder, MainContent > { PrimaryCardContainer,
SecondaryCardContainer, ActionsBar } } }`. `ShowHideCompanion` is the phone toggle next to
search. `StartSizingFrame` does **not** exist in this build.

## NectarLink data sources

Local API: loopback, opt-in. The user turns it on in NectarLink → Settings → Local
automation API. Token and port are in `%LOCALAPPDATA%\Nectarlink\api-token.json`
(`{port, token}`, default port 53320). Every request needs `Authorization: Bearer <token>`.
Requests with an `Origin` header are rejected. Docs: NectarLink `docs/api.md`.

- `GET /v1/devices`: name, online, link (`lan` | `usb` | `relay` | `connecting` |
  `offline`), battery `{level, charging}`
- `GET /v1/timeline?limit=N`: items `{kind, direction, title, detail, target, timestamp}`.
  `target` = saved path(s), newline-separated, or a link
- `GET /v1/clipboard`: `history[]` `{kind text|image, preview (full text), deviceName,
  incoming, timestamp}`
- `POST /v1/camera {"mode":"scan"}`, `POST /v1/play-on-phone {}`. Errors include
  `nothing_playing` and `no_phone` / 409.

Actions that work without the API run `nectarlink-desktop.exe --task <action>` (the Jump
List tasks): `find-phone`, `send-clipboard`, `take-photo`, `send-files`, `open-timeline`,
`open-settings`. With no args it just shows the window. The exe is found through
`HKLM\Software\Nectarlink\InstallDir`, then `%ProgramFiles%\Nectarlink`, or the `exePath`
setting.

Not available from NectarLink yet (would need changes in NectarLink): messages,
notifications, calls, phone toggles (Wi-Fi, Bluetooth, DND). The phone's toggles and its
wallpaper-derived accent color reach the PC (`AppState.toggles`, Material You seed) but
aren't in the API. The wallpaper image isn't sent at all; on Android 13+ reading it needs
"All files access".

## Decisions to keep

- **Safety:** Recent files open directly only for an allowlist of media and document
  extensions (`IsSafeToOpen`). Everything else opens with `explorer /select`. Only
  http(s) links open. Don't loosen this.
- **Trademark:** NectarLink's policy allows "compatible with" / "for NectarLink" but not
  anything that looks official. Keep "Unofficial" and "not affiliated" in the description and
  readme. Don't use the NectarLink logo.
- **Network:** only `127.0.0.1`. Nothing leaves the PC.
- **Look:** monochrome Segoe Fluent Icons. The battery glyph steps in 10%
  (`E850`–`E859`, `E83F`; charging `E85A`–`E862`, `E83E`, `EA93`) at FontSize 22, with
  `TextLineBounds="Tight"` on the status text so it centers with the icon. Theme colors use
  system ThemeResources (accent, base low).
- The Bluetooth status icon was dropped on purpose (the user asked for it to be left out
  for now).

## Gotchas met so far

- `#undef GetObject` (the windows.h macro clashes with `IJsonValue::GetObject`). Use
  `v.GetObject()`, not `as<JsonObject>()`, which fails with E_NOINTERFACE.
- Generic lambdas (`auto&` params) break `x.try_as<T>()` with a "dependent template" error.
  Use explicit parameter types.
- The mod compiles inside Windhawk (clang + C++/WinRT headers). Test by enabling the mod and
  reading Windhawk's log or DebugView output.

## To do / ideas

- Check the light theme and high contrast. Only dark mode has been seen.
- Translations (Greek first). All strings are English now.
- Optional phone wallpaper: a user-picked local image shown in a phone frame (mod only), or
  the phone's accent color (needs a small NectarLink API patch).
- Retest after Windows updates. Start's element names can change.
- ARM64 build once NectarLink ships ARM64.
