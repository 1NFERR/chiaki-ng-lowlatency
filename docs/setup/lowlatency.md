# Low-latency profile (LAN + HDR)

Target: PS5 on wired LAN, DualSense plugged into the PS5 (so only video/audio latency matters), HDR on.

## What this fork changes
| Area | Change | Effect |
|---|---|---|
| Audio | Default buffer 9600 -> 1920 bytes (10 ms), jitter prefill 3 -> 2 frames | ~150-200 ms -> ~40-50 ms audio delay |
| Decoder | `AV_CODEC_FLAG_LOW_DELAY`; slice threading for software decode | no reorder/lookahead delay, no frame-thread delay |
| Windows | 1 ms timer resolution, HIGH priority class, EcoQoS off, MMCSS "Games" + highest priority on network threads | removes 15.6 ms scheduler quantization and OS throttling |
| Defaults | PS5 local codec = HEVC (SDR; pick HEVC HDR in Settings for an HDR display); local auto bitrate = 100 Mbit/s | better picture, bandwidth is free on LAN |

Already present upstream and kept: immediate present mode (vsync off), swapchain depth 1, frame queue depth 2, no frame mixer, zero-copy hardware decode, frame flushed to the decoder as soon as enough FEC units arrived.

## Recommended settings
* Resolution 1080p, 60 fps, codec HEVC HDR (PS5: enable HDR in system settings and "Enable HDR" only if the display is HDR).
* Hardware decoder: `auto` (Vulkan/D3D11VA), zero copy on, VSync **off**, frame mixer **none**, placebo preset **Fast**.
* Windows: set the monitor to its highest refresh rate; use "Fullscreen"/borderless; disable Windows HDR auto-brightness tricks.
* If audio crackles raise *Audio buffer size* (3840 = 20 ms, 4800 = 25 ms).
* Network: wired gigabit, PS5 and PC on the same switch, no VPN.

## Bitrate and "CBR"
* Automatic bitrate for local PS5 sessions is now **100 Mbit/s** (slider range was already 2-100). The console may still cap what the encoder really produces; the measured value is shown in the stream stats.
* **Constant Bitrate (CBR-like)** checkbox (Settings, next to *Packet Loss Reported Max*): the Remote Play protocol only carries a requested bitrate (`bwKbpsSent` / `RP-StartBitrate`) and has no field to select the encoder's rate-control mode. The console lowers its bitrate (VBR-like) when the client reports packet loss, so this option reports 0% loss and the console keeps the requested bitrate. On a clean wired LAN this gives stable frame sizes; on a lossy link it can cause frame loss instead of adaptation, so leave it off on Wi-Fi.

