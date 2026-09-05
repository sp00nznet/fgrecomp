# fgrecomp

> *Family Guy: The Quest for Stuff* as a native cross-platform desktop
> application. Bring your own APK.

**Status: the engine is lifted and its static initialisation runs.** `fg_host`
maps `libfg.so`, applies its 190,492 relocations and binds its imports --
**350 of 386 resolve** with a live GL context, and all 41 host-contract entry
points are present in the image. The lifter covers **99.99% of instructions and
99.7% of functions**, and `arc_boot` runs **1,201 of the engine's 1,202 static
constructors**. No JNI bridge yet. See [Milestones](#milestones).

The first lift of this engine needed *no* changes to the toolkit and completed
99.0% of functions, which is the strongest evidence so far that androidrecomp
is a kit rather than one game's scaffolding. Everything since has been general
work that flows back to the sibling ports, not Family Guy work -- and it did:
[tstorecomp](https://github.com/sp00nznet/tstorecomp) gained 27 constructors
from fixes found here.

---

## What this is

FGQFS is not a Java game. The renderer, the simulation and the whole isometric
Quahog live in a single 45 MB ARM64 shared object, `libfg.so`. The Java side is
a window, a GL context and a touch handler.

So this is a port, not an emulator: replace the Android host with a native
desktop host, satisfy the library's POSIX/OpenGL import surface, and lift the
ARM64 code to C for machines that are not ARM.

Sibling of [tstorecomp](https://github.com/sp00nznet/tstorecomp), built on the
same [androidrecomp](https://github.com/sp00nznet/androidrecomp) kit.

## Why this target

It is a materially easier port than TSTO, and the triage says so:

| | libscorpio (TSTO) | **libfg (this)** |
|---|---|---|
| `.eh_frame` coverage of `.text` | 98.8% | **100.0%** |
| undisassembled bytes | 24,916 | **2** |
| undefined symbols | 481 | **386** |
| linked libraries | 13 | **10, all stock Android** |
| ships `libc++_shared.so` | yes | **no — statically linked** |
| proprietary middleware | `libNimble.so` | **none** |
| syscall sites | 115 | **0** |
| pointer-auth | 166 | **0** |
| LSE atomics | 42 | **0** |

Function-boundary recovery — the hardest problem in static recompilation — is
completely free here: 100.0% coverage with two undisassembled bytes across
28.6 MB of `.text`. There are no lifter special-cases beyond the ones TSTO
already forced. The C++ runtime is statically linked, so the NDK-mangled
`_ZNSt6__ndk1...` symbols that make `libc++_shared.so` awkward simply do not
appear: the library has exactly **one** C++ ABI import.

Of the 386 undefined symbols, most are ground androidrecomp already covers:

```
plain libc/libm  240   host C runtime, resolved by name, no code
GL/EGL            88   desktop GL exports these names, no code
pthread           25   shim_pthread
android/NDK       13   real work
OpenSLES           8   real work -- the same audio gap TSTO owes
bionic-specific    6   small
dl                 5   shim_sys
C++ ABI            1   trivial
```

Full numbers in [`docs/triage-libfg-arm64.md`](docs/triage-libfg-arm64.md).

## The engine is cocos2d-x 4.0

The single most useful fact about this target. Griffin — TinyCo's engine, the
`com.tinyco.griffin` namespace — is a fork of **cocos2d-x 4.0**, and the binary
says so in plain text.

That means the host contract is not reverse-engineered, it is *read*. Every
`Java_org_cocos2dx_lib_*` entry point in
[`contract/griffin.txt`](contract/griffin.txt) has public source on the other
side of it: `Cocos2dxRenderer.java`, `Cocos2dxActivity.java`,
`Cocos2dxHelper.java`, `Cocos2dxBitmap.java` at the v4.0 tag say exactly what
each call is handed, in what units, and on which thread. cocos2d-x 4.0 also
ships an official desktop backend, so there is a reference implementation of
the same engine's Windows/Linux/macOS behaviour sitting in the same tree.

Two consequences worth planning around:

**The host owes text rendering.** On Android, cocos2d-x rasterizes labels with
`android.graphics` on the Java side and hands the pixels back through
`nativeInitBitmapDC`. There is no Skia in a desktop host, so this is the host's
job. SDL2_ttf covers it, and androidrecomp already links SDL2.

**The contract generalizes.** A cocos2d-x host is not a Family-Guy host. It is
reusable across a very large slice of the mobile catalogue, and the two other
Griffin games get it for free.

## Three games, one contract

`libfg.so` and *Futurama: Worlds of Tomorrow*'s `libclient.so` are the same
engine — identical JNI shape, 29 `com_tinyco_griffin` entry points each,
matching `SGNMobile_dispatchPlatformEvent` and `PlatformIncentVideo_*`
clusters. *Marvel Avengers Academy* is the same studio and near-certainly the
same engine.

One Griffin host contract covers all three. The catch is architectural, not
technical: Futurama's final build is `armeabi-v7a` only — no arm64 was ever
shipped — so it waits on an ARM32 lifter. That is a pattern, not bad luck.
Google's 64-bit mandate only ever applied to apps still shipping updates, so
the dead and delisted catalogue is frozen at 32-bit.

## Legal / content policy

Tools only. No game code, no game assets, no extracted sprites, no save data,
no TinyCo or Jam City binaries — `.gitignore` blocks all of it, deliberately.
You supply your own legally obtained APK; everything here operates on a file
you already have. Licensed MIT; contributions must be your own work.

## Building

```sh
git clone --recursive https://github.com/sp00nznet/fgrecomp
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build

./build/fg_host --contract=contract/griffin.txt path/to/libfg.so
./build/fg_host --window --contract=contract/griffin.txt path/to/libfg.so
```

On Windows add `-DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake`
so CMake finds zlib and SDL2. A toolchain file only takes effect on a fresh
cache, so delete `build/` if you add it later.

Point it at the `lib/arm64-v8a/` directory of an extracted APK — the loader
uses the libraries sitting next to the engine to satisfy its imports.

Re-run the triage yourself with:

```sh
python androidrecomp/tools/apk_probe.py yourgame.apk --out docs/triage.md
```

## This repo is the thin half

The loader, the Bionic/Android shim layer, the host window and the triage tools
are not specific to this game and live in **androidrecomp**, vendored here as a
submodule. Nothing title-specific belongs there; nothing reusable belongs here.

```
fgrecomp/
├── androidrecomp/          # submodule -- loader, shims, window, tools
├── contract/griffin.txt    # the JNI entry points the host drives
├── docs/
│   ├── ARCHITECTURE.md
│   └── triage-libfg-arm64.md
└── CMakeLists.txt
```

## Milestones

- [x] **M0 — triage.** `libfg.so` probed: 85,702 functions, 100.0% `.eh_frame`
      coverage, 386 undefined symbols, no lifter special-cases.
- [x] **M1 — contract.** 92 JNI exports enumerated and split: 39 the host must
      drive, 53 platform services to stub. Engine identified as cocos2d-x 4.0.
- [x] **M2 — host builds.** `fg_host` compiles against androidrecomp with zlib
      and SDL2.
- [x] **M3a — image loads.** 45.5 MB mapped, 190,492 relocations applied, 1,202
      constructors found, all 41 contract entry points resolved. 350/386 imports
      bound with a window; the 87 GL entry points cost no code.
- [ ] **M3b — empty work list.** 35 imports outstanding: 9 `AAsset*`,
      8 OpenSLES, 14 Linux-isms in libc, 3 libm, `__android_log_assert`.
      `eglGetProcAddress` is done — and it mattered more than one symbol,
      because the guest branches *through* an unbound slot rather than simply
      missing the function.
- [ ] **M4 — JNI bridge.** The contract implemented against the cocos2d-x 4.0
      Java sources; window comes up on an arm64 host.
- [ ] **M5 — text and audio.** `nativeInitBitmapDC` on SDL2_ttf; the OpenSLES
      eight on a desktop backend, shared with tstorecomp.
- [ ] **M6 — server.** The game talks to something. EA is not coming back and
      neither is Jam City's backend.
- [x] **M7 — lifter.** ARM64 → C for hosts that are not ARM. **99.99% of
      instructions and 99.7% of functions**, 92,081 functions across 96
      translation units. `arc_boot` runs 1,201 of the 1,202 static
      constructors; the two that do not are a single unlifted indirect target
      apiece, reached from data rather than from any call site.
