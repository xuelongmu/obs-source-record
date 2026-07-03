# Source Record — Stability & Performance Playbook

Practical configuration guidance for running many Source Record filters reliably,
written for a high-core-count CPU + NVIDIA GPU workstation (tested reasoning:
Threadripper + RTX 4090, OBS 30/31). Every filter is a *complete extra pipeline* —
its own composite render, scaler, video/audio encoders, and muxer process — so the
knobs below are about not paying for pipeline stages you don't need.

## 1. Use NVENC, per filter — the single biggest CPU win

**The filter's encoder defaults to whatever your OBS profile's recording encoder
is.** If your profile uses x264 (the Simple-mode default), every new Source Record
filter silently defaults to software x264: a per-frame GPU→RAM download plus a
software encode, multiplied by every recording filter. That is the usual cause of
"CPU pegged while recording a few sources."

For each filter, set **Video Encoder → NVENC HEVC** (or **NVENC AV1** on RTX 40xx —
better quality per bitrate; H.264 only if downstream tooling requires it). The
plugin automatically selects OBS's texture-based NVENC encoders
(`obs_nvenc_*_tex`), which encode directly from the GPU texture with no
system-memory copy — near-zero CPU per stream.

Alternatively, change the OBS profile's recording encoder to NVENC so new filters
inherit it.

### NVENC session budget

- GeForce drivers cap **concurrent NVENC sessions at 8 system-wide** (driver
  550+; older drivers allow 5, pre-2023 allow 3). Quadro/RTX-Pro cards are
  uncapped.
- Every active recording counts: the main OBS recording/stream, each Source
  Record filter, plus anything else on the box (ShadowPlay, other apps).
- An RTX 4090 has **two hardware NVENC engines**; sessions are load-balanced
  across them. 6–8 concurrent 1080p60 HEVC sessions is comfortable; watch
  "Video Encode" utilization in Task Manager / `nvidia-smi dmon -s u`.
- **Budget sessions deliberately.** A failed `obs_output_start` (session limit
  reached) leaves the filter silently not recording, and on older builds of this
  plugin a failed start followed by teardown could leak or hang. Stay under the
  cap rather than probing it.

If you genuinely need more concurrent encodes than the session cap allows, spread
them across encoders: put some filters on QSV (if the CPU/iGPU has it) or accept
1–2 x264 `veryfast` sessions — a Threadripper has the cores; the problem is only
when *all* filters land on x264 by default.

## 2. Scale and decimate at the encoder — it's GPU-side and cheap

Per filter:

- **Scale** (checkable group): downscales on the GPU before encoding
  (`obs_encoder_set_scaled_size` + GPU scale type). A 4K source recorded at
  1080p quarters the encode cost. Pick **Lanczos** or **Area** for downscales.
- **Framerate** (divisor): records at fps/N without touching the source. Talking
  heads / static feeds rarely need more than 30.

Both settings reduce NVENC session load too, stretching the 8-session budget.

## 3. Audio: avoid "All tracks" unless you need it

- **Different Audio → All (-1)** creates **six AAC encoders per filter** (one per
  OBS track). Use a single named track, or leave Different Audio off (the filter
  then mixes just its parent source on a private stereo bus — one encoder).
- Each filter with audio "None" runs one lightweight private audio thread; that's
  normal and cheap.

## 4. Idle filters are no longer free — keep them, but know the cost model

With the lazy-view change in this branch, a filter whose mode is **None** or that
is **disabled** tears down its render pipeline and costs ~nothing per frame. On
builds without that change, every attached filter renders its parent a second
time every frame even when idle — if you're on the upstream plugin, *remove*
filters you aren't using rather than disabling them.

## 5. Container and output hygiene

- Use **Hybrid MP4** (`hybrid_mp4`) or **MKV** so a crash doesn't destroy the
  recording; plain MP4 is unrecoverable if the muxer dies mid-write.
- Each recording spawns an `obs-ffmpeg-mux` child process and its own disk
  writes. Many simultaneous recordings want an SSD that can absorb N × bitrate;
  put recordings on NVMe, not the OS drive, if you run 5+.
- **Split File** (time or size) bounds loss if something goes wrong on long
  sessions.

## 6. Change settings when idle, not mid-recording

Changing encoder, audio track, or resolution in the filter properties while the
filter is actively recording forces encoder/pipeline rebuilds on live objects.
The branch hardens these paths, but the robust habit is: stop (disable the
filter or set mode None), change settings, re-enable.

Known-fragile operations this branch specifically fixed — worth retesting after
any upstream rebase:

| Operation | Old failure |
|---|---|
| Different Audio: All → None (or back), then remove filter / quit | Freed OBS's global audio engine → crash/hang |
| Parent source resizing while recording (browser/window/game capture) | Per-frame pipeline rebuild storm; `video_t` freed under live encoders |
| Record → stop → remove filter / quit OBS | Force-stop of an idle output stranded `stopping_event` → hang on exit |
| Disable/re-enable or websocket restart while recording | Queued stop/start tasks ran after the filter was freed (UAF) |
| Track switch while recording | Private audio bus freed under a live audio encoder |

## 7. Quick triage

- **OBS hangs on exit / removing a filter** → almost always an output stop path;
  grab a minidump and check for a thread parked in `os_event_wait` under
  `obs_output_destroy` or `obs_output_actual_start`.
- **High CPU while recording** → check each filter's Video Encoder property; any
  on `x264`? Check `obs_x264` instances in the log.
- **High GPU 3D (not encode) with many filters** → idle filters still rendering
  (pre-lazy-view build), or resize churn from a browser/window capture (look for
  repeated `adding X video mix` lines in the OBS log).
- **Recordings silently missing** → NVENC session cap or bad output path;
  `obs_output_start` failures land in the OBS log as `Failed to start recording`
  / NVENC init errors.
