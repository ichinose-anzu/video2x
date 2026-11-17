# P1 Analysis — video2x macOS arm64 port (read-only recon)

Date: 2026-11-17. Author: opencode (mimo-v2.6-flash-free), P1 only, no source edits.
Inputs: the 28 extracted upstream files under `./recon/` (from `ichinose-anzu/video2x` @ `7db9c18`,
branch `macos-arm64`). `RECON_BUNDLE.md` was IGNORED per instruction (truncated/broken).

Codespace reachability check (one command, executed):

```
$ gh cs ssh -c video2x-macos-arm64-wvvqx6w95v67h5xx4 -- 'cd /workspaces/video2x && git status --short --branch'
## macos-arm64...origin/macos-arm64
```

Real output: single line, branch `macos-arm64`, tracking `origin/macos-arm64`, no dirty files. Codespace REACHABLE.

Note: file paths below are the upstream repo paths (recon/ is an extraction of the real tree).
Quotes are exact from our files.

---

## 1. Evidence list — macOS/arm64-relevant platform assumptions in OUR files

### 1.1 CMakeLists.txt (root)

| File:line | Exact quote | Why it matters on macOS arm64 |
|---|---|---|
| CMakeLists.txt:44 | `add_link_options(-Wl,-s -flto -Wl,--gc-sections)` | GNU-ld flags inside `elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")` — Apple's ld64/lld do not accept `--gc-sections` (macOS uses `-dead_strip`); will fail or warn at link on macOS. `-flto` also requires matching LLVM bitcode across all libs. |
| CMakeLists.txt:22 | `option(VIDEO2X_USE_EXTERNAL_NCNN "Use the system-provided ncnn library" ON)` | Default ON → `find_package(ncnn REQUIRED)` (line 167). This is THE option for Homebrew arm64 ncnn; exact name `VIDEO2X_USE_EXTERNAL_NCNN`. |
| CMakeLists.txt:23 | `option(VIDEO2X_USE_EXTERNAL_SPDLOG "Use the system-provided spdlog library" ON)` | External spdlog default ON; exact name `VIDEO2X_USE_EXTERNAL_SPDLOG`. |
| CMakeLists.txt:24 | `option(VIDEO2X_USE_EXTERNAL_BOOST "Use the system-provided Boost library" ON)` | External Boost default ON; exact name `VIDEO2X_USE_EXTERNAL_BOOST`. |
| CMakeLists.txt:26-28 | `option(VIDEO2X_ENABLE_NATIVE ...)`, `option(VIDEO2X_ENABLE_X86_64_V4 ...)`, `option(VIDEO2X_ENABLE_X86_64_V3 ...)` | x86-specific arch options; `-march=x86-64-v4` (line 59) / `-march=x86-64-v3` (line 65) are invalid on arm64 — defaults OFF, must stay OFF. `-march=native` (line 53) is accepted by Apple arm64 clang but is set ON by `.justfile` unix build (see below). |
| CMakeLists.txt:121 | `$<$<NOT:$<PLATFORM_ID:Windows>>:-fPIC>` | Non-Windows branch applies to macOS; `-fPIC` is a no-op/accepted on Darwin, harmless. |
| CMakeLists.txt:127-136 | `if(WIN32)` … `set(ffmpeg_base_path "${PROJECT_SOURCE_DIR}/third_party/ffmpeg-shared")` … `"${ffmpeg_base_path}/lib/avcodec.lib"` | Prebuilt FFmpeg is WIN32-only. macOS takes the `else()` branch (line 137+): `find_package(PkgConfig REQUIRED)` + `pkg_check_modules(... REQUIRED libavcodec/libavfilter/libavformat/libavutil/libswscale)` (lines 139-146) → needs Homebrew ffmpeg + pkg-config. |
| CMakeLists.txt:151 | `${libavfilter_LIBRARIES}` (inside `target_include_directories`) | Pre-existing upstream bug (libraries var used as include dir); not macOS-specific, noted only — do not "fix" in P2 unless it blocks us. |
| CMakeLists.txt:166-167 | `if(VIDEO2X_USE_EXTERNAL_NCNN)` / `find_package(ncnn REQUIRED)` | System (Homebrew) ncnn path — the macOS path of record. |
| CMakeLists.txt:171 | `set(ncnn_base_path "${PROJECT_SOURCE_DIR}/third_party/ncnn-shared/x64")` | Prebuilt **x86** ncnn reference — WIN32-only branch (line 169), never taken on macOS. |
| CMakeLists.txt:185-186 | `option(NCNN_VULKAN "" ON)` / `option(NCNN_VULKAN_ONLINE_SPIRV "" ON)` | Only used when vendored ncnn submodule is built (`add_subdirectory(third_party/ncnn)`, line 292) — relevant if external ncnn proves insufficient (Vulkan-on-moltenVK capable ncnn needed either way). |
| CMakeLists.txt:311 | `option(USE_SYSTEM_NCNN "" ${VIDEO2X_USE_EXTERNAL_NCNN})` | Pass-through option handed to the three `third_party/lib*-ncnn-vulkan` subprojects (lines 312-314). Exact name `USE_SYSTEM_NCNN`; it mirrors `VIDEO2X_USE_EXTERNAL_NCNN`, so configuring with `VIDEO2X_USE_EXTERNAL_NCNN=ON` also makes the model libs use system ncnn. |
| CMakeLists.txt:331-337 | `install(TARGETS libvideo2x … LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" …)` | Install rules use GNUInstallDirs; **no rpath / install_name handling anywhere** — a dylib built against `/opt/homebrew/lib/libncnn.dylib` would embed an absolute Homebrew path unless we add `@rpath` handling (CONTROL rule: never absolute `/opt/homebrew/...`). |
| CMakeLists.txt:372-376 | `install(DIRECTORY "${CMAKE_SOURCE_DIR}/models" DESTINATION "${CMAKE_INSTALL_DATADIR}/video2x")` | Non-Windows installs models to `<prefix>/share/video2x`; lookup happens via `fsutils::find_resource` (see fsutils.cpp evidence) — no macOS bundle location considered. |
| CMakeLists.txt:380-394 | `file(GLOB ffmpeg_dlls ...)`, `boost_program_options-vc143-mt-x64-1_86.dll` | Windows-only x86 DLL references (guarded `if(WIN32)`), not taken on macOS. |
| CMakeLists.txt:454-455 | `find_package(Vulkan REQUIRED)` / `target_link_libraries(video2x PRIVATE Vulkan::Vulkan)` | CLI links Vulkan — on macOS requires MoltenVK + Vulkan loader (Homebrew `molten-vk`/`vulkan-loader` or Khronos VulkanSDK, `VULKAN_SDK` set). |
| CMakeLists.txt:459 | `find_package(Boost REQUIRED COMPONENTS program_options)` | External Boost path; Homebrew `boost` provides it. |
| CMakeLists.txt:1 | `cmake_minimum_required(VERSION 3.10 FATAL_ERROR)` | Works with modern CMake; CMP0167 guard at lines 9-11 already anticipates CMake ≥3.30. |
| CMakeLists.txt (whole file) | *(no occurrence of `OpenMP`, `omp`, `find_package(OpenMP)`)* | **OpenMP is never requested by the top-level build** — it enters implicitly through the ncnn submodule / `lib*-ncnn-vulkan` subprojects. On Linux CI it comes from apt `libomp-dev`. CONTROL's double-libomp trap therefore applies to whatever those subprojects (or a Homebrew ncnn) pull in — must be handled in P2 by explicit `OpenMP_ROOT` / `OpenMP_libomp_LIBRARY` if the build shows two libomps. Unverified: third_party CMakeLists are NOT in our recon set. |

