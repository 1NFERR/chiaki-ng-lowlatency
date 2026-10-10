# Low-latency profile (LAN + HDR)

Target: PS5 on wired LAN, DualSense plugged into the PS5 (so only video/audio latency matters), HDR on.

## What this fork changes
| Area | Change | Effect |
|---|---|---|
| Audio | Default buffer 9600 -> 1920 bytes (10 ms), jitter prefill 3 -> 2 frames | ~150-200 ms -> ~40-50 ms audio delay |
| Decoder | `AV_CODEC_FLAG_LOW_DELAY`; slice threading for software decode | no reorder/lookahead delay, no frame-thread delay |
| Windows | 1 ms timer resolution, HIGH priority class, EcoQoS off, MMCSS "Games" + highest priority on network threads | removes 15.6 ms scheduler quantization and OS throttling |
| Defaults | PS5 local codec = HEVC HDR; 1080p auto bitrate = 30 Mbit/s | better picture, bandwidth is free on LAN |

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
