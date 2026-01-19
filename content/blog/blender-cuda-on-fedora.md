+++
title = "Blender CUDA on Fedora: Why It Breaks and How to Fix It"
date = 2026-01-19
taxonomies.categories = ["3D"]
draft = false
+++

## Blender CUDA on Fedora: Why It Breaks and How to Fix It

Blender’s Cycles renderer uses CUDA for GPU rendering on NVIDIA cards. If Blender can’t find a matching precompiled kernel, it compiles one locally using `nvcc`.

On Fedora 42, this can fail even when CUDA is installed correctly.

### What goes wrong

The failure is caused by a mismatch between:

* Fedora’s `glibc` headers
* NVIDIA CUDA’s header declarations

In newer `glibc` versions, math functions like:

```
sinpi, cospi, sinpif, cospif
```

are declared with `noexcept(true)` (Don't ask - I don't know lol).

CUDA’s `math_functions.h` declares the same functions **without** `noexcept`.
C++ treats this as a hard error, so `nvcc` fails during compilation and Blender reports a generic “failed to execute compilation command”.

This is not a Blender bug and not a user configuration error. CUDA simply hasn’t caught up with current `glibc`.

### The fix

Patch CUDA’s `math_functions.h` so the function declarations match `glibc`.

This only changes exception specifications. Runtime behavior is unchanged.

### Patch script

Save the following as, for example, `fix-cuda-glibc.sh`:

```bash
#!/usr/bin/env bash
set -e

CUDA_MATH=/usr/local/cuda/targets/x86_64-linux/include/crt/math_functions.h

# Backup
sudo cp -a "$CUDA_MATH" "${CUDA_MATH}.bak.$(date +%Y%m%d_%H%M%S)"

# Patch declarations to match glibc
sudo perl -pi -e 's/\bsinpi\(double x\);/sinpi(double x) noexcept(true);/g' "$CUDA_MATH"
sudo perl -pi -e 's/\bsinpif\(float x\);/sinpif(float x) noexcept(true);/g' "$CUDA_MATH"
sudo perl -pi -e 's/\bcospi\(double x\);/cospi(double x) noexcept(true);/g' "$CUDA_MATH"
sudo perl -pi -e 's/\bcospif\(float x\);/cospif(float x) noexcept(true);/g' "$CUDA_MATH"

echo "CUDA math_functions.h patched for glibc compatibility."
```

Then run:

```bash
chmod +x fix-cuda-glibc.sh
./fix-cuda-glibc.sh
```

After patching, clear Blender’s kernel cache and start Blender again:

```bash
rm -rf ~/.cache/cycles/kernels
blender
```

### Notes

* The patch survives reboots.
* It may be overwritten by a CUDA update.
* If CUDA is updated and the problem returns, re-run the script.

### tl;dr

This issue exists because Fedora updates `glibc` faster than CUDA updates its headers. When Blender compiles CUDA kernels locally, the mismatch becomes visible.

Until NVIDIA updates CUDA to match newer `glibc` versions, this small header patch is the practical fix.
