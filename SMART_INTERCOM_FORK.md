# Smart Intercom fork of WiFiManager

Local fork used by the Smart Intercom firmware (`lib/WiFiManager`, picked up by PlatformIO's
library finder instead of the registry package `tzapu/WiFiManager`).

- Upstream: https://github.com/tzapu/WiFiManager (remote `upstream`), fork remote `origin`.
- Base: `v2.0.17-22-g4131fe6` (upstream master, commit `4131fe6`, "Revert Improve SSID selection
  interface (#1851)"). `library.properties` still says 2.0.17.
- License: MIT, unchanged (`LICENSE`, Copyright (c) 2015 tzapu). Authors: tzapu, tablatronix.

## Local changes (all marked `[Smart Intercom fork]` in the sources)

All changes are inert unless `WM_HEADLESS` is defined (the project sets it in
`platformio.ini`), except the two "never log secrets" edits which apply always.

| Change | File | Why |
|---|---|---|
| `WM_HEADLESS`: `setupHTTPServer()` creates no `WebServer` and registers no route | `WiFiManager.cpp` | The firmware has exactly one HTTP server (`src/web/local_web_server.cpp`). None of WiFiManager's unauthenticated pages (`/wifi`, `/wifisave`, `/erase`, `/update` (OTA!), `/exit`, `/restart`, `/param`, `/info`, ...) exists in the binary, so they cannot bypass the admin panel's authentication. Unreferenced handlers and HTML strings are dropped by the linker (~-30 KB flash). |
| `if(server)` guards in `processConfigPortal()` / `shutdownConfigPortal()` | `WiFiManager.cpp` | The server pointer is null in headless builds. The portal (soft AP + captive DNS) works as before. |
| `beginConnectWithoutSaving()`, `saveConnectedCredentials()`, `restoreSavedCredentials()` | `WiFiManager.h/.cpp` | Non-blocking provisioning: the driver is switched to RAM storage for an attempt, the new network is written to NVS only after `WL_CONNECTED`, and a failed attempt restores the previous network. Upstream's `/wifisave` path blocks for the whole connect timeout and saves before the result is known. |
| `getLastDisconnectReason()` / `clearLastDisconnectReason()` + capture in `WiFiEvent()` | `WiFiManager.h/.cpp` | Lets the application distinguish wrong password / network not found without blocking. |
| Passwords never printed in debug output (`Using Password:`, `ap pass`) | `WiFiManager.cpp` | Secrets must not reach logs, at any debug level. |

Not changed on purpose: `autoConnect()` (still blocks up to the connect timeout at boot with
saved credentials, relay OFF meanwhile), `startAP()`'s 500 ms settle delay and
`shutdownConfigPortal()`'s 1 s delay (upstream workarounds, not verifiable without hardware;
the application never calls them during a relay pulse).

## Updating from upstream

1. `git -C lib/WiFiManager fetch upstream`
2. `git -C lib/WiFiManager rebase upstream/master` (or merge) and resolve conflicts in the blocks
   marked `[Smart Intercom fork]`.
3. Re-check that `setupHTTPServer()` still returns before creating a server under
   `WM_HEADLESS`, and that no new `server->` use outside handlers lacks a null check:
   `grep -n "server->" WiFiManager.cpp`.
4. Build and test: `pio run -e esp32dev`, `pio test -e native`; then the manual hardware checks
   in `docs/provisioning.md`.

Note: `lib/WiFiManager` is its own git repository (clone of the fork). The parent repository
does not track it yet; decide whether to commit it as a submodule or as plain files.
