# Phase 0.2/0.3 — PixelPilot Code Analysis

Analyzed state: `OpenIPC/PixelPilot` commit `3d6ca17` (master, 2026-09-02),
cloned into `scratchpad/repos/pixelpilot/`. Submodule pins are documented in
`docs/provenance.md`; since checking out the submodules failed in the sandbox
due to a Windows path-length issue (`fatal: '$GIT_DIR' too big`), `devourer`
and `wfb-ng` were instead checked as standalone clones and compared against
the exact PixelPilot pin via `git diff <pin>..HEAD` (see the citations below,
which come from the pinned — and therefore unchanged — source).

## 0.1 Submodule inventory

Per `.gitmodules`, PixelPilot has **exactly two** git submodules (no others):

| Path | URL | Branch | Purpose |
|---|---|---|---|
| `app/wfbngrtl8812/src/main/cpp/devourer` | `https://github.com/openipc/devourer.git` | `master` | Userspace USB/monitor-mode driver for Realtek RTL8812AU/8811AU/8821AU/8812EU/8822EU chips (libusb-based, no kernel module). |
| `app/wfbngrtl8812/src/main/cpp/wfb-ng` | `https://github.com/svpcom/wfb-ng.git` | `master` | FEC (Reed-Solomon/zfex), decryption (libsodium), frame reassembly, and the `AggregatorUDPv4`/`Forwarder` classes that implement the reference `wifibroadcast.cfg` logic. |

`LiveVideo10ms` is **no longer** a submodule — the decoder/renderer code is
vendored directly into the repo as a standalone Gradle module `app/videonative`
(no entry in `.gitmodules`). PixelPilot has decoupled from upstream here.

### Other (non-submodule) Gradle modules, for architectural context

| Module | Purpose |
|---|---|
| `app/src` | Main app: `VideoActivity`, USB/lifecycle orchestration (`WfbLinkManager`), OSD, settings, DVR UI. |
| `app/wfbngrtl8812` | JNI wrapper (`WfbNgLink.java`) + native `WfbngLink.cpp/.hpp` that wire devourer and wfb-ng together. **The central building block for this project.** |
| `app/videonative` | Vendored LiveVideo10ms fork: `VideoPlayer`/`VideoDecoder` (NALU parsing, MediaCodec decoding), its own `UDPReceiver`/`UDSReceiver`. Removed in Phase 2. |
| `app/mavlink` | Standalone native module (`mavlink.cpp`) that parses MAVLink UDP packets for the OSD display (altitude, battery, GPS, etc.). Not part of wfb-ng/devourer. |

## Guiding questions

### 1. USB/driver layer (devourer)

**USB host permission flow (standard Android pattern):**
`app/src/main/java/com/openipc/pixelpilot/WfbLinkManager.java:92-132`
(`getAttachedAdapters()`) filters attached USB devices against
`app/src/main/res/xml/usb_device_filter.xml` (vendor/product ID list).
`WfbLinkManager.java:141-158` (`refreshAdapters()`) checks
`usbManager.hasPermission(...)` and requests missing permissions via
`usbManager.requestPermission(dev, pendingIntent)` (`ACTION_USB_PERMISSION`
broadcast, `PendingIntent.FLAG_IMMUTABLE` plus an explicit
`setPackage(...)`, since Android 14 otherwise won't deliver the
PendingIntent to a runtime receiver — see the comment at lines 148-150). The
`AndroidManifest.xml` declares the `android.hardware.usb.host` feature, the
`USB_PERMISSION` permission, and a `USB_DEVICE_ATTACHED` intent filter on
`VideoActivity` referencing the same `usb_device_filter.xml` (cold-start
trigger on plug-in).

**Opening the device handle:** `WfbNgLink.java:116-124` (`start()`) calls
`usbManager.openDevice(usbDevice)` and passes the raw
`UsbDeviceConnection.getFileDescriptor()` (an `int fd`) to the native layer.
On the native side, `libusb_wrap_sys_device()`
(`app/wfbngrtl8812/src/main/cpp/WfbngLink.cpp:100-111`) takes over this
already-opened fd — libusb doesn't open anything itself, it just "wraps" the
access already granted app-side. This is the standard way libusb works on
Android without root.