### 1.2 src/

| File:line | Exact quote | Why it matters |
|---|---|---|
| src/fsutils.cpp:46 | `std::filesystem::path filepath = std::filesystem::read_symlink("/proc/self/exe", ec);` | **Linux-only** (`/proc` does not exist on macOS). This sits in the `#else // _WIN32` branch (line 43), so macOS compiles THIS branch → `get_executable_directory()` always fails at runtime on macOS (lines 48-51 log the error and return empty) → `find_resource` candidate 5 (line 94) is dead on macOS. Needs a guarded `__APPLE__` branch using `_NSGetExecutablePath` (P3). |
| src/fsutils.cpp:80-90 | `if (const char* appdir = std::getenv("APPDIR")) … "/usr/local/share/video2x" … "/usr/share/video2x"` | Resource candidates are Linux FHS + AppImage only; no `@executable_path/../Resources` / bundle candidate. |
| src/fsutils.cpp:94 | `candidates.push_back(get_executable_directory() / resource);` | The only portable fallback — depends on the broken `/proc/self/exe` code above. |
| src/libvideo2x.cpp:31 | `[[gnu::target_clones("arch=x86-64-v4", "arch=x86-64-v3", "default")]]` | x86-only attribute on `VideoProcessor::process`; clang on arm64 warns (attribute ignored). Per CONTROL: guard/conditionalize only if it actually breaks the build (P3, conditional). |
| src/filter_realesrgan.cpp:90 | `uint32_t heap_budget = ncnn::get_gpu_device(gpuid_)->get_heap_budget();` | Assumes a working ncnn Vulkan GPU device (MoltenVK must report a nonzero heap budget or tilesize collapses to 32). Runtime concern for P5, not a compile issue. |
| src/filter_realcugan.cpp:120 | `uint32_t heap_budget = ncnn::get_gpu_device(gpuid_)->get_heap_budget();` | Same as above. |
| src/filter_realesrgan.cpp:52-53 / filter_realcugan.cpp:63-66 / interpolator_rife.cpp:38-42 | `model_param_path = std::filesystem::path(STR("models")) / STR("realesrgan") / …` resolved via `fsutils::find_resource(...)` | Model lookup is CWD-independent but funnels through the Linux-only `get_executable_directory()`; must work for CLI built in `build/`. |
| src/logger_manager.cpp:49 | `std::make_shared<spdlog::sinks::stdout_color_sink_mt>()` | Portable; no platform assumption. |
| src/processor_factory.cpp | (no platform code) | Portable. |

