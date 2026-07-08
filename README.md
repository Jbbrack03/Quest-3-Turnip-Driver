# Quest 3 Turnip Driver

Patched Mesa **Turnip** (open-source Adreno Vulkan driver) that gives the **Eden** Switch emulator a working, correct GPU driver on the **Meta Quest 3** (Adreno 740v3 / XR2 Gen 2, HorizonOS, no root).

Switch games render correctly and play. Balatro, Diablo III, and other titles boot to gameplay with full GPU acceleration (tiling, UBWC, LRZ). This is the first known working Turnip driver for the Quest 3.

**Full technical brief → [`HANDOFF.md`](HANDOFF.md)** — every root cause, the evidence, the build environment, and the operational workflow.

## Status: working

Three distinct root causes were found and fixed, each verified with on-device evidence (kernel-source cross-checks, driver-side SPIR-V dumps, driver-side frame dumps):

1. **KGSL allocator / SELinux** — HorizonOS blocks the `GPUMEM_ALLOC_ID` (0x34) ioctl stock Turnip uses to allocate GPU memory. Rerouted to the allowlisted `GPUOBJ_ALLOC`/`FREE`/`INFO` family. Verified byte-equivalent against Meta's published Quest 3 kernel source.
2. **Eden's dual push-constant SPIR-V** — Eden emits shaders with two push-constant blocks in one stage (invalid SPIR-V that proprietary drivers tolerate). Relaxed Mesa's front-end to accept it the same way.
3. **Swapchain layout mismatch** — Meta's gralloc hides its buffer-layout metadata from Mesa (one extra word before the QTI magic), so Turnip rendered tiled frames into buffers the compositor scans out linear → full-screen corruption. Default to linear when the layout is unknown.

## Performance & smoothness notes

- The Quest 3 panel supports only **72 / 90 / 120 Hz — no 60 Hz mode.** A 60 fps Switch game therefore can't land on an even cadence unless the framerate is *stable*. Judder on a wandering framerate is inherent to the panel, not a driver fault.
- Best results: **120 Hz panel + Eden VSync = Mailbox + internal resolution low enough that the game holds a locked 60** (clean 2:1). Diablo III is GPU-bound to ~55 fps at 0.75×; lighter titles (Mario Kart 8, Hades II) should pin 60 with headroom.
- **Memory:** HorizonOS enforces a documented per-app cap (~5.75 GiB PSS on Quest 3). Heavy titles that stream large amounts of texture/guest memory can hit it. No root-free way to raise the cap; the lever is reducing Eden's footprint.

See HANDOFF.md for the full pacing/memory analysis and per-game tuning guidance.

## Files

- `HANDOFF.md` — the complete writeup (all patches, evidence, build env, workflow, open items).
- `patch1-kgsl-gpuobj-alloc.diff` — Patch #1 (KGSL GPUOBJ allocator).
- `patch2-spirv-multi-push-const.diff` — Patch #2 (multi-push-constant SPIR-V tolerance).
- `patch3-gralloc-linear-fallback.diff` — Patch #3 (linear swapchain fallback for HorizonOS gralloc).
- `patch-quest3-release-v1.0.diff` — all three patches combined + sparse-path ioctl fix, as shipped in v1.0 (no debug instrumentation).
- `mesa-debug-full-snapshot.diff` — the full debug build (adds a present-time frame dumper, SPIR-V dump-on-failure, push-constant/pipeline logging, and an adb-writable `TU_DEBUG_FILE` path). Apply for further investigation.
- `Turnip_Quest3_v1.0.adpkg.zip` — the built release driver (adrenotools package). Install via Eden → GPU Driver Manager.
- `android-aarch64-cross.ini` — meson cross file for the Android arm64 build.

## Building

The Mesa source tree is **not** in this repo (large third-party source; see `.gitignore`). Reconstruct it per HANDOFF.md § "Build & tooling environment": clone Mesa at the pinned commit, apply `patch-quest3-release-v1.0.diff`, and cross-compile with the NDK using `android-aarch64-cross.ini`. Package the resulting `libvulkan_freedreno.so` as `vulkan.ad07xx.so` in an `.adpkg.zip` with a `meta.json`.

## Installing on the headset

Push `Turnip_Quest3_v1.0.adpkg.zip` to the Quest, then in Eden: **Settings → Graphics → GPU Driver Manager → Install** → pick the zip → **select it** as active → launch a game. (Editing `config.ini`'s `driver_path` does **not** work — Eden caches the extracted `.so`; you must install through the UI.)

## Not in this repo (by design)

Switch firmware, `prod.keys` / `title.keys`, game dumps, and the Eden APK are **excluded** (`.gitignore`) — copyrighted / piracy-sensitive. Debug/intermediate driver builds are also excluded.

## License

Our patches and the built Turnip driver derive from Mesa, which is MIT-licensed.
