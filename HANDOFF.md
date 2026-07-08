# Quest 3 + Eden Emulator: Turnip GPU Driver — Session Handoff

**Date:** 2026-07-07
**Goal:** Get a working, performant Vulkan GPU driver for the Meta Quest 3 so the Eden (yuzu-fork) Switch emulator renders games well. Sideloaded, **no root**.

---

## TL;DR — where we are

- **Root cause of "no working GPU driver" is FOUND and FIXED at the init level.** HorizonOS's SELinux policy blocks the specific KGSL ioctl (`GPUMEM_ALLOC_ID`, 0x34) that stock Mesa Turnip uses as its main GPU memory allocator. That is why Turnip loads but every game crashes instantly at GPU init with `VK_ERROR_OUT_OF_DEVICE_MEMORY`.
- **We patched Mesa Turnip** to allocate via the *allowlisted* `GPUOBJ_ALLOC` (0x45) / `GPUOBJ_FREE` (0x46) / `GPUOBJ_INFO` (0x47) ioctls instead, cross-compiled it for the headset, and packaged it as an adrenotools `.adpkg.zip`.
- **Result: massive progress.** With the patched driver, Eden now initializes the GPU successfully (recognizes "Turnip Adreno (TM) 740v3", Vulkan 1.4.335), and boots Balatro **past** the previously-fatal wall — into actual rendering.
- **Current blocker (NEW, downstream):** During rendering, Eden's memory balloons from ~170 MB to ~3.4 GB in ~4 seconds and the Android low-memory-killer SIGKILLs it. User also observed a **black screen with a stats overlay** before the kill. So rendering is partly broken AND memory runs away. **This is the open problem to solve next.**

---

## Hardware / software facts (verified on-device)