**Thread/process model:** devourer runs **in the same app process**, but on
its **own dedicated Java/native thread per adapter**: `WfbNgLink.java:130`
starts `new Thread(() -> nativeRun(...))`; `nativeRun` calls
`WfbngLink::run()` (`WfbngLink.cpp:86-290`), whose final step
`current_device->StartRxLoop(packetProcessor)` (line 256) **blocks** until
`StopRxLoop()` is called externally — the comment at lines 250-251
("Blocking RX loop on this thread; devourer pumps the libusb events itself")
confirms this explicitly. There's also a second thread (`usb_tx_thread`,
lines 219-242) for the TX path (telemetry/control to the drone), and
optionally a third thread for the "adaptive link" quality control loop
(`start_link_quality_thread`, line 431+). No separate process, no
Android service isolation — everything runs in-process.

**Interface to the layer above:** No JNI ring buffer. Devourer delivers raw
802.11 frames via a **C++ lambda callback** directly on the calling thread:
`packetProcessor` (`WfbngLink.cpp:149-202`) is passed to `StartRxLoop()` as a
`std::function<void(const Packet&)>` and called synchronously for every
frame received — an in-process callback, no queue, no JNI boundary at this
point (the only JNI boundary is between Java (`WfbNgLink`) and C++
(`WfbngLink`), not between devourer and wfb-ng processing, which are both
plain C++).

### 2. wfb-ng integration

**Wiring:** `WfbngLink::initAgg()` (`WfbngLink.cpp:58-84`) instantiates
**three separate** `wfb_ng::AggregatorUDPv4` objects (the class comes
straight from the wfb-ng submodule, `wfb-ng/src/rx.hpp:367-382`, used
unmodified):

```
video_aggregator   → 127.0.0.1:5600   (radio_port 0x00)
mavlink_aggregator → 127.0.0.1:14550  (radio_port 0x10)
udp_aggregator     → 127.0.0.1:8000   (radio_port 0x20, "tunnel" channel for WfbNgVpnService)
```

Each aggregator gets its own `channel_id` (`link_id << 8 | radio_port`) and
the same `gs.key` path. The `packetProcessor` callback
(`WfbngLink.cpp:150-202`) demultiplexes incoming 802.11 frames by this
`channel_id` (`frame.MatchesChannelID(...)`, lines 161/179/190) and hands the
payload to the matching aggregator — **this is the demultiplexing** that
follows the `[gs_video]`/`[gs_mavlink]` radio-port scheme of the Linux
reference config. Decryption (`crypto_aead_chacha20poly1305`, session key via
`crypto_box`) and FEC recovery (Reed-Solomon via `zfex`) happen **inside**
`Aggregator::process_packet` in `wfb-ng/src/rx.cpp` (base class, unmodified),
not in PixelPilot's own code.

**Most important question — channel separation:** **Yes, it already exists,
and structurally it's nearly identical to the Linux reference model.** Each
aggregator's `send_packet()` method (`wfb-ng/src/rx.cpp:1009-1043`, base
class) ends by calling `send_to_socket()`, which for `AggregatorUDPv4`
(`wfb-ng/src/rx.cpp:1115-1118`) is a plain
`sendto(sockfd, payload, packet_size, MSG_DONTWAIT, &saddr, ...)` to the
`127.0.0.1` address/port named above — **regardless of whether anything is
listening.** The three channels are therefore already exposed as three
independent, loosely coupled UDP targets on the loopback interface, at
exactly the default ports 5600/14550 required by the project plan.

