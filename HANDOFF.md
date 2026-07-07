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