### 1.3 include/

| File:line | Exact quote | Why it matters |
|---|---|---|
| include/libvideo2x/libvideo2x_export.h:9-10 | `#else` / `#define LIBVIDEO2X_API` | Non-Windows = no visibility attribute → default visibility exports symbols from the dylib. Sufficient for CLI↔dylib linking on macOS as long as we do NOT add `-fvisibility=hidden`. |
| include/libvideo2x/fsutils.h:10-16 | `#ifdef _WIN32 … typedef char CharType; #define STR(x) x` | macOS takes the narrow-char path (same as Linux) — portable. |
| include/libvideo2x/libvideo2x.h:7-10 | `extern "C" { #include <libavcodec/avcodec.h> … }` | Needs FFmpeg headers from pkg-config (Homebrew). |
| include/libvideo2x/filter_realesrgan.h:8 / filter_realcugan.h:8 / interpolator_rife.h:8 | `#include "realesrgan.h"` / `"realcugan.h"` / `"rife.h"` | Come from `third_party/lib*-ncnn-vulkan/src` include dirs (CMakeLists.txt:108-110) → submodules MUST be initialized (`--recursive`) even when using external ncnn. |

### 1.4 tools/video2x

| File:line | Exact quote | Why it matters |
|---|---|---|
| tools/video2x/src/vulkan_utils.cpp:10-13 | `VkInstanceCreateInfo create_info{}; create_info.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO; VkResult result = vkCreateInstance(&create_info, nullptr, instance);` | **No extensions, no `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR`** — MoltenVK is a portability driver; without `VK_KHR_portability_enumeration` (+ `VK_KHR_get_physical_device_properties2`) the loader may hide the Apple GPU → `device_count == 0` error at lines 22-28. P3 candidate. |
| tools/video2x/src/vulkan_utils.cpp:60-76 | `switch (device_properties.deviceType) { … default: std::cout << "Unknown"; }` | Device-type printing is generic; MoltenVK reports Integrated GPU — no fix needed here. |
| tools/video2x/src/vulkan_utils.cpp:21-22 | `if (result != VK_SUCCESS || device_count == 0)` | The observable failure mode if portability enumeration is missing on macOS. |
| tools/video2x/src/argparse.cpp:70 | `("version,V", "Print program version and exit")` | `--version` / `-V` implemented at lines 231-234: prints `Video2X version <…>` and returns 1; `main` returns 0 for `parse_ret > 0` (video2x.cpp:80-82) → exit code 0. Verification command safe. |
| tools/video2x/src/argparse.cpp:76 | `("list-devices,l", "List the available Vulkan devices (GPUs)")` | Implemented lines 236-241 → returns 1 → main returns 0 on success, -1 on failure. Exit code is our pass/fail signal; output text comes from vulkan_utils.cpp. |
| tools/video2x/src/argparse.cpp:493-502 | `get_vulkan_device_prop(arguments.vk_device_index, &dev_props)` … `return -1` if it fails | **Every real run validates a Vulkan device up front** — on macOS, if MoltenVK enumeration fails, even `video2x -i … -o …` dies before decoding. So `--list-devices` success is a hard prerequisite for P5. |
| tools/video2x/src/video2x.cpp:7-9 | `#include <fcntl.h> <termios.h> <unistd.h>` | POSIX path (non-Win32) — valid on macOS. |
| tools/video2x/src/video2x.cpp:43 | `int wmain(int argc, wchar_t* argv[])` | Windows-only entry; macOS uses `main` (line 54). Portable. |
| tools/video2x/include/vulkan_utils.h:3 | `#include <vulkan/vulkan.h>` | Requires Vulkan headers (brew `vulkan-loader`/VulkanSDK). |

### 1.5 Packaging

