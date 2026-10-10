**chiaki-ng low-latency edition** - a special, unofficial build of chiaki-ng tuned for minimum input lag on a wired LAN (PS5 over Ethernet, controller connected to the PS5).

**Download:** `chiaki-ng-win_x64-VC-portable-*-lowlatency-*.zip` - unzip anywhere and run `chiaki.exe` (portable, Windows x64).

**Not run against a console by the author** - it compiles and packages in CI only. Latency numbers in the docs are estimates; measure on your own setup.

## Changes vs. upstream
- Low-latency presentation: frames shown immediately, no pacing wait (setting, on by default)
- Audio buffer 10 ms (was ~200 ms total), jitter prefill 2 frames
- LOW_DELAY decoding; 4 ms (was 16 ms) wait for late video packets
- Loss reported after FEC; **Constant Bitrate (CBR-like)** option (on by default); automatic local bitrate 100 Mbit/s
- IDR request on FEC failure on by default, with retry
- Windows: 1 ms timer, high priority, MMCSS for network threads
- Defaults: HEVC (SDR), libplacebo *Default* preset

Recommended settings and full notes: [docs/setup/lowlatency.md](https://github.com/1NFERR/chiaki-ng-lowlatency/blob/claude/vigilant-bell-zi00jq/docs/setup/lowlatency.md)

Not endorsed or certified by Sony Interactive Entertainment LLC. AGPL-3.0 (see COPYING).