**Video channel format:** **Already RTP-encapsulated — no custom RTP
payloading needed.** `Aggregator::send_packet()` forwards the raw payload of
the original UDP datagram 1:1 (no parsing/reframing in wfb-ng). Since the
drone side (majestic/wfb_tx in the OpenIPC ecosystem) already hands complete
RTP/H264 packets to wfb_tx via UDP, and wfb-ng passes every UDP datagram
through the radio link as an atomic, opaque payload, the ground segment
receives exactly the same RTP packet the camera produced. Confirmed by
`app/videonative/src/main/cpp/VideoPlayer.cpp:164-174`: the internal
`UDPReceiver` on port 5600 passes received data straight to
`onNewRTPData(...)` (not to a raw-NALU parser) — the name alone already
confirms the expected format.

**MAVLink channel — raw data vs. internal-only use:** **Both, depending on
which layer you look at.** At the UDP layer, the MAVLink channel is already a
clean, complete byte stream (`mavlink_aggregator` sends unmodified to
`127.0.0.1:14550`, exactly like the video channel). Inside the app, though,
this stream is **also** consumed by a second, independent consumer:
`app/mavlink/src/main/cpp/mavlink.cpp:377-383`
(`Java_..._MavlinkNative_nativeStart`) starts its own thread that **binds the
same port 14550 itself** (`listen(14550)`) and parses the packets into
telemetry fields (altitude, pitch/roll/yaw, battery, GPS, …) for the OSD
display (`MavlinkNative.java`, `onNewMavlinkData` callback) — this path is
purely internal/parsed, not a passthrough. For this project that means: the
raw byte stream already exists (produced by wfb-ng), it's just currently
also being read by a second in-app consumer, which needs to be disabled in
the fork (see recommendation below).

### 3. Decoder/renderer boundary (LiveVideo10ms/videonative)

The "tap point" where the app currently hands data into the video pipeline
is **not** after the decoder — it's earlier, at the UDP layer:
`VideoPlayer::start()` (`app/videonative/src/main/cpp/VideoPlayer.cpp:160-174`)
opens its own `UDPReceiver` on `127.0.0.1:5600` (`VS_PORT`) and passes every
received packet to `onNewRTPData(...)`, which internally kicks off the
RTP/NALU parser and then `VideoDecoder`/MediaCodec.

**There's already a clearly delimited, swappable sink — actually two:**

1. Cleanest option: **simply don't let `VideoPlayer::start()` bind the
   `UDPReceiver` at all** (remove/disable the entire `app/videonative`
   module per Phase 2). Since `WfbngLink.cpp` already sends to the same port
   independently of this, that alone is enough to let an external process
   (QGroundControl) bind the port instead — no new code needed.
2. Alternatively, if in-app forwarding is desired: `UDPReceiver`
   (`app/videonative/src/main/cpp/UdpReceiver.h/.cpp`) already has a ready
   `setForwarding(ip, port, enabled)` method (`UdpReceiver.cpp:176-190`) that
   forwards every received packet via `sendto()` to a target **before**
   local processing (`onDataReceivedCallback`) (`UdpReceiver.cpp:134-141`,
   comment "Forward packet first (minimize latency)"). This is already
   user-configurable via a settings menu ("UDP Forwarding",
   `VideoActivity.java:1057-1137`, SharedPreferences keys
   `forward_udp_enabled`/`forward_udp_ip`/`forward_udp_port`, default port
   already **5600**). For QGC on the same device the target would be
   `127.0.0.1:5600` — but that then collides with the app's own receive
   port, so option 1 (disable the consumer) is the cleaner choice.

### 4. Key handling (`gs.key`)

- **File path (native, hardcoded):**
  `/data/user/0/com.openipc.pixelpilot/files/gs.key`
  (`WfbngLink.cpp:125`, `const char *keyPath`). Used by all three
  `AggregatorUDPv4` instances as well as the TX path (`TxArgs::keypair`,
  `WfbngLink.cpp:224`).
- **Java-side management:** `VideoActivity.java:1481-1497` (`getGsKey()`/
  `setGsKey()`) stores the key **base64-encoded in SharedPreferences**
  (`"general"` prefs, key `"gs.key"`), not directly as a file — the file at
  `keyPath` is written from prefs on startup (`VideoActivity.java:1747-1751`).
