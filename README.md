# STAR-CCM+ Hardware Rendering Fix for AMD Radeon PRO W7900 on Ubuntu 22.04

> A working fix for "Visualization performance impaired. Hardware accelerated rendering not active" when running Simcenter STAR-CCM+ on a modern AMD RDNA3 workstation GPU.

## Short description (for the GitHub repo)

Fix STAR-CCM+'s software-rendering fallback on AMD Radeon PRO W7900 (RDNA3) under Ubuntu 22.04. STAR-CCM+ ships an old bundled Mesa stack that doesn't recognize gfx1100 and an old `libstdc++` that blocks the system's `radeonsi` DRI driver from loading. This repo documents the root cause and provides a 3-step fix that restores native GPU-accelerated rendering without breaking the solver.

---

## Table of contents

- [Symptom](#symptom)
- [Tested environment](#tested-environment)
- [Why STAR-CCM+ falls back to software rendering](#why-star-ccm-falls-back-to-software-rendering)
- [How to confirm you have the same problem](#how-to-confirm-you-have-the-same-problem)
- [The fix (3 steps)](#the-fix-3-steps)
- [Verification](#verification)
- [How to revert](#how-to-revert)
- [Notes for other GPUs / versions](#notes-for-other-gpus--versions)
- [Debugging guide for other users](#debugging-guide-for-other-users)
- [Risk disclosure](#risk-disclosure)
- [Acknowledgements](#acknowledgements)

---

## Symptom

When opening a Scene in Simcenter STAR-CCM+, the `Scenes` node in the simulation tree shows a yellow warning triangle, and the Properties panel at the bottom displays:

```
Visualization performance impaired. Hardware accelerated rendering not active.
```

Rotating, zooming, or interacting with even modestly sized meshes is visibly laggy because rendering is happening on the CPU instead of the GPU.

Meanwhile, the GPU itself works perfectly fine for everything else — including STAR-CCM+ GPGPU computation via ROCm/HIP. The problem is **strictly a rendering-path issue**, not a driver, ROCm, or hardware problem.

## Tested environment

| Component | Version |
|---|---|
| GPU | AMD Radeon PRO W7900 Dual Slot (48 GB, RDNA3, gfx1100, PCI ID `1002:744a`) |
| CPU / Platform | AMD Threadripper on Gigabyte TRX50-AERO-D |
| OS | Ubuntu 22.04 LTS |
| Kernel | 6.8.0-107-generic |
| Display server | X11 (Xorg) |
| AMD kernel driver | `amdgpu-core 6.2.60203` (installed via `amdgpu-install`, open-source AMDGPU; **NOT** AMDGPU-PRO) |
| Mesa (system) | 23.2.1-1ubuntu3.1~22.04.2 |
| OpenGL (system, via `glxinfo`) | 4.6 Compatibility Profile, vendor AMD, renderer `GFX1100 (gfx1100, LLVM 15.0.7, DRM 3.57)` |
| `libLLVM-15.so.1` (system, used by radeonsi) | requires `GLIBCXX_3.4.30` |
| System `libstdc++.so.6` | provides up to `GLIBCXX_3.4.30` |
| Simcenter STAR-CCM+ | 20.06.007-R8 (User Guide PDF dated 2025-09-30) |
| STAR-CCM+ bundled Mesa | 21.1.8-cda-002 (released 2021) |
| STAR-CCM+ bundled `libstdc++.so.6.0.29` | provides up to `GLIBCXX_3.4.29` |

This fix is also expected to apply, with minor path changes, to:

- AMD Radeon PRO W7800 (gfx1100) and other RDNA3 cards
- Other STAR-CCM+ 20.0x versions where the bundled `libstdc++` is older than the system's `libLLVM` requires
- Other Ubuntu 22.04 / 24.04 systems using the open-source AMDGPU stack

Per the official STAR-CCM+ 2510 User Guide (page corresponding to "GPGPU Supported Hardware"):

> AMD Radeon PRO W6800, V620, W7800, and W7900 are recommended for use in workstations. **The AMDGPU driver is required, while the AMDGPU-PRO driver is unsupported.**

So this issue affects what is officially the *recommended* configuration. Do **not** install AMDGPU-PRO to try to fix it — that is explicitly unsupported by Siemens and would break GPGPU.

## Why STAR-CCM+ falls back to software rendering

There are **two independent failures** that compound. Fixing only one is not enough.

### Failure 1: STAR-CCM+'s graphics checker uses bundled Mesa 21.1.8

When STAR-CCM+ launches with `-graphics auto` (the default), it runs an internal probe to decide whether hardware acceleration is available. The probe uses STAR-CCM+'s **bundled** Mesa libraries:

```
$STAR_HOME/mesa/21.1.8-cda-002/linux-x86_64-2.17/gnu9.2/...
```

Mesa 21.1.8 was released in mid-2021 and **does not contain the gfx1100 (RDNA3) device tables**. RDNA3 support was added in Mesa 22.x. So the bundled Mesa probe sees the W7900, fails to recognize it, reports "no hardware acceleration", and STAR-CCM+ silently switches to its OpenSWR software renderer.

This shows up in the Test Graphics report as:

```
library:  STAR Mesa OpenGL
window:   OSMesaOffscreenRenderWindow
OpenGL vendor string:   Intel Corporation        ← OpenSWR identifies as Intel
OpenGL renderer string: SWR (LLVM 12.0.0, 256 bits)
OpenGL version string:  3.3 (Core Profile) Mesa 21.1.8
```

`OSMesaOffscreenRenderWindow` means it isn't even talking to the X server — it's drawing into a CPU-side buffer.

The `-graphics native` command-line argument (documented for some STAR-CCM+ versions, undocumented in others) bypasses the graphics checker and forces the use of the **system** OpenGL stack. This is necessary but not sufficient.

### Failure 2: STAR-CCM+'s bundled `libstdc++` blocks system `radeonsi`

Once `-graphics native` is in play, STAR-CCM+ tries to load the system's `radeonsi_dri.so` (the actual driver that talks to the W7900). The dynamic linker resolves `radeonsi`'s dependency on `libLLVM-15.so.1`, but `libLLVM-15.so.1` requires symbol version `GLIBCXX_3.4.30` from `libstdc++.so.6`.

STAR-CCM+'s `LD_LIBRARY_PATH` puts its own bundled `libstdc++.so.6.0.29` ahead of the system path. That version only provides up to `GLIBCXX_3.4.29`. So loading fails with:

```
libGL error: MESA-LOADER: failed to open /usr/lib/x86_64-linux-gnu/dri/radeonsi_dri.so:
  /home/USER/.../star/lib/linux-x86_64-2.28/system/clang17.0-64/libstdc++.so.6:
  version `GLIBCXX_3.4.30' not found (required by /lib/x86_64-linux-gnu/libLLVM-15.so.1)
libGL error: failed to load driver: radeonsi
```

This is the actual root cause. To fix it, the system `libstdc++.so.6` has to be visible to `radeonsi` instead of STAR-CCM+'s bundled one.

### Compounding issue: Mesa DRI search path

There is also a minor pathing wrinkle on Ubuntu. The Mesa loader searches three paths in order:

```
/usr/lib/x86_64-linux-gnu/dri
${ORIGIN}/dri
/usr/lib/dri
```

Ubuntu's multiarch layout puts the actual driver at `/usr/lib/x86_64-linux-gnu/dri/`, and `/usr/lib/dri` does not exist. This is fine when the first path works, but if the first path fails for any reason (like Failure 2), the error message reports the **last** path tried, which is the misleading `/usr/lib/dri does not exist`. Adding a symlink covers all three paths and makes the real error easier to spot.

## How to confirm you have the same problem

Inside STAR-CCM+, right-click the `Scenes` node in the simulation tree and select **Test Graphics**. If the report contains:

```
library:  STAR Mesa OpenGL
window:   OSMesaOffscreenRenderWindow
OpenGL renderer string:  SWR (...)
```

…you have the same software-rendering fallback. The official Siemens-documented troubleshooting flow for "A Scene Will Not Open" / hardware acceleration issues uses this same `Test Graphics` button.

To confirm Failure 2 specifically, run from a terminal:

```bash
LIBGL_DEBUG=verbose starccm+ -graphics native 2>&1 | grep -iE "GLIBCXX|radeonsi"
```

If you see `GLIBCXX_3.4.30' not found`, that's Failure 2.

You can also verify your system OpenGL stack is independently healthy:

```bash
glxinfo | grep -E "direct rendering|OpenGL vendor|OpenGL renderer"
```

Expected output on a working W7900 system:

```
direct rendering: Yes
OpenGL vendor string: AMD
OpenGL renderer string: GFX1100 (gfx1100, LLVM 15.0.7, ...)
```

If the system OpenGL is broken too (e.g. shows `llvmpipe`), that's a different problem — fix that first before applying this README.

## The fix (3 steps)

> **Before you start:** Save and close any open simulations. Step 2 modifies STAR-CCM+ installation files; if you have a deadline, do this in a maintenance window. All steps are reversible — see [How to revert](#how-to-revert).

### Step 1 — Add the missing Mesa DRI symlink

```bash
sudo ln -s /usr/lib/x86_64-linux-gnu/dri /usr/lib/dri
```

Verify:

```bash
ls -la /usr/lib/dri/radeonsi_dri.so
# expected: a real file, not a broken symlink
```

This is harmless to any other software on the system and benefits other OpenGL applications too.

### Step 2 — Disable STAR-CCM+'s bundled `libstdc++`

Replace `<STAR_HOME>` with your installation path. For our case it was `/home/aero404/20.06.007-R8/STAR-CCM+20.06.007-R8`.

```bash
cd <STAR_HOME>/star/lib/linux-x86_64-2.28/system/clang17.0-64/

# Confirm it's older than what the system needs
strings libstdc++.so.6.0.29 | grep "^GLIBCXX_3" | sort -V | tail -3
# Expected: GLIBCXX_3.4.27 / 3.4.28 / 3.4.29 (no 3.4.30)

strings /usr/lib/x86_64-linux-gnu/libstdc++.so.6 | grep "^GLIBCXX_3" | sort -V | tail -3
# Expected: 3.4.28 / 3.4.29 / 3.4.30 (or higher)

# Rename the real file and both symlinks (reversible)
sudo mv libstdc++.so.6.0.29 libstdc++.so.6.0.29.disabled
sudo mv libstdc++.so.6      libstdc++.so.6.disabled-link
sudo mv libstdc++.so        libstdc++.so.disabled-link

# Verify
ls -la libstdc++*
```

After this, STAR-CCM+'s dynamic linker will fall back to the system `libstdc++.so.6`, which is ABI-backward-compatible. The `radeonsi` driver will now load successfully.

> ⚠️ This step changes the C++ runtime used by **all** STAR-CCM+ processes, including the solver. Validation in [Verification](#verification) is mandatory before relying on this for production work.

### Step 3 — Create a launcher script that uses `-graphics native`

```bash
cat > ~/starccm-fixed.sh << 'EOF'
#!/bin/bash
# STAR-CCM+ launcher with hardware-accelerated rendering on AMD W7900.
# Forces native system OpenGL instead of the bundled OpenSWR software renderer.
exec <STAR_HOME>/star/bin/starccm+ -graphics native "$@"
EOF
chmod +x ~/starccm-fixed.sh
```

Replace `<STAR_HOME>` with the full path to your STAR-CCM+ installation.

Optionally put it on `PATH`:

```bash
mkdir -p ~/.local/bin
mv ~/starccm-fixed.sh ~/.local/bin/starccm-vis
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Now `starccm-vis` launches STAR-CCM+ with hardware rendering.

## Verification

### Verify rendering is hardware-accelerated

Launch with the new launcher:

```bash
~/starccm-fixed.sh
# or  starccm-vis  if you put it on PATH
```

Right-click `Scenes` → `Test Graphics`. The report should now show:

```
library:  native
window:   vtkXOffscreenOpenGLRenderWindow
rendering: direct
OpenGL vendor string:   AMD
OpenGL renderer string: AMD Radeon PRO W7900 Dual Slot  (gfx1100, LLVM 15.0.7, ...)
OpenGL version string:  4.6 (Core Profile) Mesa 23.2.1
```

Key indicators of success:

- `library: native` (not `STAR Mesa OpenGL`)
- `rendering: direct` (the OpenGL commands go directly to the GPU)
- `OpenGL renderer` names your actual GPU
- The yellow warning triangle on the `Scenes` node is gone

### Validate the solver — do this before relying on it for production

Step 2 substitutes the C++ runtime. `libstdc++` is ABI-backward-compatible by design, but no compatibility guarantee is absolute for a large C++ codebase like STAR-CCM+.

1. Open a small simulation you have run before and whose results you know well.
2. Run 10–50 iterations on **CPU** and confirm the residual history matches your previous runs.
3. If you use the GPGPU solver (ROCm/HIP), run a small case there too and confirm both convergence and final field values match expectations.
4. Watch for unusual warnings, segfaults, or mid-run crashes.

In our testing the solver was unaffected, but you should re-verify after any STAR-CCM+ update.

## How to revert

Reverse each step in order:

```bash
# Revert Step 2: restore bundled libstdc++
cd <STAR_HOME>/star/lib/linux-x86_64-2.28/system/clang17.0-64/
sudo mv libstdc++.so.6.0.29.disabled libstdc++.so.6.0.29
sudo mv libstdc++.so.6.disabled-link libstdc++.so.6
sudo mv libstdc++.so.disabled-link   libstdc++.so

# Revert Step 1: remove the symlink (optional, harmless to leave in place)
sudo rm /usr/lib/dri

# Revert Step 3: remove the launcher
rm ~/starccm-fixed.sh
# or  rm ~/.local/bin/starccm-vis
```

After reverting Step 2, STAR-CCM+ goes back to its sealed bundled runtime and rendering returns to OpenSWR software fallback.

## Notes for other GPUs / versions

**Other RDNA3 cards (W7800, RX 7900 XTX/XT in workstation use):** Same problem, same fix. Only `gfx1100` strings differ; the `libstdc++` issue is identical.

**RDNA2 cards (W6800, V620):** Mesa 21.1.8 added gfx1030 support, so Failure 1 may not apply, but Failure 2 (libstdc++ ABI) can still bite depending on system Mesa/LLVM versions. Run `Test Graphics` first; if it already says `library: STAR Mesa OpenGL`, apply the fix.

**Older STAR-CCM+ versions (e.g. 18.x, 19.x):** The bundled Mesa is even older. Same approach works. The exact path to the bundled `libstdc++` may differ — search for `find <STAR_HOME> -name 'libstdc++.so.6.*' -not -name '*-gdb.py'` to locate it.

**Newer STAR-CCM+ versions (2511, 2602, etc.):** Try without the fix first. Siemens may have updated the bundled runtime. If the bundled `libstdc++` provides `GLIBCXX_3.4.30` or higher, only Step 1 (symlink) and possibly Step 3 (`-graphics native`) are needed. Check with:

```bash
strings <STAR_HOME>/star/lib/linux-x86_64-2.28/system/clang17.0-64/libstdc++.so.6 \
  | grep "^GLIBCXX_3" | sort -V | tail -3
```

**Ubuntu 24.04 / Debian 13:** `libLLVM` version may be 17 or 18, requiring an even newer `GLIBCXX`. Same diagnostic flow applies; the symbol name in the error message will tell you which version is needed.

**NVIDIA users:** This README does not apply. NVIDIA's proprietary OpenGL bypasses the Mesa DRI loader entirely and STAR-CCM+ generally finds it without help.

## Debugging guide for other users

If your symptoms differ from the ones described above, here is the diagnostic sequence we used:

**1. Check what STAR-CCM+ actually sees.** This is Siemens' own documented troubleshooting tool:

> Right-click `Scenes` → `Test Graphics`

The first three lines (`library`, `window`, `OpenGL vendor`) tell you exactly which renderer STAR-CCM+ ended up with.

**2. Check what the system can do, independent of STAR-CCM+:**

```bash
glxinfo | grep -E "direct rendering|OpenGL vendor|OpenGL renderer|OpenGL version"
```

If this shows your GPU correctly with `direct rendering: Yes`, the system is fine and the problem is inside STAR-CCM+. If this shows `llvmpipe` or fails, fix the system first (driver install, X11 session, etc.) — STAR-CCM+ cannot do better than the system.

**3. Get the real error message:**

```bash
LIBGL_DEBUG=verbose starccm+ -graphics native 2>&1 | tee /tmp/starccm-gl.log
grep -iE "MESA-LOADER|GLIBCXX|undefined symbol|version" /tmp/starccm-gl.log
```

The exact error in `MESA-LOADER: failed to open ...` lines tells you what to fix:

| Error contains | Likely cause | Fix |
|---|---|---|
| `cannot open shared object file: No such file or directory` | DRI driver path missing | Step 1 (symlink) |
| `GLIBCXX_X.Y.Z not found` | Bundled `libstdc++` too old | Step 2 (rename) |
| `GLIBC_X.Y not found` | Bundled `glibc` too old | More invasive; may need to upgrade STAR-CCM+ |
| `undefined symbol: ...` | ABI mismatch between bundled and system libraries | Identify which library is too old, rename that |

**4. Check the STAR-CCM+ wrapper script:**

```bash
which starccm+
cat $(readlink -f $(which starccm+)) | head -50
```

The wrapper script sources `starenv` which sets `LD_LIBRARY_PATH` to put bundled libraries before system ones. This is why `LD_PRELOAD` set externally usually doesn't survive — `reset_env` early in the wrapper clears it. If you need to inject something, you generally need to either modify `starenv`, rename the offending bundled library (the approach in this README), or write a launcher that exports variables `starenv` doesn't reset.

**5. If `Test Graphics` reports `vtkOpenGLRenderWindow context not initialized` instead of a Mesa loader error**, the DRI driver loaded but VTK couldn't create an OpenGL context. Possible causes:
- Wayland session (force X11: `XDG_SESSION_TYPE=x11` at login, or `GDK_BACKEND=x11 starccm+`)
- Missing X11 GLX visual at the depth STAR-CCM+ requested
- Conflicting `LD_LIBRARY_PATH` from another tool's environment

## Risk disclosure

**This is an unofficial workaround.** Siemens does not document or support modifying STAR-CCM+'s bundled libraries. By following this README you accept the following risks:

1. **Solver behavior changes.** Substituting a newer `libstdc++` is ABI-backward-compatible in theory, but corner cases in C++ exception handling, `std::string` SSO, locale facets, and similar low-level mechanics are not guaranteed identical. Always validate against known-good results before relying on this for new production work.

2. **Crashes after STAR-CCM+ updates.** Patch updates may restore the bundled `libstdc++` or change paths. Re-apply Step 2 after every update and re-validate.

3. **Loss of Siemens support.** If you contact Siemens technical support with a problem, you may need to revert the changes (see [How to revert](#how-to-revert)) before they can reproduce or assist. The revert is fast and clean.

4. **System updates.** Ubuntu updates that change the multiarch layout or `libstdc++` ABI (extremely rare within a major version) could affect this. The diagnostic flow above will identify any new error.

This setup has been used in our environment without issue for solver runs both on CPU and GPGPU (HIP/ROCm), but your mileage may vary.

## Acknowledgements

Diagnosis worked through end-to-end with Claude (Anthropic), using the Simcenter STAR-CCM+ 20.06.007 / 2510 User Guide as the primary reference. The diagnostic flow combined Siemens' documented `Test Graphics` tool with standard Linux OpenGL debugging (`LIBGL_DEBUG=verbose`, `glxinfo`, `ldd`, `strings`).

Special thanks to the AMD open-source graphics stack maintainers — `radeonsi` on RDNA3 is mature and the issue here was entirely in STAR-CCM+'s sealed bundled runtime, not in the driver itself.

---

## License

This README is released into the public domain (CC0). Use it however you want.

STAR-CCM+ is a commercial product of Siemens Digital Industries Software. This document is independent technical documentation for end users and is not endorsed by or affiliated with Siemens.