| File:line | Exact quote | Why it matters |
|---|---|---|
| packaging/debian/control.ubuntu2404:5 | `Architecture: amd64` | Debian package is x86_64-only; irrelevant to macOS (no brew formula being added in P1-P5). |
| packaging/debian/control.ubuntu2404:7 | `Depends: ffmpeg (>=6.0), libvulkan1 (>=1.3.0.0), libboost-program-options1.83.0 (>=1.83.0)` | Documents the runtime dep set the project expects (FFmpeg, Vulkan loader, Boost.PO) — macOS equivalents: brew ffmpeg, vulkan-loader+molten-vk, boost. |
| packaging/arch/PKGBUILD:5 | `arch=('x86_64')` | Explicitly x86_64 packaging. |
| packaging/arch/PKGBUILD:8 | `depends=('ffmpeg' 'ncnn' 'openmp' 'vulkan-driver' 'spdlog' 'boost-libs')` | Confirms upstream treats **OpenMP as an external runtime dep** even though top-level CMake never finds it — macOS needs a single, unambiguous `libomp.dylib` (CONTROL's double-load trap). |
| packaging/arch/PKGBUILD:22 | `-DCMAKE_CXX_COMPILER=clang++ -DVIDEO2X_ENABLE_X86_64_V3=ON` | Example build enabling x86-64-v3 — must NOT be replicated on arm64. |

### 1.6 .github/workflows

| File:line | Exact quote | Why it matters |
|---|---|---|
| build.yml:3-20 | `on: push: … pull_request: … workflow_dispatch:` | Upstream triggers on push/PR — **fork rule 5 forbids that for OUR new workflow** (dispatch-only). We add a separate workflow; do not modify these. |
| build.yml:24 | `runs-on: ubuntu-latest` | Only ubuntu/windows jobs exist — **no macOS runner anywhere in our workflows**; P4 must add one. |
| build.yml:37-47 | `sudo apt-get install -y … libvulkan-dev glslang-tools libomp-dev libboost-program-options-dev` | Linux dep list — the macOS analog is brew: `ffmpeg ncnn spdlog boost glslang molten-vk vulkan-loader libomp` (names to be confirmed at P5). `libomp-dev` here is the only OpenMP install step in CI. |
| build.yml:54 | `-DVIDEO2X_USE_EXTERNAL_NCNN=OFF -DVIDEO2X_USE_EXTERNAL_SPDLOG=OFF` | CI builds vendored ncnn/spdlog on Linux; mac workflow will use `ON` (external) per CONTROL. |
| build.yml:89-95 | `"https://github.com/GyanD/codexffmpeg/releases/download/$ffmpegVersion/ffmpeg-$ffmpegVersion-full_build-shared.zip"` and `"ncnn-$ncnnVersion-windows-vs2022-shared.zip"` | **Prebuilt x86 binaries** (Windows x64 FFmpeg 7.1, ncnn 20241226) — not usable on arm64; must not be referenced from macOS jobs. |
| build.yml:165 | `linuxdeploy-x86_64.AppImage` | x86_64-only AppImage tooling — not for macOS. |
| release.yml:3-6 | `on: push: tags: - "*"` | Upstream tag-triggered release; fork keeps it untouched (rule 5 applies to the NEW workflow only). |
| release.yml:14 | `if: github.event.base_ref == 'refs/heads/master'` | Releases tied to upstream master — another reason our workflow must be standalone dispatch. |

### 1.7 .justfile

| File:line | Exact quote | Why it matters |
|---|---|---|
| .justfile:5-7 | `bindir := "build"` / `generator := "Ninja"` / `cxx := "clang++"` | Default unix build: Ninja + `clang++`. On macOS, per CONTROL's OpenMP trap, `clang++` should resolve to Homebrew LLVM (`$(brew --prefix llvm)/bin/clang++`) if OpenMP flags become necessary, else Apple clang may suffice. |
| .justfile:16-21 | `cmake -G 'Ninja' … -DVIDEO2X_ENABLE_NATIVE=ON` | The `[unix] build` recipe is the closest starting point for a macOS recipe; `-march=native` is accepted on arm64. |
| .justfile:77-128 | `apt-get update` / `apt-get install -y … libomp-dev …` (debian/ubuntu recipes) | All dep-install recipes are apt/Linux-only — **no brew recipe exists**; P2 may add a `[unix] macos` recipe (optional; workflow can call cmake directly). |
| .justfile:124 | `-DCMAKE_INSTALL_PREFIX=video2x-linux-ubuntu-amd64/usr` | Linux amd64 artifact naming. |
| .justfile:219 | `LD_LIBRARY_PATH={{bindir}} {{bindir}}/video2x …` | Test recipes use `LD_LIBRARY_PATH` (Linux); macOS needs `DYLD_LIBRARY_PATH` or proper `@rpath` — evidence that rpath handling must be part of P2. |
| .justfile:196 | `linuxdeploy-x86_64.AppImage` download | x86_64-only, ignored for macOS. |
| .justfile:97-100 | `-DINSTALL_BIN_DESTINATION=. -DINSTALL_INCLUDE_DESTINATION=include …` | Stale variables (not present in our CMakeLists.txt; it uses GNUInstallDirs) — pre-existing upstream drift, not macOS-specific; do not rely on them. |

### 1.8 Docs

| File:line | Exact quote | Why it matters |
|---|---|---|
| docs/book/src/building/windows.md:51 | `cmake -S . -B build -DUSE_SYSTEM_NCNN=OFF -DUSE_SYSTEM_SPDLOG=OFF -DUSE_SYSTEM_BOOST=OFF …` | **Stale option names** — the real options are `VIDEO2X_USE_EXTERNAL_*` (CMakeLists.txt:22-24). Only `USE_SYSTEM_NCNN` still exists, as the pass-through at CMakeLists.txt:311. Do not copy the docs' names into our workflow. |
| docs/book/src/building/linux.md:1 | `# Linux` | Only Linux/Windows build docs exist; **no macOS doc** → P6 scope. |
| docs/book/src/building/linux.md:25-26 | `pacman -Sy ffmpeg ncnn vulkan-driver spdlog boost-libs` / `… openmp boost` | Documents system-dep story: system ncnn + system OpenMP + vulkan driver — the exact model we follow with Homebrew. |
| docs/book/src/developing/architecture.md | (no platform content) | No macOS/arm64 assumptions. |
| .gitmodules:1-18 | `third_party/ncnn` (Tencent/ncnn), `spdlog`, `boost`, `librealesrgan_ncnn_vulkan`, `librealcugan_ncnn_vulkan`, `librife_ncnn_vulkan` | Six submodules; the three `lib*` ones are ALWAYS needed (CMakeLists.txt:312-314) even with external ncnn/spdlog/boost → checkout must be `submodules: recursive`. ncnn/spdlog/boost submodules are only *compiled* when the corresponding `VIDEO2X_USE_EXTERNAL_*=OFF`. |

---

## 2. Dependency / build story AS WRITTEN IN OUR FILES

### 2.1 Exact CMake option names (authoritative = CMakeLists.txt)

- `VIDEO2X_USE_EXTERNAL_NCNN` (default **ON**) → `find_package(ncnn REQUIRED)` (CMakeLists.txt:22, 166-167)
- `VIDEO2X_USE_EXTERNAL_SPDLOG` (default **ON**) → `find_package(spdlog REQUIRED)` (CMakeLists.txt:23, 298-299)
- `VIDEO2X_USE_EXTERNAL_BOOST` (default **ON**) → `find_package(Boost REQUIRED COMPONENTS program_options)` (CMakeLists.txt:24, 458-459)
- `USE_SYSTEM_NCNN` — mirrors `VIDEO2X_USE_EXTERNAL_NCNN`, forwarded into `third_party/lib*-ncnn-vulkan` (CMakeLists.txt:311)
- `VIDEO2X_BUILD_CLI` (default ON, CMakeLists.txt:20), `BUILD_SHARED_LIBS` (default ON, :19)
- `VIDEO2X_ENABLE_NATIVE` / `VIDEO2X_ENABLE_X86_64_V4` / `VIDEO2X_ENABLE_X86_64_V3` (all default OFF, :26-28)
- Vendored-ncnn-only sub-options: `NCNN_VULKAN`, `NCNN_VULKAN_ONLINE_SPIRV`, `NCNN_BUILD_SHARED_LIBS`, `NCNN_DISABLE_RTTI`, … (CMakeLists.txt:179-290)
- NOT real (stale doc names): `USE_SYSTEM_SPDLOG`, `USE_SYSTEM_BOOST` (windows.md:51) — do not use.

### 2.2 Which ncnn?

- Default = **system/external** ncnn via `find_package(ncnn)` (CMakeLists.txt:167) → Homebrew arm64 `ncnn` is the intended macOS provider.
- Vendored alternative = `add_subdirectory(third_party/ncnn)` submodule (CMakeLists.txt:292), options at :179-290.
- Windows prebuilt x86: `third_party/ncnn-shared/x64` (CMakeLists.txt:171), downloaded in CI as `ncnn-20241226-windows-vs2022-shared.zip` (build.yml:93) / `20240820` (release.yml:95, windows.md:35).
- The three model libs (`librealesrgan/realcugan/rife_ncnn_vulkan`) are submodules linked into libvideo2x (CMakeLists.txt:312-314, 324-328) and receive `USE_SYSTEM_NCNN` (CMakeLists.txt:311).

### 2.3 Which FFmpeg?

- Windows: vendored prebuilt `third_party/ffmpeg-shared` (CMakeLists.txt:128-136), GyanD codexffmpeg 7.1 x64 zip (build.yml:84-91).
- Linux/macOS (the `else()` branch): **pkg-config system FFmpeg** — `pkg_check_modules(... REQUIRED libavcodec libavfilter libavformat libavutil libswscale)` (CMakeLists.txt:139-146; same for CLI at :437-441). On macOS = Homebrew `ffmpeg`.
- Runtime dep documented as `ffmpeg (>=6.0)` (packaging/debian/control.ubuntu2404:7).

### 2.4 OpenMP

- **Top-level CMakeLists.txt never mentions OpenMP.** It enters only via ncnn / `lib*-ncnn-vulkan` subprojects (files NOT in our recon — unverified).
- External evidence in OUR files: apt `libomp-dev` in every Linux CI/just recipe (build.yml:46, .justfile:89/117/147/174), Arch `openmp` dep (PKGBUILD:8), Linux doc `pacman -Sy … openmp` (linux.md:26).
- CONTROL.md:44-50 records the double-libomp crash (`__kmp_suspend_64`) and the correct shape (Homebrew LLVM, `-Xclang -fopenmp`, `OpenMP_ROOT`, explicit `OpenMP_libomp_LIBRARY`). Our files contain no counter-evidence, so that guidance stands for P2.

### 2.5 Vulkan

- CLI: `find_package(Vulkan REQUIRED)` + `Vulkan::Vulkan` (CMakeLists.txt:454-455).
- Linux CI: `libvulkan-dev` + `glslang-tools` (build.yml:44-45); Windows: Khronos Vulkan SDK 1.4.304.1 (build.yml:75-78).
- macOS: no provision anywhere in our files → MoltenVK + loader required (Homebrew `molten-vk`, `vulkan-loader`, `glslang`, or Khronos SDK).

### 2.6 Prebuilt x86 binary references (all locations)

1. CMakeLists.txt:128-136 — `third_party/ffmpeg-shared` + `avcodec.lib` … (WIN32-guarded)
2. CMakeLists.txt:171-177 — `third_party/ncnn-shared/x64`, `ncnn.dll`, `ncnn.lib` (WIN32-guarded)
3. CMakeLists.txt:488,490 — `boost_program_options-vc143-mt-x64-1_86.dll` (WIN32-guarded)
4. build.yml:89-95 — GyanD FFmpeg 7.1 + ncnn 20241226 **Windows x64** zips
5. release.yml:99-105 — same zips (ncnn 20240820)
6. build.yml:165 — `linuxdeploy-x86_64.AppImage`
7. windows.md:38-45 — manual Windows x64 downloads
8. PKGBUILD:5,22 — `arch=('x86_64')` + `-DVIDEO2X_ENABLE_X86_64_V3=ON`
9. debian control:5 — `Architecture: amd64`

All x86 references are either WIN32-guarded, Linux-package-only, or docs. **No macOS/arm64 artifact exists anywhere.**

---

## 3. Minimal proposed change list for P2 / P3 / P4

(P-sizes per CONTROL.md:69-71. Every item tied to evidence above; WHY = what breaks otherwise.)

### P2 — Build-system adaptation (CMake / just)

1. **Neutralize GNU link flags on Darwin** — CMakeLists.txt:44 `add_link_options(-Wl,-s -flto -Wl,--gc-sections)`.
   WHY: `--gc-sections`/`-s` are GNU-ld syntax; Apple ld errors on unknown options (or at minimum the LTO+Homebrew-dylib mix is fragile). Minimal change: guard the GNU set to `if(UNIX AND NOT APPLE)` and, if desired, substitute `-Wl,-dead_strip` on Apple.
2. **Explicit macOS dependency contract in configure step (mostly no code change)** — keep `VIDEO2X_USE_EXTERNAL_NCNN/SPDLOG/BOOST=ON` (their defaults; CMakeLists.txt:22-24) and provide brew equivalents: `ffmpeg` (pkg-config branch :139-146), `ncnn` (:167), `spdlog` (:299), `boost` (:459), `molten-vk`+`vulkan-loader`+`glslang` (:454).
   WHY: it is the only combination that avoids the x86 prebuilt/vendored paths and the huge vendored-ncnn build on a 2-vCPU-class runner budget; the option names in windows.md:51 are stale and must not be used.
3. **OpenMP single-libomp configuration** (conditional) — top-level CMake has no OpenMP logic; ncnn/libomp comes from brew. Apply CONTROL's shape only if the first macOS build links or loads two libomps: `-DOpenMP_ROOT=$(brew --prefix libomp)` and `-DOpenMP_libomp_LIBRARY=$(brew --prefix libomp)/lib/libomp.dylib`, compiler = Homebrew LLVM where needed.
   WHY: dual LLVM-libomp + brew-libomp ⇒ `__kmp_suspend_64` crash during inference (CONTROL.md:46-50); but per minimal-change rule, don't add flags the build doesn't demand.
4. **rpath / install_name handling for `libvideo2x.dylib` and `video2x`** — install rules at CMakeLists.txt:331-337 and :478 have no rpath logic; test recipes rely on `LD_LIBRARY_PATH` (.justfile:219).
   WHY: dylib must find Homebrew ncnn/FFmpeg at runtime yet never hardcode `/opt/homebrew/...` into install names (CONTROL.md:57) and the CLI must locate `libvideo2x.dylib` next to itself. Minimal: `CMAKE_INSTALL_RPATH` = `@executable_path/../lib` + `@loader_path` (and `INSTALL_RPATH_USE_LINK_PATH TRUE` for local runs).
5. **(Optional) Add a `[unix] macos` just recipe** — .justfile:77-128 are apt-only; :219 uses `LD_LIBRARY_PATH`.
   WHY: gives devs a reproducible local build; workflow itself will call cmake directly so this is non-blocking. Keep `VIDEO2X_ENABLE_NATIVE` OFF or ON deliberately; never enable `VIDEO2X_ENABLE_X86_64_V3/V4` (CMakeLists.txt:55-66).
6. **Do NOT touch**: WIN32 blocks (:127-136, :169-177, :379-394, :481-494), packaging/debian, packaging/arch, existing build.yml/release.yml, stale `-DINSTALL_*` vars (.justfile:97-100), `libavfilter_LIBRARIES` bug (CMakeLists.txt:151). All are pre-existing, platform-unrelated, or out of P2 scope.

### P3 — Source fixes (only if P1 evidence requires — it does, for 1-3)

1. **`src/fsutils.cpp:43-54` — add a macOS branch for executable-directory discovery.**
   WHY: `read_symlink("/proc/self/exe")` (line 46) does not exist on macOS; without it `find_resource`'s exe-relative candidate (line 94) never resolves → model files not found → all ncnn processors fail on first run. Fix shape: `#elif defined(__APPLE__)` using `_NSGetExecutablePath` + `realpath`, keeping the Linux branch byte-identical.
2. **`tools/video2x/src/vulkan_utils.cpp:8-17` — enable portability enumeration when available.**
   WHY: `VkInstanceCreateInfo` with zero extensions/flags (lines 10-13) can leave MoltenVK's Apple GPU unenumerated → `device_count == 0` (lines 21-28) → `--list-devices` fails and argparse aborts every run at lines 493-502. Minimal change: request `VK_KHR_portability_enumeration` + `VK_KHR_get_physical_device_properties2` (guarded so Linux behavior is unchanged when the loader doesn't expose them) and set `VK_INSTANCE_CREATE_ENUMERATE_PORTABILITY_BIT_KHR`.
