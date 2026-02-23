# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the NVIDIA OptiX 9.0.0 SDK — a header-only ray tracing API backed by the NVIDIA display driver, plus sample applications demonstrating the API. The repository tracks multiple OptiX SDK versions as separate git commits (7.2 through 9.0), with the `v9.0.0_custom_gl` branch containing a local patch to use the system OpenGL instead of the bundled GLAD loader.

## Build

The samples are built with CMake out-of-source. The build directory must be outside `SDK/`.

```bash
mkdir build && cd build
cmake ../SDK -DOptiX_INSTALL_DIR=../ -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
# Executables land in build/bin/
```

Key CMake options:
- `OptiX_INSTALL_DIR` — path to the root of this repo (where `include/` lives); defaults to `../` relative to `SDK/`
- `SAMPLES_INPUT_GENERATE_OPTIXIR=ON/OFF` — generate OptiX-IR (preferred with CUDA ≥ 11.7) vs PTX
- `CUDA_NVRTC_ENABLED=ON` — compile device code at runtime with NVRTC instead of NVCC at build time
- `OPTIX_DEBUG_DEVICE_CODE=ON` — build device code with debug flags
- `CMAKE_BUILD_TYPE=Debug` — enables `-G -O0` for NVCC

To build a single sample:
```bash
make -j$(nproc) optixHello
```

## Repository Layout

```
include/          OptiX public headers (optix.h, optix_host.h, optix_device.h, etc.)
SDK/
  CMakeLists.txt  Top-level build; registers all samples and the sutil/support libs
  CMake/          Custom CMake modules (FindOptiX, FindCUDA, Macros, ptx2cpp, etc.)
  cuda/           Shared device-side CUDA headers used across multiple samples
  sutil/          Sample utility library (sutil_7_sdk): camera, GLDisplay, scene loader,
                  CUDAOutputBuffer, trackball, PPM I/O, math helpers
  support/        Third-party libs built as part of the SDK: imgui, miniz, zeromq, glad
  optix*/         One directory per sample application
doc/              PDF programming guide and API reference for OptiX 9.0.0
```

## Architecture

### Host/Device split
OptiX programs span two compilation domains:
- **Host** (`*.cpp`): sets up the OptiX context, builds acceleration structures (BVH), compiles modules from PTX/OptiX-IR, links pipeline programs groups (raygen, miss, hit, callable), allocates CUDA buffers, and launches `optixLaunch`.
- **Device** (`*.cu`): implements the shader programs — raygen, closest-hit, any-hit, miss, callable — compiled by NVCC/NVRTC to PTX or OptiX-IR.

Each sample follows the pattern `optixFoo.cpp` (host) + `optixFoo.cu` (device) + `optixFoo.h` (shared params struct passed via SBT/launch params).

### sutil library
`sutil_7_sdk` is the shared utility library linked by every sample. Key classes:
- `CUDAOutputBuffer<T>` — manages the render target with GL interop modes (GL_INTEROP, ZERO_COPY, CUDA_P2P)
- `GLDisplay` — displays a CUDA buffer via OpenGL
- `Scene` — glTF/OBJ scene loader
- `Camera` / `Trackball` — interactive camera

### SBT (Shader Binding Table)
The `sutil::Record<T>` template wraps the OptiX SBT record header + user data. Samples allocate device memory for the SBT and call `optixSbtRecordPackHeader` before launch.

### CMake macros
`OPTIX_add_sample_executable(name ...)` in the root `CMakeLists.txt` is the macro used by every sample. It handles separating OBJ-targeted CUDA sources from OptiX-IR/PTX sources, compiling them, and linking against `sutil_7_sdk`, GLFW, and imgui.

## Local Customization

The `v9.0.0_custom_gl` branch patches `SDK/sutil/Exception.h` to use `<GL/gl.h>` (system OpenGL) instead of `<glad/glad.h>`:
```cpp
//#include <glad/glad.h>
#include <GL/gl.h> // Manually patched to use openGL from system
```
This is intentional — preserve this when merging or rebasing.

## Key Headers

- `include/optix.h` — main entry point; dispatches to `optix_host.h` or `optix_device.h`
- `include/optix_stubs.h` — dynamic loading stubs for the OptiX function table (used by host code via `optixInit()`)
- `include/optix_types.h` — all OptiX types and enums
- `SDK/sutil/Exception.h` — `OPTIX_CHECK`, `CUDA_CHECK`, `GL_CHECK` macros used throughout samples