- **Default value:** `VideoActivity.java:1466-1479` (`setDefaultGsKey()`)
  loads the default key shipped in `app/src/main/assets/gs.key` (an APK
  asset) on first launch — this is the publicly known OpenIPC/wfb-ng default
  key, not an individual secret.
- **User selection:** the "gs.key" settings item
  (`VideoActivity.java:791-795`) opens a Storage Access Framework dialog to
  import a custom `gs.key` file from the device; the result again lands in
  SharedPreferences. Changing it triggers `WfbNgLink.refreshKey()` →
  `nativeRefreshKey()` → `WfbngLink::initAgg()` (re-instantiating all three
  aggregators with the new key path).

### 5. App lifecycle

**None of video/USB/wfb-ng/MAVLink runs as a foreground service.** The
entire pipeline is tied to the visible `VideoActivity`:
`VideoActivity.onPause()` (`VideoActivity.java:1538-1555`) explicitly calls
`videoPlayer.stop()`, `wfbLinkManager.stopAdapters()` (→ stops the USB RX
loop in `WfbngLink`), and even stops the VPN tunnel service — so the
pipeline dies as soon as the app goes to the background (e.g. when
QGroundControl is brought to the foreground). The only actual Android
`<service>` in the manifest is `WfbNgVpnService`
(`AndroidManifest.xml`, `android.net.VpnService`) — an optional extra
feature for the tunnel channel (port 8000), **not** a foreground service and
not part of the video/MAVLink path. For Phase 4 of this project (guaranteed
background operation alongside QGC), an entirely new foreground service
needs to be built around `WfbngLink`/`WfbNgLink` — nothing reusable exists
here beyond the `WAKE_LOCK`/`FOREGROUND_SERVICE` permission pattern already
declared in the manifest (`AndroidManifest.xml`, presumably intended for the
VPN service).

### 6. Build setup (NDK/CMake)

Classic multi-module Gradle/CMake setup, one native module per Gradle
submodule:

- `app/wfbngrtl8812/src/main/cpp/CMakeLists.txt` (`project("WfbngRtl8812")`,
  `cmake_minimum_required(VERSION 3.22.1)`) builds **three** targets:
  1. `wfb-ng` (STATIC) — compiles the wfb-ng submodule sources (`zfex.c`,
     `radiotap.c`, `rx.cpp`, `wifibroadcast.cpp`) directly from the
     submodule path, with its own `ZFEX_*` SIMD compile definitions.
  2. `devourer` (STATIC) — does **not** compile devourer via its own CMake
     (`add_subdirectory` isn't possible since devourer internally uses
     `pkg-config` for libusb, which is missing in the NDK toolchain
     context); instead it manually mirrors devourer's source file list in
     PixelPilot's own `CMakeLists.txt` (lines 35-93) — grouped by chip
     generation (Jaguar1 = RTL8812AU family, Jaguar3 = RTL8812EU/8822EU
     family), controlled via `DEVOURER_HAVE_JAGUAR1`/`DEVOURER_HAVE_JAGUAR3`
     compile flags. **This is a maintenance trap for a fork:** new devourer
     source files (e.g. for RTL8733B, currently under active development per
     `provenance.md`) have to be manually added to this list, or the build
     breaks on the next devourer submodule bump.
  3. `WfbngRtl8812` (SHARED, `${CMAKE_PROJECT_NAME}`) — PixelPilot's own code
     (`WfbngLink.cpp`, `RxFrame.cpp`, `TxFrame.cpp`,
     `SignalQualityCalculator.cpp`), linked against the two targets above
     plus precompiled prebuilts (`libs/${ANDROID_ABI}/libusb1.0.so`,
     `libsodium.so`, `libpcap.a`, from
     `app/wfbngrtl8812/src/main/cpp/libs/`). Loaded from Java via
     `System.loadLibrary("WfbngRtl8812")` (`WfbNgLink.java:30`). C++20
     (`CXX_STANDARD 20`).
- `app/videonative` and `app/mavlink` each have their own independent
  `CMakeLists.txt`/Gradle module with their own `System.loadLibrary(...)`
  call — cleanly decoupled from the wfb-ng/devourer build, so they can be
  removed from the build by disabling the respective Gradle module/build
  flavor without touching `app/wfbngrtl8812`.