3. **`src/libvideo2x.cpp:31` — guard `[[gnu::target_clones(...)]]`.**
   WHY: x86-only attribute; clang arm64 warns (non-blocking per CONTROL). Change ONLY if P5 logs show it as an error (e.g. `-Werror`); then guard with `#if defined(__x86_64__)`.
4. **Bundle resource candidates (`fsutils.cpp:78-91`) — NOT in P3.**
   WHY: milestone order (CONTROL.md:60-61) is CLI-first; the exe-relative candidate (line 94) covers `build/` and install-prefix layouts. Defer `.app` `../Resources` lookup to P6.
5. **No export-visibility change** — `libvideo2x_export.h:10` already exports everything on non-Windows; touching it would risk a broken dylib with zero benefit.

### P4 — New GitHub Actions workflow (one new file)

1. **Create `.github/workflows/macos-arm64.yml` with `on: workflow_dispatch:` ONLY** (no push/PR triggers).
   WHY: CONTROL rule 5 (fork CI discipline); upstream build.yml's push/PR triggers (build.yml:3-20) must stay untouched.
2. **`runs-on: macos-latest`** (Apple arm64).
   WHY: Codespaces are Linux x86_64 and cannot compile arm64 macOS (CONTROL.md:10-11); no macOS runner exists in our workflows (build.yml:24, release.yml build jobs).
