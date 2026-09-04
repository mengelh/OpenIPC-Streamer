# Phase 1 — Architecture Decision

Builds on the Phase 0 findings in [`provenance.md`](provenance.md) and
[`pixelpilot-analysis.md`](pixelpilot-analysis.md). Those findings changed
the shape of this project considerably: the video/MAVLink channel separation
and the UDP sink at ports 5600/14550 already exist in PixelPilot. This
project is therefore not "add a UDP bridge to PixelPilot" but "turn
PixelPilot into a headless service that stops competing with QGroundControl
for the ports it already emits to."

## 1. Fork base

**`OpenIPC/PixelPilot`**, per the recommendation in `provenance.md`. Before
forking, bump the `wfb-ng` submodule from the pinned `0da52790` (2025-08-05)
to current `master` (`3504a38` or later, 2026-08-27+) and rebuild/smoke-test
against `WfbngLink.cpp` — the pinned code path is structurally unchanged
against `HEAD`, so this should be a clean bump. The `devourer` submodule can
stay on PixelPilot's pin (only ~1 month/38 commits behind) unless issue #352
(40 MHz Jaguar1 RX/TX) turns out to affect the target hardware.

## 2. Fork strategy: new build flavor, not a rewrite

PixelPilot currently has **no product flavors** — only a `release` build
type — and `app/build.gradle` depends on all three feature modules
unconditionally:

```gradle
implementation(project(":app:mavlink"))
implementation(project(":app:videonative"))
implementation(project(":app:wfbngrtl8812"))
```

