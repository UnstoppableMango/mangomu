# Which DS core should v1 embed?

Research for [#2](https://github.com/UnstoppableMango/mangomu/issues/2), under the map in [#1](https://github.com/UnstoppableMango/mangomu/issues/1).
Researched 2026-10-22 against source code, read at these commits:

| Repo | Commit | Notes |
| --- | --- | --- |
| [melonDS-emu/melonDS](https://github.com/melonDS-emu/melonDS) | [`7d8dea4`](https://github.com/melonDS-emu/melonDS/tree/7d8dea42e221b13521270c423b61add41a741b97) (master, 2026-10-21) | Latest release is tag `1.1` (2025-11-18); every API cited below except the renderer split also exists at `1.1`. |
| [JesseTG/melonds-ds](https://github.com/JesseTG/melonds-ds) ("melonDS DS", libretro) | [`f394adb`](https://github.com/JesseTG/melonds-ds/tree/f394adbacb5722ee97c1b37c8064da9a25818310) (tag `v1.4.0`) | Builds against a melonDS fork, `JesseTG/melonDS@jtg/fix-uninitialized-opengl`. |
| [libretro/melonDS](https://github.com/libretro/melonDS) (legacy libretro core) | `66b5d26` | README still shows melonDS 0.9.2; superseded by melonDS DS. Not evaluated further. |
| [TASEmulators/desmume](https://github.com/TASEmulators/desmume) | [`1275dc6`](https://github.com/TASEmulators/desmume/tree/1275dc64f5d1f5ef18dc1bc1fa9e012024b6df1a) | Last release tag `release_0_9_13`. |
| [libretro/desmume](https://github.com/libretro/desmume) | [`95b4d79`](https://github.com/libretro/desmume/tree/95b4d798731caa809125b6c3c11d17cc332ff6ef) | The libretro DeSmuME core. |
| [libretro/libretro-common](https://github.com/libretro/libretro-common) | `a3194c1` | `include/libretro.h`. |
| [giongto35/cloud-game](https://github.com/giongto35/cloud-game) | `1fab9ef` | Prior art only. |

## Recommendation

**Embed melonDS's own core directly** (its `core` static library, C++ API), through a thin `extern "C"` shim called from Go with cgo.
Build it with the Qt/SDL frontend and the OpenGL renderer off, use the software renderer, and boot games with the built-in FreeBIOS and generated firmware (direct boot), so no BIOS or firmware dumps are needed.
Pin a release (`1.1`) or a specific master commit; the C++ API is not a stable ABI and changes between versions (the renderer API was reworked on master after `1.1`).

Keep **melonDS DS over libretro** as the fallback if writing the shim proves costly.
It is the same emulator underneath, but it composites both screens into one frame, owns its own mic and save plumbing through libretro, and would make mangomu a general libretro frontend, which v1 does not need.

**This rules out DeSmuME in both forms.**
The libretro DeSmuME core has no real microphone input, does not expose its save RAM to the frontend, and has no JIT on arm64.
The DeSmuME embedding API (`frontend/interface`) is a process-global singleton that sends audio straight to SDL with no way to pull samples, and has no microphone API.

It also rules out the legacy `libretro/melonDS` core, and it means mangomu's core seam (#6) is mangomu's own interface rather than libretro, since the shim is the seam.

## Comparison at a glance

| | melonDS core (direct) | melonDS DS (libretro) | DeSmuME (libretro) | DeSmuME `interface` (direct) |
| --- | --- | --- | --- | --- |
| License | GPL-3.0-or-later | GPL-3.0 | GPL-2.0-or-later | GPL-2.0-or-later |
| Go embedding | cgo + C shim over C++ `melonDS::NDS`; host implements 57 `Platform::` functions | cgo `dlopen` of a `.so` + libretro C API | same as melonDS DS | cgo over a C API (`desmume_*`) |
| Dumps needed | No (FreeBIOS + generated firmware) | No (same) | No (HLE BIOS, built-in firmware) | No (same) |
| Screens | Two separate 256×192 buffers | One composited frame, 17 layouts | One composited frame | One stacked 256×384 buffer |
| Pixel format | 32-bit BGRA | `XRGB8888` | `RGB565` or `XRGB8888` | RGB555 (16-bit), helper to RGBX |
| Audio | s16 stereo, any output rate (core resamples) | s16 stereo at ~32,728 Hz | s16 stereo at 44,100 Hz | Plays through SDL; no pull API |
| Touch | `TouchScreen(x, y)` in screen pixels | libretro pointer, in layout space | libretro pointer | `desmume_input_set_touch_pos` |
| Real mic input | Yes, pull callback, s16 mono at ~47.6 kHz | Yes, via libretro mic interface | No (noise button only) | No API |
| SRAM | Callback with full buffer and dirty range | `retro_get_memory_data(SAVE_RAM)`; frontend saves | Core writes its own `.dsv` file | Core writes its own `.dsv` file |
| Save states | In-memory `Savestate` buffer | `retro_serialize` | `retro_serialize` | File or slot only |
| Multiple instances per process | Yes (instance object + `userdata`) | No (libretro is one core per process) | No | No (globals) |
| JIT | x86-64 and arm64 | x86-64 and arm64 | x86-64 only (arm64 build disables it) | x86-64 |
| Needs a GPU | No | No (OpenGL is optional) | No | No (but initialises SDL video) |

## melonDS, embedded directly

### Embedding and license

melonDS's root `CMakeLists.txt` has `option(BUILD_QT_SDL "Build Qt/SDL frontend" ON)` and `option(ENABLE_OGLRENDERER "Enable OpenGL renderer" ON)`, and `src/CMakeLists.txt` builds the emulator as `add_library(core STATIC ...)`.
Configured with `-DBUILD_QT_SDL=OFF -DENABLE_OGLRENDERER=OFF -DENABLE_GDBSTUB=OFF`, the master commit above builds `libcore.a` (~9.7 MB) plus `libteakra.a` (the DSi DSP) on stock Linux with only a C++ compiler, CMake and Ninja; no Qt, SDL, OpenGL or zlib was needed.

The API is C++ (`melonDS::NDS` in `src/NDS.h`, constructed from an `NDSArgs` in `src/Args.h`), so cgo cannot call it directly; a small `extern "C"` shim is required.
The core also calls out to free functions in `melonDS::Platform` (`src/Platform.h`) that the frontend must define at link time.
`nm` on the built `libcore.a` shows 57 undefined `Platform::` symbols: file I/O, threads, mutexes and semaphores, logging, `WriteNDSSave`/`WriteGBASave`/`WriteFirmware`/`WriteDateTime`, local multiplayer (`MP_*`), networking (`Net_*`), camera, AAC (DSi), addon rumble/motion, and the mic.
Most are stubs or one-liners for v1, but they must all exist.
Every `Platform` callback takes the `void* userdata` passed to the `NDS` constructor, so state is per instance rather than global.

License: GPL-3.0-or-later (`LICENSE`, file headers).
FreeBIOS is BSD-2-Clause (`freebios/drastic_bios_readme.txt`, from DraStic).
mangomu itself is MIT, which is GPL-compatible, but a host binary or image that links melonDS is a combined GPLv3 work when distributed.

### Firmware and BIOS

`NDSArgs` defaults to FreeBIOS for both ARM9 and ARM7 (`FreeBIOSGetNtrArm9()`/`FreeBIOSGetNtrArm7()`) and to a generated firmware (`Args.h`).
`NDS::NeedsDirectBoot()` returns true when the firmware is not bootable (the generated one is not) or the BIOS is not a known native dump, so with the defaults every game is direct-booted, skipping the DS menu.
The README states firmware boot (not direct boot) needs a dump from a DS or DS Lite.
DSi mode needs real DSi BIOS and NAND (`DSiArgs` defaults to `BrokenBIOS`), so it is out of reach without dumps.
The generated firmware's user settings (name, language, birthday) come from the `Firmware` object the host constructs.

### Video

`NDS::RunFrame()` runs one frame.
melonDS DS uses 33,513,982 / 560,190 ≈ 59.8261 fps (`src/libretro/constants.hpp`), the DS master clock over cycles per frame.
The software renderer double-buffers: `SoftRenderer::GetFramebuffers(void** top, void** bottom)` (`src/GPU_Soft.cpp`) returns the front buffer of each screen separately.
`GPU.h` documents them as "32-bit BGRA, 256x192 for each screen".
Screen layout is therefore the host's job, which suits a browser that can place two screens itself.
On `1.1` the same buffers are reached through `GPU` (`Framebuffer[2][2]`); on master the renderer was split into `Renderer`/`SoftRenderer`, so the shim differs between the two.
The 3D software rasterizer can run on its own thread (`SoftRenderer3D::SetThreaded`, `GPU3D_Soft.h`).

### Audio

`SPU::ReadOutput(s16* data, int samples)` (`src/SPU.cpp`) returns interleaved stereo s16 frames from a ring buffer.
The output rate is a constructor argument (`SPU(..., double outputSampleRate)`) and the core resamples with blip-buf (`blip_set_rates(..., INTERNAL_SAMPLE_RATE * OutputSkew, OutputSampleRate)`), so the host can ask for 48 kHz directly and feed Opus without its own resampler.
`SetOutputSkew` lets the host nudge the rate to keep audio locked to video.

### Input

- Buttons: `NDS::SetKeyMask(u32 mask)`; the low 10 bits map to `KEYINPUT` and the next 2 (X, Y) to the ARM7-only bits, and the mask is active-low like the hardware register.
- Touch: `NDS::TouchScreen(u16 x, u16 y)` in bottom-screen pixels (0–255, 0–191), `NDS::ReleaseScreen()` to lift the stylus.
- Lid: `NDS::SetLidClosed(bool)`.
- Microphone: pull-based. The core calls `Platform::Mic_Start`/`Mic_Stop` when the game opens or stops the mic, and `Platform::Mic_ReadInput(s16* data, int maxlength, void* userdata)` to fill its buffer; `Platform.h` documents the format as "signed 16-bit mono at 47.6 KHz" (33,513,982 / 704 Hz per melonDS DS's `MIC_SAMPLE_RATE`). `Mic.cpp` stops the NDS mic if the game does not sample it for two frames. The host therefore has to resample browser mic audio (typically 48 kHz) to ~47.6 kHz and keep a small buffer ready.

### Saves

- SRAM: the host loads it with `NDS::SetNDSSave(data, len)` and reads it with `GetNDSSave()`/`GetNDSSaveLength()`. The core calls `Platform::WriteNDSSave(savedata, savelen, writeoffset, writelen, userdata)` with the whole buffer and the changed range. For ordinary SPI save chips this happens in `CartRetail::SPIRelease()`, when the game releases chip-select after a write command, so once per completed write sequence rather than per byte. The core never touches disk; when and where to persist is the host's choice. The Qt frontend debounces with a background `SaveManager` that copies to a secondary buffer and writes later (`src/frontend/qt_sdl/SaveManager.cpp`).
- **Loading a save state rewrites SRAM:** `CartRetail::DoSavestate` calls `WriteNDSSave(SRAM, len, 0, len)` after a load (`src/NDSCart/CartRetail.cpp`), so restoring a state overwrites the in-game save with the state's copy and fires the persistence callback.
- Save states: `NDS::DoSavestate(Savestate*)`, where `Savestate` can be a growable in-memory buffer (`Savestate(u32 initial_size)`, `Buffer()`, `Length()`) or wrap an existing one. No file is involved.

### Headless and CPU

No GPU is required: the software renderer is the default, and the core built above has no OpenGL at all.
The JIT is on by default where supported: `ENABLE_JIT` is a dependent option that is only available on x86-64 and ARM64 (`CMakeLists.txt`).
No first-party benchmark of CPU cost at 1× native resolution was found; melonDS publishes no system requirements.
A measurement was attempted but not completed: no freely licensed DS ROM could be downloaded from this environment.
The first prototype should measure it (frame time per `RunFrame`, with and without the threaded 3D renderer) on a thecluster node.

## melonDS DS (libretro)

### Embedding and license

A standard libretro core: a shared library the host `dlopen`s and drives through `libretro.h`'s C API, which cgo can call directly.
cloud-game does exactly this (`pkg/worker/caged/libretro/nanoarch/dl_unix.go`, `C.dlopen`), but cloud-game ships no DS core in its config, and its frontend implements no `RETRO_DEVICE_POINTER` or microphone interface, so its libretro code would not cover a DS core as is.
License: GPL-3.0.
`ENABLE_OPENGL` is a build option, and `src/libretro/render/software.hpp` provides a software render path.

### Firmware and BIOS

The README says melonDS "includes built-in BIOS and firmware replacements that work with most games", with optional dumps in the frontend's system directory.

### Video

`retro::set_pixel_format(RETRO_PIXEL_FORMAT_XRGB8888)` is required (`src/libretro/core/core.cpp`).
Both screens are composited into one frame using one of 17 layouts (`ScreenLayout` in `src/libretro/config/types.hpp`: top/bottom, left/right, hybrid, single-screen, rotated, and so on).
The host gets one image, not two.

### Audio

`SAMPLE_RATE = 33513982.0 / 1024.0` ≈ 32,728.5 Hz, pushed through `audio_sample_batch` as interleaved s16 stereo.
The host must resample to 48 kHz for Opus.

### Input

Buttons through the RetroPad, touch through `RETRO_DEVICE_POINTER` (`src/libretro/input/input.cpp`), in libretro's −0x7FFF..0x7FFF pointer space relative to the composited layout.
The mic uses `RETRO_ENVIRONMENT_GET_MICROPHONE_INTERFACE` (`src/libretro/environment.cpp`), which `libretro.h` marks experimental; the frontend must implement it and resamples to whatever rate the core asks for.

### Saves

NDS SRAM is exposed through `retro_get_memory_data(RETRO_MEMORY_SAVE_RAM)`; `CoreState::WriteNdsSave` (`src/libretro/sram.cpp`) notes "retro_get_memory lets us delegate autosave to the frontend", so the host decides when to flush.
GBA SRAM and firmware are written by the core itself on a timer, through libretro's task queue and VFS.
Save states through `retro_serialize`.

## DeSmuME (libretro)

### Embedding and license

Same libretro mechanics as above.
License: GPL-2.0-or-later.

### Firmware and BIOS

External BIOS and firmware are optional (`desmume_use_external_bios`, looking for `bios7.bin`, `bios9.bin`, `firmware.bin` in the system directory); without them DeSmuME HLEs the BIOS SWI calls (`NDSSystem.cpp`) and uses built-in firmware settings.

### Video

`RGB565` by default, `XRGB8888` as an option; screens composited into one frame.
`timing.fps = 59.8261`.

### Audio

`timing.sample_rate = 44100.0` (`DESMUME_SAMPLE_RATE` in `SPU.h`).

### Input

RetroPad buttons, touch through `RETRO_DEVICE_POINTER`, lid on L2.
**There is no real microphone input.**
The core requests no libretro mic interface, and the mic is a button ("Make Microphone Noise") that injects generated noise.
The `physical` mic mode reads `Mic_DefaultBufferRead()`, a buffer that only the noise generator writes in this build.

### Saves

`retro_get_memory_data` returns only `RETRO_MEMORY_SYSTEM_RAM`; save RAM is **not** exposed.
The core writes the save itself to a `.dsv` file under its battery path (`BackupDevice` in `mc.cpp`), byte by byte through `EMUFILE` as the game writes.
Save states through `retro_serialize`.

### Headless and CPU

Software rasterizer by default; no GPU needed.
`Makefile.libretro` sets `DESMUME_JIT = 0` and `DESMUME_JIT_ARM = 0` when building on aarch64, so arm64 runs the interpreter.

## DeSmuME, embedded directly

`desmume/src/frontend/interface/interface.h` exposes a C API (`desmume_init`, `desmume_open`, `desmume_cycle`, `desmume_input_keypad_update`, `desmume_input_set_touch_pos`, `desmume_savestate_save`, `desmume_backup_import_file`, and so on), which cgo could call without a shim.
But:

- State is process-global (`MMU`, `GPU`, `CommonSettings`), so one emulator per process.
- `desmume_init` hard-wires the SDL sound core (`SPU_ChangeSoundCore(SNDCORE_SDL, ...)`) and calls `SDL_Init(SDL_INIT_VIDEO | SDL_INIT_TIMER)`; there is no call to pull audio samples.
- `desmume_draw_raw()` returns both screens stacked as 16-bit RGB555, with `desmume_draw_raw_as_rgbx` as a converting helper.
- There is no microphone API.
- Saves go to `.dsv` files; save states go to files or slots.

## What later tickets depend on

- **#6 host/core seam:** the seam is a Go interface over a C shim, not libretro. The shim exposes, per instance: run one frame, two framebuffers (BGRA 256×192 each), pull N stereo s16 samples at a rate chosen at construction, set key mask, touch/release, lid, and the callbacks for mic pull, SRAM write and mic start/stop. melonDS's per-instance `userdata` means nothing forces one emulator per process, so the seam can stay neutral between pod-per-session and in-process sessions. A future home-grown core implements the same Go interface. cgo plus C++ means the host image needs a C++ toolchain at build time and cannot be a pure-Go static build.
- **#10 storage/saves:** SRAM persistence is entirely the host's job: load bytes with `SetNDSSave` at start, receive `WriteNDSSave(full buffer, offset, len)` after each completed save write, and persist on the host's own schedule (debounce, plus a flush on shutdown/SIGTERM). Save states are an in-memory blob, so they are cheap to add later. **Loading a save state overwrites SRAM** through that same callback, which a save-state feature must account for (for example by keeping SRAM and states separate and warning, or snapshotting SRAM first).
- **#11 input:** buttons are a 12-bit mask; touch is absolute bottom-screen pixels with an explicit release, so the browser must map pointer events on the bottom-screen element to 0–255 × 0–191. The mic is pull-based at ~47.6 kHz s16 mono and only active while the game asks for it (`Mic_Start`/`Mic_Stop`), so browser mic capture can be opened on demand. Lid close is available if wanted.
- **Media pipeline:** two separate screens means the host chooses the composition (one stacked 256×384 video track is the simplest, and the browser can split it). Audio can be produced at 48 kHz directly for Opus. The frame rate is ~59.83 fps, not 60, so pacing must follow the core's frames rather than a 60 Hz timer, or let audio drive timing via `SetOutputSkew`.
- **Images/charts:** melonDS is GPL-3.0; a published image containing it carries GPL source obligations. No BIOS, firmware or ROMs need to be shipped or mounted for v1, only the ROM itself and a save path.

## Open questions

- CPU cost per frame at 1× is unmeasured; measure it on a thecluster node in the first prototype, and check whether nodes are x86-64 or arm64 (both have a melonDS JIT).
- Pin `1.1` or a master commit? Master has the reworked renderer API; `1.1` is the last release.
- Is a GPLv3 combined binary/image acceptable for mangomu's published artifacts? (The source can stay MIT.)