## Research notes (input lag)
* Upstream chiaki-ng issue [#807](https://github.com/streetpea/chiaki-ng/issues/807): the 2026 frame-pacing/presentation rework added ~50 ms of display latency (queue depth, deferred swap, pending-frame staging). This fork already defaults to *direct frame mapping* (latest decoded frame wins, no queue) with immediate present, vsync off and swapchain depth 1 - the closest equivalent of the v1.10.0 arrival-driven path. Keep **Direct Frame Mapping on, Vulkan deferred swap off, frame mixer none**.
* Android fork [Wnt/chiaki-ng](https://github.com/Wnt/chiaki-ng) measured many transport/decoder knobs (socket tuning, DSCP, recvmmsg batching, shorter feedback interval, big-core pinning): none showed a latency gain on a clean link, so they were not added here. What did help in this fork: smaller audio queue, LOW_DELAY decoding, timer resolution/thread priority on Windows, shorter reorder wait (16 ms -> 4 ms, only matters when a packet is late/lost).
* Moonlight reports ([moonlight-android#1096](https://github.com/moonlight-stream/moonlight-android/issues/1096), [moonlight-qt#1447](https://github.com/moonlight-stream/moonlight-qt/issues/1447)): extra frame pacing/buffering adds 40-100 ms; borderless fullscreen is better than exclusive fullscreen with vsync. Use borderless fullscreen, a high-refresh display (120/144 Hz) and keep pacing off.
* The remaining latency is mostly the console's capture+encode time and the display; these cannot be changed from the client.

## IDR request on FEC failure
Enabled by default. When a video frame cannot be recovered by FEC the client asks the console for a fresh IDR frame and drops P-frames until it arrives. The request is now resent every 200 ms if no IDR shows up (lost request or lost IDR), and after 5 retries decoding resumes without it, so the picture can no longer freeze waiting for an IDR that never comes. An existing saved setting (unchecked) is kept; tick it once in Settings.


## Second research round (what the code and the web showed)
### Input lag: the frame pacing wait (biggest finding)
Before every `pl_swapchain_submit_frame` the render thread slept (`throttleFramePresentation`) until the next slot of a pacing grid locked to the stream frame interval. With direct frame mapping the frame has already been picked at that point, so the sleep delays a frame that is already getting stale by up to one frame interval (~16.7 ms at 60 fps, ~8 ms on average). This is the mechanism behind the ~50 ms regression reported upstream in [chiaki-ng#807](https://github.com/streetpea/chiaki-ng/issues/807). New setting **Low-latency Presentation (no frame pacing)**, on by default, submits the frame immediately (only with Direct Frame Mapping). Cost: frame spacing follows network arrival, so a little judder is possible; with vsync off and a wired LAN this is the lowest-latency configuration.

### CBR
The console has no client-selectable rate-control mode. What it does have (see the vitaki-vitarps5 investigation [#206](https://github.com/mauricio-gg/vitaki-vitarps5/issues/206)) is a **downward-only bitrate ratchet driven by the loss the client reports** (target bitrate steps down within a session, e.g. 5 Mbit -> 1.6 Mbit, and does not recover). Two things were wrong in the client: loss was counted before FEC, so units repaired by FEC were reported as loss, and nothing could stop the ratchet. Now:
* Loss is reported after FEC: complete frames report 0, FEC-repaired frames a quarter of the repaired units, unrecoverable frames in full.
* **Constant Bitrate** (default on): reports no loss at all, so the console keeps the requested bitrate (100 Mbit/s). Turn it off on Wi-Fi.
* The console's current `target_bitrate` is logged every 10 s (`StreamConnection console target bitrate`), so you can check that it stays at the requested value. A third-party experiment ([atameric/ps5-portal-high-bitrate](https://github.com/atameric/ps5-portal-high-bitrate)) saw the PS5 honour ~97 Mbit/s for a 100 Mbit/s request.

### Not done (and why)
* Takion v20 (claimed audio loss protection, unverified, licence unclear) - see [vitaki#410](https://github.com/mauricio-gg/vitaki-vitarps5/issues/410).
* Moving the decoder->render hand-off off the Qt GUI thread: saves well under 1 ms unless the GUI thread is busy, and is a large untestable refactor of `qmlmainwindow.cpp`.
* DSCP/socket tuning/recvmmsg: no measurable gain on a clean LAN in the Android fork's tests.

## Third round: remaining ideas
Done in code: render and frame-delivery threads run at high priority; the default libplacebo preset is now *Default* instead of *High Quality* (EWA upscaling, deband and friends cost several ms of GPU time per frame; a preset you already saved is kept).

Setup-side (no code change):
* **Display scan-out**: a 60 fps stream on a 60 Hz panel is scanned top to bottom over ~16 ms; on 144/240 Hz the same frame is scanned in 7/4 ms. Set the monitor to its maximum refresh rate.
* **Variable refresh (G-Sync/FreeSync)** with vsync off: frames are shown immediately and without tearing - the best combination with immediate present.
* **Fullscreen/borderless covering the whole screen** (Windows "Independent Flip") avoids an extra DWM composition frame; keep the Windows HDR switch on only if you use HDR.
* **Window size = stream size** (or integer ratio): no scaling pass at all. Check the log for "Transferring hardware-decoded frames to software" - if it appears, zero-copy/hardware decoding is not in use and latency will be much worse.
* **Network**: wired gigabit to the same switch as the PS5; disable *Interrupt Moderation* and *Energy Efficient Ethernet* in the NIC driver properties; no VPN/packet-inspection software.
* **PS5**: use performance mode (60 fps) in games, set the console video output to 1080p, avoid HDMI-capture-heavy features (Share/background recording); game-side latency is part of the total.

## Fourth round: code-level audit and measuring where the time goes
Audit of the video path after the pacing fix: there is no other blocking wait or sleep in it. What remains client-side is (a) waiting for all packets of a frame, (b) handing the frame to the decoder, (c) three thread hand-offs (decoder -> frame thread -> GUI thread -> render thread, each ~0.05-0.3 ms when idle) and (d) GPU render time (<1 ms for 1080p -> 1080p with the Default preset, see above). Rewriting the thread model would save well under 1 ms and risks races in a 7000 line file that cannot be tested here, so it was deliberately not done.

New diagnostic: the log now contains a line every 5 s:

`Video pipeline timing over N frames: frame assembly (first to last packet) avg X ms max Y ms, decoder submit avg A ms max B ms`

* **frame assembly** is how long the console takes to deliver one whole frame. If this is several ms, the client could in theory decode earlier (start decoding before the frame is complete); that would be the one worthwhile big rewrite, and this number tells whether it is. If it is below ~2 ms there is nothing to gain.
* **decoder submit** is the time to hand the frame to the hardware decoder.

Run with `CHIAKI_VERBOSE_LOG_NOISE=1` for the detailed `[latency]` render/present lines.

Other research findings (not implemented): `VK_EXT_present_mode_fifo_latest_ready` is a newer tear-free present mode that shows the newest ready frame at vblank (like MAILBOX); an unverified Khronos forum post reports that on Windows `VK_PRESENT_MODE_IMMEDIATE_KHR` can take the "Independent Flip Immediate" path (no DWM composition) in a full-screen borderless window.
