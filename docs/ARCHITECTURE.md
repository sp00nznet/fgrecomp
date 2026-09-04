# Architecture

## The shape of the target

`libfg.so` is a stock NDK C++ build: stripped, position-independent, ARM64,
with complete `.eh_frame` unwind tables. 45 MB on disk, 28.6 MB of `.text`,
12,381 symbols exported and 386 imported. It maps as two segments and applies
190,492 relocations, and it registers **1,202 static constructors** — the
engine does a great deal of work before `main` would ever have run.

Two numbers make this an unusually good target. `.eh_frame` recovers **85,702
functions covering 100.0% of `.text`**, leaving two undisassembled bytes in the
whole library; function-boundary recovery, normally the hardest problem in
static recompilation, is simply free here. And the lifter's special-case
counters are all zero: no syscall sites, no pointer authentication, no LSE
atomics. There is nothing in this binary the TSTO work has not already met.

The C++ runtime is statically linked. There is no `libc++_shared.so` beside the
engine and no NDK-mangled `_ZNSt6__ndk1...` import surface to answer — the
library has exactly one C++ ABI import. The four `libcrashlytics*.so` files
shipping next to it are Firebase telemetry, not engine, and are discarded.

## The engine is cocos2d-x 4.0

Griffin — the `com.tinyco.griffin` namespace — is a fork of cocos2d-x 4.0, and
the binary says so in a plain version string. This is the single most useful
fact about the port, because it converts the host contract from something
reverse-engineered into something *read*.

Every `Java_org_cocos2dx_lib_*` entry point in `contract/griffin.txt` has public
source on the other side of it. `Cocos2dxRenderer.java`, `Cocos2dxActivity.java`,
`Cocos2dxHelper.java` and `Cocos2dxBitmap.java` at the v4.0 tag give the exact
signature, units, thread and call order for each one. cocos2d-x 4.0 also ships
an official desktop backend, so the same engine's Windows/Linux/macOS behaviour
is available as a reference implementation rather than a guess.

Two things follow that the TSTO port never had to think about.

**The host owes text rendering.** On Android, cocos2d-x rasterizes labels with
`android.graphics` on the Java side and hands the pixels back down through
`nativeInitBitmapDC`. A desktop host has no Skia, so this becomes the host's
job. SDL2_ttf covers it, and androidrecomp already links SDL2.

**The contract generalizes.** What is being written here is a cocos2d-x host,
not a Family Guy host. It is reusable across a large slice of the mobile
catalogue, and the other two Griffin games inherit it whole.

## The host contract

Of 92 `Java_*` exports, 41 are the contract and all 41 resolve in the loaded
image. The other 51 are platform services — IronSource, Tapjoy, TapResearch,
AdColony, Facebook, Google Play, Sentry, push, IAP, video — which are stubbed
or answered by the server. They are not translated; the Java side that called
them has no desktop counterpart and is discarded along with the `classes*.dex`.

The contract splits four ways: boot and GL config, surface lifecycle and the
render tick, input, and the services the Android side used to *perform* that the
host must now perform itself (text rasterization, audio device info, IME).
`contract/griffin.txt` is grouped in that order and annotated.

## Where the port actually stands

`fg_host` loads, relocates and binds the real library today. With a live GL
context:

```
imports    386 total, 350 resolved, 36 outstanding
```

The 87 GL and GLES entry points cost no code at all — desktop GL exports them
under identical names, so they bind straight through the live driver the moment
a window exists. That is the single largest block of the import surface and it
is already answered.

The 36 that remain are the whole M3b work list:

| owed by | n | what it is |
|---|---|---|
| `libandroid.so` | 9 | the `AAsset*` family — how the engine reads content. Backs onto the extracted APK's `assets/` directory. |
| `libOpenSLES.so` | 8 | audio. Seven are `SL_IID_*` interface constants; the work is the object model behind `slCreateEngine`. Shared with tstorecomp. |
| `libc.so` | 14 | Linux-isms: an `epoll` + `eventfd` cluster, `setitimer`, `sigaltstack`, `prctl`, `statfs`, `memalign`, `arc4random`. |
| `libm.so` | 3 | `hypotf`, `ldexpf`, `frexpf`. |
| `libEGL.so` | 1 | `eglGetProcAddress` — forwards to `SDL_GL_GetProcAddress`. |
| `liblog.so` | 1 | `__android_log_assert`. |

Twenty-eight of those are an afternoon. The `AAsset*` nine are the interesting
ones — mapping them onto a directory is exactly what turns an APK-shaped game
into a desktop-shaped one. The OpenSLES eight are the only substantial piece,
and doing them once pays for two ports.

## Two execution paths, one host

The host program is required either way, so it is built first.

**Path A — native ARM64 host.** On Apple Silicon, Windows-on-ARM and ARM64
Linux the instructions in `libfg.so` already run. What it lacks is Bionic and
Android: the loader plus the shim is enough to call `nativeInit` and
`nativeRender` directly. This gets a real window early and validates the shim,
the asset paths and the GL usage before any lifting exists to be blamed.

**Path B — lifted.** Everywhere else, `tools/lifter.py` turns the 85,702
recovered functions into C, one function per unit, and the host links the
result. `fg_host` on x86-64 says so plainly today:

```
x86-64 host: image loaded and relocated but not executable here;
             running it needs a lifter.
```

Path A is the debugger for path B. Every bug found with a working ARM64 window
is a bug the lifter cannot be blamed for.