Plan: add a `variant` flavor dimension to `app/build.gradle` with two
flavors, `pixelpilot` (existing behavior, default) and `bridge` (this
project's headless mode):

```gradle
flavorDimensions += "variant"
productFlavors {
    pixelpilot { dimension "variant"; applicationIdSuffix "" }
    bridge     { dimension "variant"; applicationIdSuffix ".bridge" }
}
```

For the `bridge` flavor, `app/videonative` and `app/mavlink` stay in the
Gradle module graph (touching `CMakeLists.txt`'s manually-mirrored devourer
source list, per the analysis doc, is a maintenance trap worth avoiding) —
they're compiled but never asked to bind a socket. Concretely:

- `VideoPlayer::start()` (`app/videonative/src/main/cpp/VideoPlayer.cpp:160-174`)
  is gated behind a flavor-aware flag (`BuildConfig.BRIDGE_MODE` or
  equivalent) in the Java caller, so the internal `UDPReceiver` on port 5600
  never binds.
- `Java_..._MavlinkNative_nativeStart` (`app/mavlink/src/main/cpp/mavlink.cpp:375-383`)
  is gated the same way, so nothing binds port 14550.
- `app/wfbngrtl8812` (`WfbngLink.cpp`, devourer, wfb-ng) is **untouched** —
  this is the one part of the plan's original "fork strategy" goal ("new
  module/flavor instead of rewrite, to keep benefiting from upstream fixes")
  that Phase 0 showed is already satisfied by construction, since this
  module doesn't need any changes at all.
- UI: `bridge` flavor swaps `VideoActivity`'s layout for a minimal
  start/stop/status screen (see Phase 4), instead of maintaining a second
  Activity graph.

This keeps a single repo, a single `app/wfbngrtl8812` implementation shared
between both flavors, and lets `bridge` pick up upstream devourer/wfb-ng
fixes for free on every submodule bump — matching the plan's stated goal.

## 3. Target architecture

```
 [RTL8812AU USB]
        │  (libusb fd handed over by Android USB-Host permission grant)
        ▼
 devourer (native, own thread, WfbngLink.cpp:256 StartRxLoop — unchanged)
        │  raw 802.11 frames, in-process C++ lambda callback
        ▼
 WfbngLink::packetProcessor (WfbngLink.cpp:150-202 — unchanged)
        │  demux by channel_id (radio_port 0x00 / 0x10 / 0x20)
        ├──────────────┬───────────────┬─────────────────┐
        ▼              ▼               ▼                 
 video_aggregator  mavlink_aggregator  udp_aggregator (tunnel, unused here)
 (wfb-ng::AggregatorUDPv4 — decrypt + FEC recover, unchanged, x3)
        │              │
        ▼              ▼
 UDP 127.0.0.1:5600  UDP 127.0.0.1:14550
 (RTP/H264, as-is)   (raw MAVLink stream)
        │              │
        └──────┬───────┘
               ▼
   QGroundControl (separate app, unmodified,
   Video Source = "UDP h.264 Video Stream" :5600,
   MAVLink link = UDP :14550)

 [Bridge Foreground Service]
   wraps WfbNgLink/WfbngLink lifecycle so the above keeps running
   while QGC (not this app) is in the foreground — see §5.
```

Everything above the `packetProcessor` demux is existing, unmodified
`app/wfbngrtl8812` code. The only new *runtime* behavior is: don't start the
two competing consumers (`app/videonative`, `app/mavlink`), and keep the
pipeline alive from a service instead of an Activity.

## 4. Ports / defaults

Unchanged from the plan and already what PixelPilot's `WfbngLink::initAgg()`
(`WfbngLink.cpp:58-84`) emits today:

| Channel | Address | Configurable via |
|---|---|---|
| Video (RTP/H264) | `127.0.0.1:5600` | existing settings-backed `gs.key`/adapter config; port itself is currently a compile-time constant in `WfbngLink.cpp` — Phase 1 follow-up: expose it as a setting if non-default ports are ever needed, not required for the base case |
| MAVLink | `127.0.0.1:14550` | same as above |

No new socket code, no new settings UI required to hit the plan's default
ports — they're already correct out of the box.

## 5. Foreground service (design sketch, implemented in Phase 4)

Per the analysis doc, nothing in the current pipeline runs as a foreground
service — `VideoActivity.onPause()` tears down `wfbLinkManager` and
`videoPlayer` explicitly. Plan for Phase 4:

- New `BridgeService extends Service` (foreground), started from the
  `bridge`-flavor minimal UI on "Start," stopped on "Stop" or task removal
  if the user explicitly stops it (not on `onPause`/backgrounding — that's
  the entire point).
- Moves `WfbLinkManager`'s adapter start/stop calls
  (`app/src/main/java/com/openipc/pixelpilot/WfbLinkManager.java`) out of
  `VideoActivity`'s lifecycle callbacks and into the service's
  `onStartCommand`/`onDestroy`.
- Manifest already declares `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_DATA_SYNC`,
  `WAKE_LOCK`, and `POST_NOTIFICATIONS` — reusable as-is; need to confirm the
  correct `android:foregroundServiceType` for the service declaration
  (`connectedDevice` fits the USB-adapter angle best under current Android
  foreground-service-type rules; needs a compile-SDK-version check at
  implementation time) and add the `<service>` entry + persistent
  notification.
- USB permission flow (`WfbLinkManager.getAttachedAdapters()`/
  `refreshAdapters()`) must still be triggered once with the app in the
  foreground on first run — Android does not allow granting USB-host
  permission to a purely backgrounded process. After the initial grant,
  `USB_DEVICE_ATTACHED` cold-start (already declared in the manifest) should
  let the service re-acquire the device without reopening the UI.
- Request battery-optimization exemption
  (`ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`) from the minimal UI, since
  aggressive OEM battery management is called out as an open risk in the
  project plan.

This is a genuinely new component — unlike the UDP sink, there's no existing
PixelPilot code to repurpose here beyond the manifest permissions already in
place.

## 6. What Phase 1 deliberately leaves open

- Exact `foregroundServiceType` value and minimum-SDK implications —
  resolve when writing the service in Phase 4.
- Whether to keep `app/mavlink`'s OSD parsing available as an optional
  in-bridge status display (nice-to-have, not required for QGC to work) or
  drop it entirely — deferred to Phase 4 UI design.
- Whether the `udp_aggregator` "tunnel" channel (port 8000,
  `WfbNgVpnService`) is worth keeping in the `bridge` flavor — out of scope
  for this project (QGC doesn't use it); default to leaving it compiled but
  inactive, consistent with the "touch nothing in `wfbngrtl8812`" principle.

## 7. Phase 2 scope, updated per Phase 0 findings

Per the original plan, Phase 2 ("remove the rendering part") was expected to
involve real surgery on the video pipeline. Given the findings above, its
scope is now:

1. Bump the `wfb-ng` submodule (see §1).
2. Add the `bridge` product flavor (see §2).
3. Gate `VideoPlayer::start()` and `MavlinkNative.nativeStart()` behind the
   flavor flag.
4. Confirm the app still builds and that, with the RTL8812AU dongle
   attached, `127.0.0.1:5600` and `127.0.0.1:14550` receive traffic while
   nothing in-app is bound to them (verifiable with `netstat`/a UDP test
   client before QGC is involved, per the plan's Phase 3 test-client step).

No RTP payloading, no new `DatagramSocket`, no custom demuxer — all removed
from Phase 2/3 scope per the Phase 0 analysis.

## 8. Phase 2 status: done

Implemented on the `bridge-mode` branch of the fork,
[github.com/mengelh/PixelPilot/tree/bridge-mode](https://github.com/mengelh/PixelPilot/tree/bridge-mode)
(local checkout: `../PixelPilot/`, sibling to this repo):

1. `wfb-ng` submodule bumped `0da5279` → `3504a38` (current upstream
   `master`) — commit `aa232fc`.
2. `variant` flavor dimension (`pixelpilot` / `bridge`) added to
   `app/build.gradle`, exposing `BuildConfig.BRIDGE_MODE` — commit `2065620`.
3. `VideoActivity.setupMavlink()` and the `videoPlayer.start()` /
   `startAudio()` / `updateUdpForwardingState()` block in `onResume()` are
   now gated on `!BuildConfig.BRIDGE_MODE` — same commit. `app/wfbngrtl8812`
   untouched. The corresponding `stop()`/`stopAudio()`/`nativeStop()` calls
   were left unguarded after confirming both are no-ops when the matching
   start was skipped (`VideoPlayer.stop()` checks its `timer` for null;
   `MavlinkNative.nativeStop()` only increments a signal counter).
4. Build verified locally: set up a standalone Android SDK
   (cmdline-tools 15859902, platform 34, build-tools 35.0.0, NDK r26b/
   26.1.10909125 as pinned by `app/build.gradle`) and ran
   `./gradlew assembleBridgeDebug assemblePixelpilotDebug`. **Both flavors
   build successfully** — `app-bridge-debug.apk` and
   `app-pixelpilot-debug.apk`, ~16.76 MB each (same size since `videonative`/
   `mavlink` are still compiled into both flavors, per §2 — only gated at
   runtime). All compiler warnings are pre-existing upstream code
   (mediapipe, devourer), none introduced by this change.

Not yet done: the actual QGroundControl end-to-end test from Phase 5.

## 9. On-device testing found and fixed three more bugs

Tested via wireless adb on a Galaxy Tab S7 FE (Android 13) with a real
RTL8812AU dongle (`0BDA:8812`). All fixed on `bridge-mode`:

1. **Launch crash — missing native libs.** `libusb1.0.so`/`libsodium.so`
   (linked into `libWfbngRtl8812.so`) and `libopus.so` (linked into
   `libVideoNative.so`) are referenced by absolute path in each module's
   CMakeLists.txt but were never packaged into the APK — no `jniLibs` source
   set picked them up. Pre-existing PixelPilot gap (confirmed on the plain
   `pixelpilot` flavor too), only surfaced by a fresh install with nothing to
   mask it. Fixed by copying the required prebuilts into
   `src/main/jniLibs/<abi>/` for both modules (commit `5334188`).
2. **Launch crash — hardcoded package path.** `WfbngLink.hpp` hardcoded the
   `gs.key` path as `/data/user/0/com.openipc.pixelpilot/files/gs.key`. Works
   for the `pixelpilot` flavor, but `bridge`'s `applicationIdSuffix` installs
   it under `com.openipc.pixelpilot.bridge`, so the hardcoded path pointed at
   a data directory that doesn't exist for this app -> `fopen()` failure ->
   uncaught C++ exception -> abort. Fixed by resolving the path from
   `context.getFilesDir()` via JNI at construction time instead (same
   commit) -- this also fixes the underlying fragility for any future
   rename/fork, not just our flavor.
3. **Background-crash race, found via repeated pause/resume cycling.**
   `onPause()`/`onStop()` call `wfbLinkManager.stopAdapters()`,
   `onResume()` immediately calls `startAdapters()` again. Cycling this fast
   hit a native teardown race in devourer's `RtlJaguarDevice` destructor
   (`rtw_hal_deinit()` touching HAL state on a device that hadn't finished
   bringing up yet) -> SIGSEGV/SIGABRT. Reproduced 2/3 on `bridge`, 0/6 on
   plain `pixelpilot` (bridge's faster `onResume()`, from skipping the video
   player, shifts the timing into the race window). Checked upstream
   `OpenIPC/devourer`'s issue history (#281/#344/#350/#351 already hardened
   this exact teardown path for other cases) -- this looks like a genuinely
   new edge case, not a known/patched one, so patching a third-party driver
   we don't own and can't reliably re-verify wasn't the right call. Fixed at
   the integration layer instead: bridge mode now skips
   `stopAdapters()` on pause/stop entirely, since not tearing the pipeline
   down while backgrounded is the actual point of this project. Verified
   with 5 rapid background/foreground cycles on-device (commit `cbb07c2`).
   Still only survives ordinary lifecycle transitions, not memory-pressure
   eviction -- Phase 4's Foreground Service remains necessary for
   guaranteed long-running background survival.

Updated test APK: `test_APK/app-bridge-debug.apk` on `bridge-mode`.