3. **`actions/checkout` with `submodules: recursive` + `git submodule update --init --recursive`.**
   WHY: the three `lib*-ncnn-vulkan` submodules are unconditionally compiled and their headers are on our include path (CMakeLists.txt:108-110, 312-314; .gitmodules:10-18).
4. **Brew dependency step** (ffmpeg, ncnn, spdlog, boost, glslang, molten-vk, vulkan-loader, pkg-config, ninja, cmake; libomp/llvm only if step P2-3 proves necessary) and export `VULKAN_SDK`/PATH as needed.
   WHY: mirrors the Linux dep list (build.yml:37-47) with macOS equivalents; no x86 prebuilts may be downloaded (evidence §2.6).
5. **Configure with the exact external options + Ninja**, build, then run `--version` and `--list-devices` as smoke tests, then upload artifacts.
   WHY: CLI + dylib milestone before any GUI (CONTROL.md:60-61); `--list-devices` exercises the MoltenVK/`find_package(Vulkan)` path (CMakeLists.txt:454) end-to-end.

No changes to `src/` or `include/` are part of P2/P4; P3 changes are limited to the three items above.

---

## 4. Build verification plan — macOS arm64 runner (manual `workflow_dispatch`)

### 4.1 Workflow sketch (P4 deliverable; single job `build-macos-arm64`)

