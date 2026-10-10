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
