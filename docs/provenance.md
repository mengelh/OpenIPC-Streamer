# Phase 0.0 — Provenance Check

As of: 2026-09-04. All repos were cloned into `scratchpad/repos/` (read-only,
not part of this repo) and checked for activity via the unauthenticated
GitHub API. Timestamps are commit timestamps from the respective repos.

## Overview

| Repo | Last commit | Activity (last ~3 months) | Open issues | License |
|---|---|---|---|---|
| `OpenIPC/PixelPilot` | 2026-09-02 | Very active: 2 authors (iflyhere, RomanLut/vertexodessa as reviewer), 5 PRs merged in the last 2 weeks | 31 open (1 of which is a PR) | GPL-3.0 |
| `OpenIPC/devourer` | 2026-09-01 | Very active: 3 authors (josephnef, snokvist, gilankpam), daily/multi-weekly commits | 23 open | GPL-2.0 |
| `svpcom/wfb-ng` | 2026-08-27 | Active, but a one-person project: almost all commits by svpcom himself, one external contribution (InanixFR, PR #457) | 23 open | GPL-3.0 |
| `Consti10/LiveVideo10ms` | 2023-10-30 | **Inactive** (no push in nearly 3 years) | 6 open | LGPL-3.0 |
| `gehee/FPVue_android` | 2024-08-16 | **Explicitly declared unmaintained by the author** (see below) | 14 open | GPL-3.0 |
| `OpenIPC/fpv4win` | 2024-12-11 | Dormant since Dec. 2024 (only a few scattered commits in late 2024) | 2 open | GPL-3.0 |

## Detail per building block

### 1. `OpenIPC/devourer` — USB/monitor-mode driver

Very actively maintained, daily commits from multiple authors (josephnef,
snokvist, gilankpam). Recent focus: RTL8733B chip support, USB TX
aggregation (~40% CPU savings), TX power calibration, BlockAck hardening.
Open issues are mostly feature/hardware bring-up tickets; two relate to
stability:

- **#352** — 40 MHz bandwidth on Jaguar1 (RTL8812AU family) broken in both
  directions (TX not decodable, RX deaf). Relevant since RTL8812AU is the
  chip used in this project — worth checking before production use whether
  PixelPilot's pin already includes/fixes this.
- **#376** — hardware failure on a test board, not a general stability issue.

**PixelPilot's pin vs. upstream:** PixelPilot currently points at commit
`bb03774c` (2026-08-02). Upstream `master` is at `f86f92d` (2026-09-01) —
**38 commits / roughly 1 month behind.** The lag is moderate and PixelPilot
visibly keeps catching up (most recently PR #101 "updated to latest devourer
with rtl8821au support").

### 2. `svpcom/wfb-ng` — FEC/encryption/protocol

**PixelPilot actually vendors the canonical `svpcom/wfb-ng` repo as a
submodule** (`.gitmodules`: `url = https://github.com/svpcom/wfb-ng.git`,
`branch = master`) — not a fork of its own. This is the reference
implementation that PX4 docs, OpenHD, and QOpenHD also point to.

**PixelPilot's pin vs. upstream:** PixelPilot points at commit `0da52790`
(2025-08-05). Upstream `master` is at `3504a38` (2026-08-27) — **56 commits
and a good year behind.** This is the most notable finding of this check.
The missed commits include, among others:

- Packet deduplication across multiple receive adapters (`Forwarder`/`Aggregator`)
- Robustness fixes in the data plane and statistics integrity
- Fixes for `systemd` service-start races with NetworkManager
- Various compatibility/build fixes (Ubuntu 26.04, GH Actions)

For the relevant core path — `Aggregator::send_packet` /
`AggregatorUDPv4::send_to_socket` (where video/MAVLink are emitted as plain
UDP to `127.0.0.1`, see `pixelpilot-analysis.md`) — a diff between the
pinned commit and `HEAD` shows **no structural changes**; class and method
names are stable. The lag mainly affects robustness/dedup features, not the
Aggregator architecture that is central to this project. Still: a
year-old pin on a security-relevant module (decryption, `crypto_box`/
`crypto_aead`) is a maintenance risk that should be fixed when forking (bump
the submodule to current `master` beforehand and test against the app).

### 3. `Consti10/LiveVideo10ms` — decoder/renderer

Checked only briefly, as planned: last push 2023-10-30, almost three years
without activity, 6 open issues with no response. Not further relevant since
this part is removed in Phase 2 anyway. For context: PixelPilot no longer
includes LiveVideo10ms as a git submodule — the code has been vendored and
further developed as a standalone module `app/videonative` directly in the
repo (no `.gitmodules` entry for it), i.e. PixelPilot has already decoupled
from upstream and maintains this part itself.

### 4. `gehee/FPVue_android` — the original "glue"

**Explicitly unmaintained.** The README has carried this notice since August
2024:

> "August 2024: Unfortunately, this repository is now in an unmaintained
> state. I don't have the time to maintain it like I would like to and there
> was not enough interest from external developers to contribute to it. Feel
> free to fork it and take over but for now it will stay as is."

All 15 most recent visible commits come from the sole author `gehee`, last
commit 2024-08-16. The code is leaner than PixelPilot (no object
detection/MediaPipe, no audio/Opus, no VPN tunnel feature, no adaptive link
quality, no UDP-forwarding feature) — but:

- Its `devourer` submodule doesn't point at `OpenIPC/devourer` but at
  **his own fork** `gehee/devourer.git`, branch `android-compat` —
  apparently not synced with the now heavily-developed `OpenIPC/devourer`
  for over two years. That's a markedly worse starting point for the
  USB/driver layer than PixelPilot's current pin.
- No built-in foreground-service scaffolding, none of PixelPilot's hardening
  around the USB adapter lifecycle (see `pixelpilot-analysis.md`, "App
  lifecycle" section, and the comments in PixelPilot's `WfbngLink.hpp` about
  robust stop handling).

**Conclusion:** FPVue_android is not recommended as a fork base — the
apparent "leanness" comes at the cost of an outdated, abandoned devourer
fork and missing robustness work that PixelPilot has already done.

### 5. `OpenIPC/fpv4win` — Windows client (architecture reference)

Last commit 2024-12-11, dormant since (only 2 open issues, little repo
traffic). Unsuitable as an Android base (different OS, different toolchain)
— it was only meant as an architecture reference anyway.

**Finding on channel separation:** `src/wifi/WFBReceiver.cpp` instantiates
only a single `video_aggregator` (`Aggregator`, a **custom** lightweight
reimplementation from `src/wifi/WFBProcessor.h`, not wfb-ng's own
`AggregatorUDPv4`) and passes its data directly via an in-process callback to
the Qt/QML player (`src/player`). In the checked state there is **no visible
MAVLink channel** — fpv4win is a pure video client. As a reference for "data
out without rendering it yourself," it's weaker than hoped: PixelPilot's own
`WfbngLink.cpp` (see below) already demonstrates the video/MAVLink
separation considerably more completely, because it uses wfb-ng's own
`AggregatorUDPv4` class directly and unmodified for both channels. fpv4win
therefore offers no additional insight beyond PixelPilot itself and won't be
explored further.

## Most important finding (cross-cutting)

While analyzing the code for `pixelpilot-analysis.md`, it turned out that
PixelPilot's own integration layer (`app/wfbngrtl8812/src/main/cpp/WfbngLink.cpp`)
**already does almost exactly what this project needs**: it instantiates a
separate `wfb-ng::AggregatorUDPv4` for video, MAVLink, and a third "tunnel"
channel, each of which sends decrypted/FEC-corrected payload unmodified via
`sendto()` to `127.0.0.1:5600` (video) resp. `127.0.0.1:14550` (MAVLink) —
regardless of whether anything is listening there. These ports are exactly
the default ports required by the project plan. This substantially changes
the recommendation (see below and `pixelpilot-analysis.md`).

## Recommendation

**Fork base: `OpenIPC/PixelPilot`.**

Rationale:

1. **Activity/maintenance:** PixelPilot itself, along with its two most
   important building blocks (`devourer`, `wfb-ng`), are actively maintained,
   living projects with commits on a daily/weekly cadence. FPVue_android is
   dead, LiveVideo10ms is dead, fpv4win has been dormant for almost two years.
2. **Groundwork already done:** PixelPilot's `WfbngLink.cpp` already contains
   a working video/MAVLink/tunnel channel separation that functionally
   matches the Linux reference's `[gs_video]`/`[gs_mavlink]` model — including
   the exact target ports 5600/14550. Building fresh directly on
   `devourer`+`wfb-ng` would mean redoing this work entirely (USB permission
   flow, JNI bindings, robust adapter start/stop handling — see the
   extensive comments about thread races in `WfbngLink.hpp`) — with no
   benefit, since PixelPilot reuses these building blocks unmodified anyway.
3. **Less rework than assumed:** The step envisioned in the project plan
   (Phase 3) — "add a `DatagramSocket` that sends video bytes to
   `127.0.0.1:5600`" — **already exists** and doesn't need to be built. The
   actual work shifts from "add a sink" to "remove/disable the two existing
   internal consumers (video decoder on port 5600, MAVLink parser on port
   14550) so QGroundControl can bind those ports instead." That's a
   significantly smaller, lower-risk change than originally assumed.
4. **FPVue_android drops out as an alternative fork base:** declared
   unmaintained by its own author, plus an outdated custom devourer fork.
5. **A from-scratch build directly on devourer+wfb-ng drops out too:** high
   effort with no payoff, since PixelPilot's integration layer is already
   lean, current, and functionally fit for purpose.

**Prep work recommended before Phase 1 (a recommendation, not a blocker):**

- When creating the fork, bump the `wfb-ng` submodule from `0da52790`
  (2025-08-05) to a current `master` commit (at least `3504a38`,
  2026-08-27) and compile/test it against PixelPilot's `WfbngLink.cpp` — a
  year of lag on the cryptographically relevant module is avoidable and
  shouldn't be carried along unnecessarily.
- The `devourer` submodule can stay on PixelPilot's pin with acceptable risk,
  or also be bumped to `HEAD` (only 38 commits / 1 month behind, no
  structural API changes visible at the locations cited in
  `pixelpilot-analysis.md`) — before bumping, briefly check whether issue
  #352 (40 MHz Jaguar1) is relevant and fixed.