```yaml
on:
  workflow_dispatch:        # ONLY trigger — CONTROL rule 5
jobs:
  build-macos-arm64:
    runs-on: macos-latest   # Apple arm64
    steps:
      - uses: actions/checkout@v6
        with: { submodules: recursive }
      - run: git submodule update --init --recursive
      - run: brew install cmake ninja pkg-config ffmpeg ncnn spdlog boost glslang molten-vk vulkan-loader
      # conditional, only if P2-3 required: brew install libomp llvm
```

### 4.2 Configure / build commands

```bash
cmake -G Ninja -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DVIDEO2X_USE_EXTERNAL_NCNN=ON \
  -DVIDEO2X_USE_EXTERNAL_SPDLOG=ON \
  -DVIDEO2X_USE_EXTERNAL_BOOST=ON \
  -DVIDEO2X_ENABLE_X86_64_V3=OFF \
  -DVIDEO2X_ENABLE_X86_64_V4=OFF \
  -DCMAKE_INSTALL_PREFIX="$PWD/build/video2x-install"
cmake --build build --config Release --parallel --target install
```

(If P2-3 needed, append: `-DOpenMP_ROOT="$(brew --prefix libomp)" -DOpenMP_libomp_LIBRARY="$(brew --prefix libomp)/lib/libomp.dylib"` and set `CMAKE_CXX_COMPILER` to Homebrew LLVM clang++.)

