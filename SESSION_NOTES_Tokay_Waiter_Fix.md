# Session Notes - Tokay Build Root Shell Fixing Waiter Error

Date: 2026-08-17
Target: `tokay-CP2A.260605.012` (Pixel 9 / tokay, Android 17, SDK 37)
Commit: "Tokay Build Root Shell Fixing Waiter Error"

## 1. Goal

Compile the exploit POC for the target `tokay-CP2A.260605.012` as the initial
baseline build, then fix the runtime linker error encountered on-device.

## 2. Build Setup

- NDK: `android-ndk-r29` (clang 21.0.0, `aarch64-linux-android35-clang`)
  located at `/home/imeiplus/android-ndk-cache/android-ndk-r29`
- Build command:

  ```sh
  cd exploit
  make PROJECT=tokay-CP2A.260605.012
  ```

- The Makefile selects target-specific sources via `pick_src`, which prefers
  `src/targets/$(PROJECT)/<file>` when present and falls back to `src/<file>`.
- The `tokay-CP2A.260605.012` target directory contains the full
  implementation; other device targets (`comet`, `caiman`, `komodo`, etc.)
  are one-line stubs that `#include` the tokay sources.

## 3. Initial Build

Compiled successfully on the first attempt.

Artifacts (before fix, sha256 of preload.so):

- `build/embed/su_daemon_aarch64_pie` - 14 KB PIE executable (embedded)
- `build/tokay-CP2A.260605.012/bin/preload.so` - 222 KB shared object
  - sha256: `2b7a08e02a5acaec1ca3d64419e93d6d114a5e0cbb4ec3f3c2c08a27224c48ea`

## 4. Runtime Error Observed

On-device, preloading the library into a process failed at link time:

```
CANNOT LINK EXECUTABLE "/system/bin/env": cannot locate symbol
"cleanup_main_waiter_pi_state" referenced by "/data/local/tmp/preload.so"...
1|lynx:/data/local/tmp $
```

## 5. Root Cause

`cleanup_main_waiter_pi_state` is declared in the tokay target's `common.h`
and referenced by the tokay target's `fops.c`, but its definition lives in
the tokay target's `root.c`.

The Makefile hardcoded the generic root source for the shared-object build:

```make
CORE_SRCS := \
  $(call pick_src,main.c) \
  $(call pick_src,util.c) \
  $(call pick_src,slide.c) \
  $(call pick_src,fops.c) \
  $(call pick_src,pipe.c) \
  src/root.c            # <-- hardcoded, ignored the target's root.c
```

Because the build linked `src/root.c` (generic) instead of the target's
`root.c`, the symbol was never defined in `preload.so`. The shared library
link tolerated the undefined symbol (allowed for `.so`), but the Android
dynamic linker rejected it at load time.

A `llvm-nm -u` audit of the broken `preload.so` confirmed exactly one
non-libc undefined symbol: `cleanup_main_waiter_pi_state`.

## 6. Fix

`exploit/Makefile`: switched `root.c` from a hardcoded source to `pick_src`
so the target's own `root.c` is used when present:

```diff
   $(call pick_src,slide.c) \
   $(call pick_src,fops.c) \
   $(call pick_src,pipe.c) \
-  src/root.c
+  $(call pick_src,root.c)
```

Fallback behavior is preserved:

- `tokay-CP2A.260605.012` -> `src/targets/tokay-CP2A.260605.012/root.c`
- Stub targets (e.g. `comet-...`) -> their one-line `root.c` which
  `#include`s the tokay `root.c`
- `blazer-CP2A.260605.012` (no target `root.c`) -> generic `src/root.c`

## 7. Verification

Rebuilt with `make clean && make PROJECT=tokay-CP2A.260605.012`:

- Build succeeds with no errors or warnings.
- `cleanup_main_waiter_pi_state` is now defined in the output:

  ```
  0000000000030e68 T cleanup_main_waiter_pi_state
  ```

- `llvm-nm -u` shows only standard libc / libdl / pthread / crt undefined
  symbols remain (all resolvable on Android 35).
- `make info` confirms source resolution for both target classes.

Artifacts (after fix):

- `build/tokay-CP2A.260605.012/bin/preload.so`
  - sha256: `31a03064dd20d9ee13d7864d846de7eafaa3a4d94899e9690d7b2144d3d73774`

## 8. Files Changed

- `exploit/Makefile` - use `pick_src` for `root.c` (the functional fix)
- `.gitignore` - ignore generated `exploit/build/` artifacts
- `SESSION_NOTES_Tokay_Waiter_Fix.md` - this document

## 9. Commands Used

```sh
# build
make PROJECT=tokay-CP2A.260605.012
make clean && make PROJECT=tokay-CP2A.260605.012

# verify symbols
llvm-nm -u build/tokay-CP2A.260605.012/bin/preload.so
llvm-nm build/tokay-CP2A.260605.012/bin/preload.so | grep cleanup_main_waiter_pi_state
```
