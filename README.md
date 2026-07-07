# Quest 3 Turnip Driver

Patching Mesa **Turnip** (open-source Adreno Vulkan driver) so the **Eden** Switch emulator gets a working GPU driver on the **Meta Quest 3** (Adreno 740v3 / XR2 Gen 2, HorizonOS, no root).

**Start here → [`HANDOFF.md`](HANDOFF.md)** — full technical brief: root cause, the fix, current open problem, and the complete build environment.

## Status

- ✅ **Root cause found & fixed at init:** HorizonOS SELinux blocks the KGSL `GPUMEM_ALLOC_ID` (0x34) ioctl that stock Turnip uses to allocate GPU memory. Rerouting to the allowlisted `GPUOBJ_ALLOC`/`FREE`/`INFO` ioctls makes the driver load, initialize the Adreno 740v3, and boot games past the previously-fatal GPU-init wall.
- 🔧 **Open problem:** during rendering, memory runs away to ~3.4 GB → OOM-kill, with a black screen. Likely a buffer-mapping correctness bug. See HANDOFF.md § "current open problem."

## Files

- `HANDOFF.md` — the full writeup.
- `patch1-kgsl-gpuobj-alloc.diff` — the driver patch (apply to `mesa/src/freedreno/vulkan/tu_knl_kgsl.cc` at the pinned Mesa commit).
- `android-aarch64-cross.ini` — meson cross file for the Android arm64 build.
- `Turnip_R8_Quest3KGSL.adpkg.zip` — the built, patched driver package (adrenotools format), Patch #1 applied.

## Not in this repo (by design)

Switch firmware, `prod.keys`/`title.keys`, game dumps, and the Eden APK are **excluded** (`.gitignore`) — copyrighted/piracy-sensitive. The Mesa source tree is also excluded; reconstruct it per HANDOFF.md (clone at the pinned commit, apply the diff).

## License

Our patch and the built Turnip driver derive from Mesa, which is MIT-licensed.