- **Device:** Meta Quest 3 ("eureka"), HorizonOS build `releases-oculus-14.0-v205`, `ro.vros.build.version=205`, Android 14 / SDK 34, kernel **5.10.237**.
- **GPU:** `/sys/class/kgsl/kgsl-3d0/gpu_model` = `Adreno740v3SXR2230P`. GLES renderer string: `Adreno (TM) 740`, driver `V@0837.0.7 ... Date:01/12/26`.
- **Mesa chip id:** `0x43050b00` → `FD740v3` (Quest 3). Added upstream Mesa 2024-09-25, commit `7968b356f` ("freedreno/devices: Fix A740v3 from Quest 3"). Present in `src/freedreno/common/freedreno_devices.py` around lines 1177–1198. So **chip recognition already works in stock Turnip** — that was never the problem.
- **adb:** USB serial `2G0YC5ZGCY09KN`. Wi-Fi adb was `192.168.0.57:5555` (goes offline when headset sleeps — prefer USB, and a **short** cable; the user's long cable caused mid-transfer data dropouts that killed captures).
- **Eden:** package `dev.eden.eden_emulator` (yuzu fork; activities are still `org.yuzu.yuzu_emu.*`). Installed from `Eden-Android-v0.2.1-standard.apk` in project root.
  - Eden log: `/sdcard/Android/data/dev.eden.eden_emulator/files/log/eden_log.txt` (+ `.old.txt`). This is the most reliable record — survives crashes.
  - Config: `/sdcard/Android/data/dev.eden.eden_emulator/files/config/config.ini`, `[GpuDriver] driver_path=...`.
  - Custom drivers dir: `/sdcard/Android/data/dev.eden.eden_emulator/files/gpu_drivers/`.
  - Games pushed to `/sdcard/Games/`. Firmware (238 NCA files) + `prod.keys`/`title.keys` were installed via Eden's UI first-run; that works.
- **Test game:** Balatro (`0100CD801CE5E000`) — trivial 2D game, good smoke test. Diablo III Eternal Collection NSP also on device (`/sdcard/Games/`, 14 GB) as a heavier test.
- **Games library** (user's backups) at `/Volumes/Final Cut Pro Libraries/Games`.

---

## The diagnosis (how we got the root cause — all evidence, not guesses)

1. **First crash tombstone** (stock R8 Turnip): `SIGABRT` in Eden, abort message `core/core.cpp:338:Load: Failed to initialize system (Error 5)!`. Eden log showed:
   `renderer_vulkan.cpp:169: Vulkan initialization failed with error: VK_ERROR_OUT_OF_DEVICE_MEMORY` at the first BO allocation. On a device with GBs free, "out of device memory" at init = an allocation *call* being rejected, not real pressure.

2. **Kernel audit log (the smoking gun), captured live via `adb logcat` during a Turnip launch:**
   ```
   avc: denied { ioctl } for path="/dev/kgsl-3d0" ... ioctlcmd=0x934
        scontext=u:r:untrusted_app:... tcontext=u:object_r:gpu_device:... tclass=chr_file permissive=0
        app=dev.eden.eden_emulator
   ```
   `ioctlcmd=0x934` decodes as KGSL type `0x09`, nr `0x34` = **`IOCTL_KGSL_GPUMEM_ALLOC_ID`**. `permissive=0` = enforced denial. This is the exact ioctl Turnip's `kgsl_bo_init` uses to allocate GPU memory.

3. **Confirmed the allowlisted alternative empirically.** Across multiple runs, the **only** KGSL ioctl ever denied was `0x934`. `GPUOBJ_ALLOC` (nr 0x45 → 0x945) was **never** denied, even though Turnip's `kgsl_is_virtual_bo_supported` probe calls it during that same init. We also disassembled the **stock Qualcomm driver** (`notgsl.so`, extracted-from-Quest v837) and confirmed it constructs the modern `GPUOBJ_*` ioctls — i.e., the ioctls a *working* sandboxed driver uses are the `GPUOBJ_*` family, not `GPUMEM_ALLOC_ID`.

4. **Pulled the actual SELinux policy** (`/odm/etc/selinux/precompiled_sepolicy`, world-readable; `/sys/fs/selinux/policy` also readable) to try to read the exact `allowxperm` set. Could not fully parse it on macOS (real `setools`/`sesearch` is Linux-only; PyPI `setools` resolved to an unrelated typosquat package — **do not install it**). The empirical evidence in (3) was sufficient and is what we relied on.

**Conclusion:** This is a **HorizonOS-specific SELinux lockdown**, not a hardware or chip-recognition problem. The community narrative ("no Turnip driver works on Quest 3, no workaround") is symptom-level; the real cause is one denied ioctl, and it is patchable in Mesa because the loader (adrenotools) works fine on Quest.

---

## The fix (Patch #1 — DONE, gets us to rendering)

File: `mesa/src/freedreno/vulkan/tu_knl_kgsl.cc`. Replace the SELinux-blocked legacy id-based allocator with the allowlisted object-based one. Three call sites changed:

1. **`kgsl_bo_init`** (main allocator, the fatal path): `kgsl_gpumem_alloc_id` + `IOCTL_KGSL_GPUMEM_ALLOC_ID` → `kgsl_gpuobj_alloc` + `IOCTL_KGSL_GPUOBJ_ALLOC`. Because `GPUOBJ_ALLOC` does not return the GPU address, we query it afterward via `IOCTL_KGSL_GPUOBJ_INFO` (`kgsl_gpuobj_info.gpuaddr`) for the non-replayable / non-lazy path. `bo->size = req.mmapsize` (GPUOBJ_ALLOC does populate `mmapsize`). Flags (cache mode, GPUREADONLY, USE_CPU_MAP) carried over unchanged.
2. **`kgsl_bo_finish`** (free path): `kgsl_gpumem_free_id` + `IOCTL_KGSL_GPUMEM_FREE_ID` → `kgsl_gpuobj_free` + `IOCTL_KGSL_GPUOBJ_FREE`.
3. **`kgsl_is_memory_type_supported`** (probe): same `GPUMEM_ALLOC_ID`→`GPUOBJ_ALLOC`, `GPUMEM_FREE_ID`→`GPUOBJ_FREE` swap, so the probe reflects real support instead of always failing on the denied ioctl.

Note: `kgsl_is_virtual_bo_supported` in this Mesa revision already used `GPUOBJ_ALLOC`/`GPUOBJ_FREE` — no change needed. After patching, `grep GPUMEM_ALLOC_ID/GPUMEM_FREE_ID` in the file returns only comments.

**Verified effect:** patched driver loads and Eden logs:
```
renderer_vulkan.cpp:221: Driver: turnip Mesa driver 25.99.99
renderer_vulkan.cpp:222: Device: Turnip Adreno (TM) 740v3
renderer_vulkan.cpp:223: Vulkan: 1.4.335
renderer_vulkan.cpp:224: Available VRAM: 4.00 GiB
```
Zero KGSL ioctl denials. Balatro loads past GPU init into rendering. **This is the core breakthrough.**

---

## The current open problem (Patch #2 — NOT solved)

**Symptom:** With the patched driver, launching Balatro:
- Eden RSS is stable ~150–170 MB during load, then **explodes: 170 MB → 1184 MB → 3428 MB in ~4 s** once rendering starts.
- Android `lowmemorykiller` fires system-wide ("low watermark is breached … swap is low"), kills many processes, then SIGKILLs Eden's cgroup. No tombstone (SIGKILL leaves none) — matches the earlier "Eden crashed with no crash log, eden_log.txt truncated mid-write" observation.
- User sees **black screen with a stats overlay** briefly before the kill → rendering is producing nothing visible.

**`dumpsys meminfo` at the ~1 GB spike (pid alive):**
```
Native Heap   Pss 654545  PrivateDirty 654504  Rss 655568  HeapAlloc 821883   <-- ~800 MB malloc, growing
Gfx dev       Pss   2116                                                        <-- tracked GPU mem TINY (2 MB)
Other mmap    Pss 352272  PrivateDirty 352248  Rss 352996                       <-- ~344 MB mapped, growing
GL mtrack     21836 ;  EGL mtrack 5731
TOTAL PSS 1088753   TOTAL RSS 1179955
```
**Interpretation:** Properly-tracked GPU memory (`Gfx dev`) is negligible (2 MB), so this is NOT a straightforward leak of KGSL device memory (if `GPUOBJ_FREE` were failing to free, `Gfx dev` would balloon — it doesn't). What grows is **Native Heap (CPU malloc, ~800 MB)** and **Other mmap (~344 MB)**, together to 3.4 GB.

**Leading hypothesis:** Rendering is broken (black screen) — the GPU is not correctly consuming the buffers my patch allocates — so Eden's producer side keeps queuing/allocating work (command buffers, staging, guest-mem commits) that the GPU never drains, and native heap + mmap grow unbounded until OOM. In other words, the memory runaway is likely a *secondary effect* of a buffer-correctness bug, not the primary bug.

**Why buffers may be mis-mapped (the thing to investigate first):** My patch introduced a NEW combination not present in stock Turnip — allocating a normal CPU-visible BO with `GPUOBJ_ALLOC` and then CPU-mapping it via `mmap(fd, offset = bo->gem_handle << 12)` in `kgsl_bo_map`. Stock Turnip only ever CPU-maps `GPUMEM_ALLOC_ID` buffers that way; it uses `GPUOBJ_ALLOC` only for sparse VMAs (not id<<12 CPU mapping). So the open question: **is a `GPUOBJ_ALLOC`'d buffer correctly CPU-mappable via the `id<<12` mmap cookie, and is the `gpuaddr` from `GPUOBJ_INFO` the same memory the CPU map points at?** If CPU and GPU end up pointing at different/uncoherent backing, you get exactly this: black output + ballooning committed pages. `mmap` did NOT fail (no `VK_ERROR_MEMORY_MAP_FAILED`), so if it's wrong it's silently wrong.

**Next diagnostic step we were about to take (not yet done):** add lightweight instrumentation to the patched driver (`mesa_logi`, tag shows as `MESA`/`TU` in logcat) to:
- count live BOs (alloc++ in `kgsl_bo_init`, free-- in `kgsl_bo_finish`) and log periodically → distinguishes a true leak (allocs ≫ frees) from correct-but-mis-mapped.
- log the first N allocation sizes and the returned `iova` (from `GPUOBJ_INFO`) → verify iovas are valid/non-zero and sane.

Other avenues to consider for Fable:
- Compare `GPUOBJ_ALLOC` vs `GPUMEM_ALLOC_ID` KGSL semantics re: whether the object is auto-mapped into the GPU pagetable at alloc and whether `id<<12` mmap yields a coherent shared mapping. May need `KGSL_MEMFLAGS_USE_CPU_MAP` (SVM path, iova = CPU map address) for these BOs instead of the query-gpuaddr path.
- Check whether `req.mmapsize` from `GPUOBJ_ALLOC` matches expectations (not being rounded to something huge).
- Confirm whether the black screen is a GPU pagefault/hang (KGSL faults log to kernel dmesg, which needs root to read — but a hang would show as Eden getting `VK_ERROR_DEVICE_LOST`; watch for that in logcat).
- Rule out that this OOM is generic Eden-on-Turnip behavior vs. patch-specific by checking whether `mmapsize`/iova correctness changes it.

---

## Build & tooling environment (all set up and working)

- **Mesa checkout:** `mesa/` in project root, at commit `5ac41be677` — the SAME commit the known-loading community build "Turnip v26.0.0 R8" (K11MCH1/AdrenoToolsDrivers) was built from. (Deepen-fetched to reach it; it's a Jan 2026 commit.)
- **Cross-compile:** Android NDK r27 at `~/Library/Android/sdk/ndk/27.0.12077973`. Cross file: `android-aarch64-cross.ini` (project root), targets `aarch64-linux-android29`.
- **Python venv:** `buildenv/` (has `mako`, `meson`, `pyyaml`). meson picks this Python (has mako). `ninja` at `~/.local/bin/ninja`.
- **bison:** system bison is 2.3 (too old, breaks ir3 parser). Installed **homebrew bison 3.8.2** at `/opt/homebrew/opt/bison/bin` — must be on PATH before configure/build. flex 2.6.4 (system) is fine.
- **meson configure** (Vulkan-only, GL/EGL disabled). Build dir `mesa/build-android`. Configure command:
  ```
  PATH="/opt/homebrew/opt/bison/bin:/opt/homebrew/opt/flex/bin:$PWD/buildenv/bin:$PATH"
  meson setup build-android --cross-file ../android-aarch64-cross.ini \
    -Dplatforms=android -Dplatform-sdk-version=34 -Dandroid-stub=true \
    -Dgallium-drivers= -Dvulkan-drivers=freedreno -Dfreedreno-kmds=kgsl \
    -Degl=disabled -Dgles1=disabled -Dgles2=disabled -Dopengl=false -Dglx=disabled \
    -Dglvnd=disabled -Dllvm=disabled -Dshared-glapi=disabled \
    -Dvulkan-layers= -Dbuildtype=release -Dstrip=true -Db_lto=false -Dcpp_rtti=false
  ```
- **Build:** `ninja -C build-android src/freedreno/vulkan/libvulkan_freedreno.so` → produces a 14 MB ARM aarch64 `.so`. Strip with NDK `llvm-strip`.
- **Package as adrenotools driver** (`.adpkg.zip`): zip containing the `.so` (named e.g. `vulkan.ad07xx.so`) + `meta.json`:
  ```json
  {"schemaVersion":1,"name":"Turnip R8 Quest3-KGSL patch","description":"...","author":"custom",
   "packageVersion":"1","vendor":"Mesa","driverVersion":"Vulkan 1.4","minApi":27,"libraryName":"vulkan.ad07xx.so"}
  ```

---

## CRITICAL operational gotcha — how to actually load a custom driver in Eden

**Editing `config.ini` `driver_path` does NOT work.** Eden caches the extracted `.so` of the driver you installed via its UI and keeps using it, ignoring a hand-edited path. We wasted a cycle on this: the config pointed at the patched zip but Eden kept running the old R8 (proven by continued `0x934` denials that the patched binary cannot emit).

**The reliable way:** in-headset, Eden → Settings → Graphics → **GPU Driver Manager → Install** → pick the `.adpkg.zip` from `/sdcard/Download/` → **select it** so it's active → then launch the game. This forces a fresh extract+load. Requires the user in the headset (Eden renders as a flat panel app; the file picker needs human hands). Pushing the zip to `/sdcard/Download/` via adb first is fine.

---

## Artifacts on disk (this machine)

- Patched source: `mesa/src/freedreno/vulkan/tu_knl_kgsl.cc` (built).
- Built driver: `mesa/build-android/src/freedreno/vulkan/libvulkan_freedreno.so`.
- Packaged (on device + scratchpad): `Turnip_R8_Quest3KGSL.adpkg.zip` (also in `/sdcard/Download/` and `/sdcard/.../files/gpu_drivers/`).
- Scratchpad (`/private/tmp/claude-501/.../scratchpad/`): `notgsl.asm` (stock driver disasm), `msm_kgsl.h`, `freedreno_devices.py`, `precompiled_sepolicy`, `eden_patched.log` (successful-init run), `meminfo_peak.txt` (the OOM breakdown), captured tombstones under `FS/data/tombstones/`, driver packages under `drivers/` (R8, Qualcomm 837/840/849).
- Memory notes: `~/.claude/projects/-Volumes-Final-Cut-Pro-Libraries-Projects-Quest-3-GPU-Driver/memory/quest3-turnip-project.md` and `quest3-workflow-notes.md`.

## Reference sources
- K11MCH1/AdrenoToolsDrivers (driver packages, incl. extracted Quest 3 Qualcomm blobs): https://github.com/K11MCH1/AdrenoToolsDrivers
- Mesa freedreno device table: `src/freedreno/common/freedreno_devices.py` (chip 0x43050b00 = FD740v3).
- Meta forum thread documenting the symptom (Turnip "missing" on Quest 3/3S): community narrative only, no root cause — we found it.

---

## ADDENDUM 2026-07-07 (evening session) — Patch #2 SOLVED, Balatro RUNS

**The render-phase crash/memory-runaway was diagnosed and fixed. Balatro now runs on the patched Turnip driver (GPU 24-40% busy, stable for minutes).**

### What the problem actually was
- The `VK_ERROR_OUT_OF_HOST_MEMORY` abort (Eden thread `VkPipelineBuild`) was **not** a memory or KGSL problem. `tu_spirv_to_nir` was failing on Balatro's **vertex shader**: Eden's shader emitter produces **two push-constant blocks** in one stage (`rescaling_push_constants` ResolutionInfo + `render_area_push_constants` RenderAreaInfo, both `Offset 0`). That's invalid SPIR-V; proprietary drivers tolerate it, Mesa's `vtn_assert(b->shader->num_uniforms == 0)` (spirv_to_nir.c ~7560) hard-fails, and the failure surfaces as OUT_OF_HOST_MEMORY.
- **Patch #2** (`patch2-spirv-multi-push-const.diff`): accumulate `num_uniforms = MAX2(...)` instead of asserting. Semantics identical to tolerant drivers (honor each block's explicit offsets).
- The KGSL GPUOBJ_* patch (#1) was verified correct twice over: (a) instrumented driver logged every BO alloc/map — all clean, sane gpuaddrs, mmapsize==size; (b) Meta's published Quest 3 kernel source (facebookincubator/oculus-linux-kernel, branch oculus-quest3-kernel-master) confirms GPUOBJ_ALLOC and GPUMEM_ALLOC_ID share `gpumem_alloc_entry()`, gpuaddr is assigned+GPU-mapped at alloc time, and `mmap(offset=id<<12)` resolves by id for both (64-bit process ⇒ no FORCE_32BIT difference).

### Memory behavior (watch item)
RSS peaks ~5 GB during boot-time shader compilation, then settles ~3.4 GB while running. Earlier LMK kills / OOM aborts were this same peak racing the pipeline failure. May OOM heavier games — investigate if so.

### New iteration workflow (no headset needed) — see memory `quest3-iteration-workflow`
1. Swap driver zip in place at `files/gpu_drivers/Turnip_R8_Quest3KGSL.adpkg.zip` — **must bump meta.json name/packageVersion each build or Eden keeps the stale extracted .so** (verify via inode in the avc `granted execute` logcat line).
2. Launch Balatro via `am start -a VIEW -d "content://dev.eden.eden_emulator.user/document/root%2FBalatro.nsp" -n dev.eden.eden_emulator/org.yuzu.yuzu_emu.activities.EmulationActivity` (game copy lives at `files/Balatro.nsp`).
3. `svc power stayon usb` + `KEYCODE_WAKEUP` before launch; `am broadcast -a com.oculus.vrpowermanager.prox_close` to unblank for `screencap`.
4. Clear `files/shader/<titleid>/*` between runs; check `/sys/class/kgsl/kgsl-3d0/gpu_busy_percentage` for liveness.

### Current build state
- `mesa/` has Patch #1 + Patch #2 + Q3DBG debug instrumentation (tu_knl_kgsl.cc, tu_pipeline.cc, tu_shader.cc, vk_nir.c — grep `Q3DBG`). Deployed package: `Turnip_Q3DBG5.adpkg.zip` (project root). For release: revert instrumentation hunks, keep patch1+patch2.
- Remaining: user visual confirmation in headset; Diablo III heavy test; residual RSS; swap `sparse_vma_init`'s legacy `GPUMEM_GET_INFO` (0x36) → `GPUOBJ_INFO` (0x47) for safety.

---

## ADDENDUM 2026-07-07 (late evening) — Patch #3: rendering corruption SOLVED (pending final user confirmation)

**Symptom after Patch #2:** games ran but showed black screen with colored lines/bands. Both Balatro and Diablo III.

**Diagnosis chain (all evidence, no guesses):**
1. Instrumented `vkCmdPushConstants` — Eden's rescaling/render_area push-constant values arrive correct (aliased-at-offset-0 layout is intentional yuzu behavior, confirmed by Eden source research). Push constants exonerated.
2. Built a **driver-side frame dumper** (`tu_QueueSignalReleaseImageANDROID` override in tu_device.cc dumps presented swapchain frames to Eden's files dir as raw RGBA) — this is the key tool: pixel-exact vision of presented frames without headset/compositor.
3. Frame dump filename revealed `tile_mode=3` (TILE6_3) on the swapchain image, while SurfaceFlinger shows the buffer as `mod=0, compressed: false` (linear, stride 1600) → **Turnip rendered TILED into a buffer the compositor reads LINEAR** = the bands.
4. Root cause: Mesa's u_gralloc fallback backend (imapper4 not built in NDK builds; qcom backend rejects Meta's gralloc by module name) probes the QTI `'gmsm'` magic at `data[numFds]` — but Meta's HorizonOS gralloc handle prepends one extra int (`data[numFds]=0x1`, magic at `data[numFds+1]`), so detection fails → modifier = INVALID → Turnip picks tiled.
5. KGSL UBWC config (hbb=16, ubwc 4.0, swizzle 0x6, macrotile 8ch) verified consistent between kernel (oculus-linux-kernel gen7_6_0 core + anorak-gpu.dtsi) and Turnip — NOT a factor.

**Fix (patch3-gralloc-linear-fallback.diff):** in `u_gralloc_fallback.c`, when the private-handle layout is unknown and modifier would be INVALID, default to `DRM_FORMAT_MOD_LINEAR` (matches what HorizonOS actually allocates for app swapchains). Result: frame dumps show pixel-perfect Balatro title screen with full internal tiling/UBWC/LRZ enabled.

**TODO (nice-to-have):** teach the fallback the Meta handle layout properly — scan for `'gmsm'` at `data[numFds+1]` too and read the following flags word for the UBWC bit (0x08000000), so UBWC swapchain buffers would also be detected if HorizonOS ever allocates them.

**Debug tooling added this session (all greppable as Q3DBG):**
- Frame dumper at present time (tu_device.cc, dumps at presents #30/120/300/600/1200, max 5 files).
- `TU_DEBUG_FILE` fallback path hardcoded to `/storage/emulated/0/Android/data/dev.eden.eden_emulator/files/tu_debug.txt` (tu_util.cc) — write e.g. `startup,sysmem,nolrz,noubwc` there via adb to toggle driver debug flags at runtime (sysmem/nolrz are runtime; noubwc applies at app start).
- vkCmdPushConstants logging (tu_cmd_buffer.cc), pipeline/shader failure-path logging (tu_pipeline.cc, tu_shader.cc, vk_nir.c), gralloc handle word dump (u_gralloc_fallback.c).
- For a RELEASE build: strip all Q3DBG hunks + frame dumper + debug-file fallback; keep patches #1 (kgsl GPUOBJ), #2 (spirv multi-push-const), #3 (gralloc linear default).

---

## ADDENDUM 2026-07-07 (night) — Release v1.0 packaged; Diablo III memory ceiling analysis

**Release build:** `Turnip_Quest3_v1.0.adpkg.zip` (project root) — all Q3DBG instrumentation stripped; contains: Patch #1 (KGSL GPUOBJ allocator incl. sparse_vma GPUOBJ_INFO swap), Patch #2 (multi-push-constant SPIR-V tolerance), Patch #3 (linear swapchain fallback), plus the TU_DEBUG_FILE fallback path (inert unless the file exists). Combined diff: `patch-quest3-release-v1.0.diff` (211 lines). Mesa working tree = release state. To restore the debug build: `cd mesa && git checkout . && git apply ../mesa-debug-full-snapshot.diff` (548 lines, includes frame dumper + all logging).

**Install for keeps:** headset → Eden → GPU Driver Manager → Install → `Turnip_Quest3_v1.0.adpkg.zip` (push to /sdcard/Download first). Or zip-swap over the installed driver (meta.json packageVersion=100 differs, so re-extract triggers).

**Diablo III status:** renders pixel-perfect (frame dump proof), but hits HorizonOS's documented per-app memory cap (Quest 3: 5.75 GiB PSS; kill dialog reports RSS ~6.6GB). Measured at main menu: PSS 4.12 GB / RSS 5.7 GB (guest 4GB Switch RAM fully resident as shmem + ~15% duplicate-mapping RSS inflation from Eden's fastmem). Research (Meta docs) confirms: no root-free way to raise the cap; lever depends on whether the governor tracks RSS (→ patch Eden to dedup guest mappings, ~1GB reclaim) or PSS (→ real footprint diet needed, uncertain payoff). DECISIVE NEXT MEASUREMENT: play Diablo until the kill with logcat capturing; read the governor's kill line for metric+threshold. Balatro-class titles unaffected.

---

## ADDENDUM 2026-07-07 (later) — Smoothness: 120 Hz unlocked via adb, no Eden mod

Jitter at 40-55 fps was a refresh-cadence problem (panel at 72/90 Hz vs ~60fps content) compounded by GPU saturation (Eden was set to 1.5x resolution scale, GPU 98% busy).

Fixes applied:
1. Eden global `resolution_setup` 3 (1.5x) → 2 (1x) in config.ini (force-stop Eden before editing).
2. **Display forced to 120 Hz**: `adb shell settings put system min_refresh_rate 120.0` + `settings put system peak_refresh_rate 120.0` → mode 3 (4128x2208@120), verified via `dumpsys SurfaceFlinger | grep refresh-rate`. Persists across reboot. (`cmd display set-user-preferred-display-mode` is blocked by a SecurityException for shell; the settings keys work. `setprop debug.oculus.refreshRate 120` was also set in the same experiment — the settings keys are believed sufficient.) Revert: `settings delete system min_refresh_rate; settings delete system peak_refresh_rate`.
3. Release v1.0 driver deployed via zip-swap (debug build's frame dumper caused 5 one-time hitches).

60fps @ 120Hz = clean 2:1 cadence. Result pending user verification. Battery cost of 120Hz noted. Watch: whether the VR shell reverts the mode when entering/exiting immersive apps.

Diablo III OOM note: user played a long session with NO crash after the first-boot one (earlier kill was likely shader-compile memory spike stacked on gameplay). Memory ceiling analysis retained above in case it recurs.

---

## ADDENDUM 2026-07-07 (smoothness cont'd) — Root cause is the panel, not Eden

Research (2nd agent) + on-device testing conclusions:
- **Quest 3 panel has NO 60Hz mode** (only 72/90/120). Diablo III is VFR ~30-55fps. No fixed panel rate divides evenly into a wandering 45-55 → inherent judder. The Retroid Pocket Flip 2 feels flawless only because it has a NATIVE 60Hz panel matching the game's 60fps target — NOT because of "Eden Legacy" (Legacy is a *compatibility downgrade* Eden REQUIRES for the RP's Adreno 650 / SD865; Quest's Adreno 740 correctly runs standard/PGO build). Eden Legacy is a red herring — do NOT switch the Quest to it.
- User's `cpu_backend=1` = NCE (correct, fastest; Dynarmic=0). Leave it. NCE works on HorizonOS; 30-55fps VFR is a GPU-bound signature, not a CPU-path problem.
- **120Hz is mathematically the best panel rate** (even multiple of both 30 and 60). But the VR compositor re-renders the panel every refresh, so 120Hz costs more GPU than 72 — on a GPU-bound game this can drop fps. Net still usually best via Mailbox.
- **The untested lever = vsync mode.** User was on `use_vsync=2` (FIFO) which quantizes GPU-bound frames to 120/N buckets (120/60/40/30) → severe judder, worst at 120Hz. CHANGED to `use_vsync=1` (Mailbox) — shows newest frame each refresh, decouples render rate. This is the key fix to test. (Eden enum: Immediate=0, Mailbox=1, Fifo=2, FifoRelaxed=3.)

**Applied this session:** config.ini `use_vsync=1` (Mailbox), resolution_setup=2 (1x). Panel refresh override via `settings put system peak_refresh_rate 120.0` + `min_refresh_rate 120.0` (persists in settings; VR shell drives panel to 120 during active gameplay motion, idles to 72 at static screens — confirmed 120 in SurfaceFlinger during earlier gameplay). NOTE: reliable 2D-panel refresh forcing normally needs Quest Games Optimizer ($10) or SideQuest; the adb settings keys work but the compositor gates by motion.

**Two paths for the user (Path A recommended first):**
- Path A (match framerate): 120Hz + Mailbox + limiter 100%. Closest to RP feel. Test THIS first — it's the config now loaded.
- Path B (rock-solid cadence): lock 30fps + 120Hz = clean 4:1. Truly judder-free but half framerate. Caveat: Eden's speed_limit caps game SPEED not fps, so a 60fps-native game locked via speed=50 runs in slow-motion — Path B only works cleanly if a true fps limiter or a 30fps game; needs verification for Diablo (likely 60fps-native).
- Expectation-setting: Quest physically can't match RP's 60-on-60. 120Hz+Mailbox gets closest; perfect smoothness only via 30fps-lock at half rate.

Crash after 10min = OOM again (log truncates mid-write, no fatal, driver healthy to the end; VRAM now self-reports 5.68GiB = near the 5.75 cap — Eden sizes caches to available RAM). Separate issue from smoothness. Memory ceiling remains the hard limit.
