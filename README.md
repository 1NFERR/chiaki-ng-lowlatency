![chiaki-ng Logo](gui/res/chiaking-logo.svg)

# chiaki-ng — low-latency edition (special version)

> **This is a special, unofficial build of [chiaki-ng](https://streetpea.github.io/chiaki-ng/) tuned for the lowest possible input lag on a wired LAN** (PS5 over Ethernet, DualSense plugged into the PS5, PC only shows video and audio). It is **not** the official chiaki-ng release and is not supported by the chiaki-ng project. The version string carries a `-lowlatency` suffix (for example `1.10.0-lowlatency`) and the window title reads "chiaki-ng (low-latency edition)".
>
> Built and tested only through CI (it compiles and packages); **it has not been run against a console by the author**, so treat latency numbers in the notes as estimates and measure on your own setup.

**Download:** see the [Releases](../../releases) page (portable Windows x64 zip: unzip and run `chiaki.exe`).

## What is different from upstream chiaki-ng

| Area | Change |
|---|---|
| Presentation | New setting **Low-latency Presentation (no frame pacing)**, on by default: a decoded frame is presented immediately instead of waiting for the next pacing slot (removes up to one frame, ~8 ms on average at 60 fps). Works with Direct Frame Mapping (default). |
| Audio | Default audio buffer 9600 → 1920 bytes (10 ms), jitter prefill 3 → 2 frames: audio delay roughly 150–200 ms → 40–50 ms. Raise *Audio buffer size* if you hear crackling. |
| Decoder | `LOW_DELAY` decoding flag; slice threading for software decode. |
| Network | Shorter wait for a late video packet (16 → 4 ms); loss reported to the console **after FEC**, so repaired loss no longer makes the console lower its bitrate. |
| Bitrate | Automatic bitrate for local PS5 is **100 Mbit/s**. New option **Constant Bitrate (CBR-like)**, on by default: reports no packet loss so the console keeps the requested bitrate (turn it off on Wi-Fi). The console's `target_bitrate` is logged every 10 s. |
| IDR | *Request IDR Frame on FEC Failure* is on by default and now retries (every 200 ms, 5 times) so the picture cannot freeze waiting for a lost IDR frame. |
| Windows | 1 ms timer resolution, high process priority, no power throttling, MMCSS priority for the network threads, high priority render/frame threads. |
| Defaults | PS5 codec HEVC (SDR; choose HEVC HDR for an HDR display), libplacebo preset *Default* instead of *High Quality*. |

Details, recommended settings (display, VRR, window size, NIC settings) and the research notes are in [`docs/setup/lowlatency.md`](docs/setup/lowlatency.md).

### Limits to be aware of
* Most of the remaining latency is the console's capture/encode time and your display; the client cannot change it.
* A CBR-like mode cannot select the encoder's rate control (the protocol has no such field); it only stops the client from triggering the console's bitrate reduction.
* Settings you saved earlier are kept; new defaults apply to fresh configurations.

### Building
The Windows x64 portable build is produced by `.github/workflows/build-windows-x64-portable-lowlatency.yml` (GitHub Actions, `windows-2022`). See [`docs/diy/buildit.md`](docs/diy/buildit.md) for local builds.

### Credits and license
Based on [streetpea/chiaki-ng](https://github.com/streetpea/chiaki-ng) and the original [Chiaki](https://git.sr.ht/~thestr4ng3r/chiaki). Licensed under the same terms as upstream (AGPL-3.0 with OpenSSL exception, see `COPYING` and `LICENSES`). This project is not endorsed or certified by Sony Interactive Entertainment LLC.

---

# [chiaki-ng](https://streetpea.github.io/chiaki-ng/) (upstream README)

An open source PlayStation remote play project serving as the next-generation of Chiaki with improvements and ongoing support now that the original Chiaki project is in maintenance mode only. [Click here to see the accompanying site for documentation, updates and more](https://streetpea.github.io/chiaki-ng/).

## Discord
[chiaki-ng community Discord](https://discord.gg/tAMbRuwXDH)

## Disclaimer
This project is not endorsed or certified by Sony Interactive Entertainment LLC.

Chiaki is a Free and Open Source Software Client for PlayStation 4 and PlayStation 5 Remote Play
for Linux, FreeBSD, OpenBSD, Android, macOS, Windows, Nintendo Switch and potentially even more platforms.