### 4.3 Smoke tests (record real stdout in the job log)

```bash
./build/video2x --version        # expect: "Video2X version 6.4.0", exit 0 (argparse.cpp:231-234 + video2x.cpp:80-82)
./build/video2x --list-devices   # expect: ≥1 device (MoltenVK Apple GPU), exit 0 (vulkan_utils.cpp:41-97)
```

Failure interpretations (pre-registered):
- `--version` fails → link/dylib loading problem (rpath/ld flags → P2).
- `--list-devices` prints 0 devices / "Failed to enumerate" → missing portability enumeration (P3 vulkan_utils.cpp) or missing MoltenVK.
- dlopen/dyld error for libncnn/libomp → rpath (P2-4) or double-libomp (P2-3).

### 4.4 Artifacts to capture (upload-artifact)

1. `build/video2x` — the CLI executable (also from `build/video2x-install/bin/video2x` after `--target install`).
2. `build/libvideo2x.dylib` (target has `PREFIX "" OUTPUT_NAME libvideo2x`, CMakeLists.txt:97-101 → `libvideo2x.dylib` on macOS) plus `video2x-install/lib/`.
3. `video2x-install/` tree overall (bin, lib, include, `share/video2x/models` — models per CMakeLists.txt:375).
4. Job log excerpts: full configure output (proves which ncnn/FFmpeg/OpenMP were found), `--version` stdout, `--list-devices` stdout.
5. Diagnostics (echoed into log, not secrets): `otool -L build/video2x build/libvideo2x.dylib` (must show `@rpath/@loader_path`, never bare `/opt/homebrew/...` as an install name — CONTROL.md:57), `uname -m` (expect `arm64`), `brew --prefix ncnn`, `cmake --version`.

Pass criteria for P5: all four of CLI present, dylib present, `--version` exit 0, `--list-devices` exit 0 with ≥1 device.

---

## 5. Unverified / out-of-scope items (recorded for honesty)

- `third_party/*` submodule sources are NOT in the recon set (ncnn, spdlog, boost, librealesrgan/realcugan/rife) — their CMakeLists (incl. any OpenMP logic, `USE_SYSTEM_NCNN` consumption) are unverified.
- Root `cmake/` (`version.h.in`, `Video2XConfig.cmake.in`) not in recon set — referenced at CMakeLists.txt:70-74, :359-363.
- 6 of the 13 libvideo2x sources listed at CMakeLists.txt:77-91 are absent from recon: `avutils.cpp, conversions.cpp, decoder.cpp, encoder.cpp, filter_libplacebo.cpp, libplacebo.cpp` — not inspected for platform assumptions.
- Tools files `argparse.h, validators.cpp/h, newline_safe_sink.cpp, timer.cpp/h` (CMakeLists.txt:392-394) not in recon set.
- `packaging/docker/Dockerfile`, `packaging/appimage/*`, `models/`, `README.md`, `data/` not in recon set.
- No Homebrew package-name verification performed (P1 is local read-only); brew names in §4 are proposed, to be confirmed by an actual runner attempt in P5.
- No code was compiled; no inference was run. Codespace status output is the only external command result recorded (§ top).