**Assessment for the fork:** the native build is cleanly modularized;
removing `app/videonative` and `app/mavlink` (Phase 2) should be possible
without affecting `app/wfbngrtl8812`. The one notable friction point is the
manual mirroring of devourer's source file list in PixelPilot's CMake, which
needs upkeep on future devourer updates.

**Note on the module graph itself (relevant for Phase 1):** there are
currently **no Gradle product flavors** in `app/build.gradle` — only a
`release` build type. `settings.gradle` declares four modules
(`:app`, `:app:videonative`, `:app:wfbngrtl8812`, `:app:mavlink`), and
`app/build.gradle` depends on all three submodules unconditionally
(`implementation(project(":app:mavlink"))` etc., no flavor gating). A
"bridge" build variant therefore has no existing flavor infrastructure to
hook into — it has to be added.

## Recommendation: exactly where to hook in the "UDP sink"

**Central result of this analysis:** a new UDP sink in the sense of Phase 3
of the project plan (a `DatagramSocket` that sends video bytes to
`127.0.0.1:5600`) **does not need to be built** — it already exists, fully
and usably unmodified, in `wfb-ng::AggregatorUDPv4`, and PixelPilot's own
`WfbngLink::initAgg()`
(`app/wfbngrtl8812/src/main/cpp/WfbngLink.cpp:58-84`) already instantiates
it for exactly the ports the project needs:

- Video → `127.0.0.1:5600` (already RTP/H264, no custom payloading needed)
- MAVLink → `127.0.0.1:14550` (already a raw MAVLink byte stream)

The actual fork work consists of **removing the two competing internal
consumers of these ports** so QGroundControl (as a separate process on the
same device) can bind them instead:

1. **Remove the video consumer:**
   `app/videonative/src/main/cpp/VideoPlayer.cpp`, method
   `VideoPlayer::start()` (lines 160-174) — stop starting the `UDPReceiver`
   on `VS_PORT = 5600` (in practice: remove the entire `app/videonative`
   module from the build/flavor per Phase 2).
2. **Remove the MAVLink consumer:**
   `app/mavlink/src/main/cpp/mavlink.cpp`, function
   `Java_com_openipc_mavlink_MavlinkNative_nativeStart` (lines 375-383) —
   stop the `listen(14550)` call there (or drop the `app/mavlink` module for
   OSD purposes entirely, if no in-app telemetry display is needed anymore).
3. **Leave unchanged:** all of `app/wfbngrtl8812` (devourer, wfb-ng,
   `WfbngLink.cpp/.hpp`, `RxFrame`, `TxFrame`, `SignalQualityCalculator`) —
   this already satisfies the project plan's own stated principle ("fork
   strategy: new module/flavor instead of a rewrite, to keep benefiting from
   upstream fixes to devourer/wfb-ng") by construction, since this part of
   the module doesn't need to be touched at all.
4. **Inspect, don't change:**
   `app/src/main/java/com/openipc/pixelpilot/WfbLinkManager.java` and
   `VideoActivity.java` orchestrate adapter start/stop and are the right
   place to build the new foreground service around
   `WfbNgLink`/`WfbngLink` in Phase 1/4 (see question 5) — real new code is
   needed here, but as an addition, not a replacement of existing logic.

**Assessment of channel separation:** already present, doesn't need to be
built from scratch. The video/MAVLink/tunnel separation already exists
structurally one layer deeper than assumed — not as PixelPilot's own
application logic, but as a direct, unmodified use of wfb-ng's own
`AggregatorUDPv4`/`Aggregator` mechanism, which matches exactly the
radio-port scheme of the Linux reference config
(`[gs_video]`/`[gs_mavlink]`). The risk noted in the project plan ("channel
separation may not be implemented as cleanly [as the Linux reference] …
might need to be rebuilt") turns out, per this analysis, **not to apply** —
if anything, the separation is cleaner and more complete than the plan
originally assumed, which substantially reduces the scope of Phase 2/3.
