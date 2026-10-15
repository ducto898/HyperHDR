# Fork notes: `perf-dx-grabber`

A personal fork of awawa-dev/HyperHDR. It is not meant to go upstream (no PR).
This file records what changed, what was measured, and what is still only a
hypothesis. Machine: Windows 11, RTX 4070 SUPER, Gigabyte MO27Q28G
(2560x1440, Windows HDR on). HyperHDR config: DX system grabber, 60 fps,
`hardware: true`, `videoMode` 512, smoothing 120 Hz / 80 ms,
`continuousOutput` on, `udpraw` to a local bridge (96 LEDs).

## Why

- On 2026-10-14 the screen went black and came back many times while Netflix
  was playing: 14 `nvlddmkm` event 153 errors ("Error occurred on GPUID: 100").
  They stopped once HyperHDR and the presence app were closed. The cause was
  not proven. HyperHDR's DX grabber was suspected but not shown to be at fault.
  These patches do not fix that driver error.
- On 2026-10-15 at 12:22 the NVIDIA driver was updated (32.0.16.1714 to 1742).
  The stock grabber never recovered: no frames, and the ambilight stayed dark
  until HyperHDR was restarted. That failure is what these patches fix.

## Commits on v22.0.0.0

- `960be18`: recover from driver resets, and read frames back without stalling.
  - Retry a failed start with backoff (1 s, doubling to 10 s). The stock code
    retried only on `E_ACCESSDENIED`.
  - Restart the duplication on any unexpected `AcquireNextFrame` error, for
    example `DEVICE_REMOVED`. The stock code logged it six times and then
    produced nothing.
  - Call `ReleaseFrame` only for an acquired frame, as soon as the copy is
    queued.
  - Two staging textures, read one capture later with
    `D3D11_MAP_FLAG_DO_NOT_WAIT`, then `Flush()` after queueing the copy.
  - Skip the copy when `LastPresentTime == 0` (only the mouse pointer moved).
  - Create the HDR SRV once per duplication texture.
- `feaa0d4`: bind that SRV before every `Draw`. Binding it only when it was
  created made every frame after the first black (centre pixel 0,0,0). This
  was caught because the strip stopped changing, even though CPU had dropped.

## Measured (2026-10-15, user present, screen on)

Load: a fullscreen colour cycle (hue period 8 s). Each run measured 60 s;
original and patched alternated, four runs each over three sessions, the last
one blind.

| | CPU, % of one core | GPU 3D | strip updates/min |
|---|---|---|---|
| original | 6.02 to 6.83 | ~0.17-0.22 % | ~5000 |
| patched | 4.51 to 4.85 | ~0.15-0.18 % | ~5000 |

- The ranges do not overlap, so the patch saves about 1.7 % of one core.
- In the blind test the user could not tell the runs apart by eye.
- Recovery from a real driver reset is **untested**: nobody triggered one on
  purpose.
- When the presence app has switched the monitor off over DDC, no build
  produces frames, so capture tests need the user at the desk.
- Before trusting a lower CPU figure, check that frames are still arriving: a
  pipeline that produces nothing is cheap too.

## Profile of the patched build (one 30 s run each; hypotheses, not conclusions)

Tool: VSDiagnostics (works unelevated), CpuUsageHigh, attached to a
RelWithDebInfo build, then `xperf -a profile -detail` with only HyperHDR's
PDBs. Exclusive time only, no call stacks; Qt, NVIDIA and kernel show as
module names.

Shares of HyperHDR's own samples (static screen / moving):

| bucket | static | moving |
|---|---|---|
| ntoskrnl + win32k* + ntdll + Qt6Core | ~59 % | ~41 % |
| `ImageColorAveraging::calcMulticolorForLeds` | 8.8 % | 4.3 % |
| `nvwgf2umx.dll` (NVIDIA user-mode driver) | 8.3 % | 25.2 % |
| tcpip + afd + NETIO | ~9 % | ~6 % |
| `FrameDecoder::processSystemImageBGRA` | 1.4 % | 8.3 % |

Per-thread sampling: about 3.8 % of a core on a static screen and 4.7 % moving.
So most of the cost does not depend on capture.

**Unverified readings**, each followed by how to check it:
1. The kernel, win32k and Qt share is thread wake-ups from fixed-rate timers
   (grabber 60 Hz, smoothing 120 Hz, LED refresh). Check with a stack profile
   using Qt debug symbols (`aqt` debug_info).
2. On `DXGI_ERROR_WAIT_TIMEOUT` the grabber re-emits `_cacheImage` on every
   tick. Each re-emit runs `processFrame`, `calcMulticolorForLeds` and
   smoothing again. `SystemControl` marks the input inactive after 800 ms
   without a frame, so re-emitting about every 250 ms would be enough. Check
   by counting `processFrame` calls per second on a static screen.
3. The tcpip share is the 125 UDP packets/s to the bridge. Check by measuring
   with smoothing at 60 Hz.

## Candidate improvements (not done)

1. Re-emit the cached frame about every 250 ms instead of every tick. Small
   change, low risk. The black-border detector counts frames, so on a static
   screen it reacts more slowly, which does not matter while nothing moves.
2. Event-driven capture: a worker thread blocking in `AcquireNextFrame` with a
   timeout, instead of a 60 Hz `QTimer` poll. Larger change, medium risk.
3. Config only: smoothing at 60 Hz instead of 120 Hz. This halves the UDP
   traffic and the bridge's wake-ups. Needs a blind visual check.
4. Config only: `videoMode` 256 instead of 512. About 4x less
   `processSystemImageBGRA` work on a moving screen, but coarser sampling.
   Needs a blind check.

Not yet read: smoothing, `LedDevice`/udpraw, muxer, image processor,
black-border detector, web server, mDNS.

## Build and run

- Toolkit is in `..\HyperHDR-toolkit`: Qt 6.8.3 msvc2022_64 via `aqt`, and
  libjpeg-turbo 3.0.3, installed with `__COMPAT_LAYER=RunAsInvoker` so no
  admin is needed.
- `build.bat` builds Release into `build\`, and `build-prof.bat` builds
  RelWithDebInfo into `build-prof\`. Use the VS 2026 generator: Ninja passes
  `/MP` to rc.exe and breaks zstd's .rc.
- Run `build\bin\Release\hyperhdr.exe` with `libjpeg-turbo64\bin` on PATH. It
  uses the same `~\.hyperhdr` database as the installed v22.
- For a log, run with `-d` and redirect stdout. The JSON API refuses
  `logging` without auth, but `serverinfo` works.
- Do not use the Microsoft symbol server with xperf on a machine-wide trace:
  it downloaded 1.7 GB of symbols for every process.
