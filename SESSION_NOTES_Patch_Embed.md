# Session Notes - Embed Patch File (DSN/PSN) and Run After Root

Date: 2026-08-18
Device tested on: Pixel 7a (`husky`) and tokay-family targets (Android 17 / SDK 35)
Commit: "Adding Patch file to change DSN and PSN Directly After Root"

## 1. Goal

Add the payload `patch` file (a statically-linked ARM64 Go executable that sets
DSN / PSN and saves the Devinfo image) into the exploit `preload.so` so that:

- The exploit still roots the device exactly as before (POC behaviour untouched).
- After root is established, the embedded patch is extracted and executed as
  root automatically.
- The patch output is captured and shown in the terminal.

## 2. Files Added

- `exploit/src/patch.c`
  - Decompresses the embedded gzip payload (zlib `inflate`, gzip window bits).
  - Writes it to `/data/local/tmp/patch` (mode 0755).
  - Runs it as root via the already-installed `su` client:
    `su -c "cd /data/local/tmp && ./patch > /data/local/tmp/patch.log 2>&1"`.
  - Waits for the su client, then reads `/data/local/tmp/patch.log` and prints
    it to the terminal (waits for the log to stabilise first, because the patch
    writes some output asynchronously).
  - Registers a constructor (`patch_init`) that runs after the exploit
    constructor (`load`) because of link order, so root is already established
    (`root_child_done == 1`) when it executes.
- `exploit/src/patch_blob.S`
  - `.incbin "patch-build/embed/patch.gz"` (the gzip-compressed patch).

## 3. Files Changed

- `exploit/Makefile`
  - New `PATCH` variable (`PATCH=1`) and `patch` convenience target.
  - Patch builds output to `patch-build/$(PROJECT)/bin/` instead of
    `build/$(PROJECT)/bin/`.
  - Adds `src/patch.c` + `src/patch_blob.S` to the sources and `-lz` to the
    link flags only when `PATCH=1`.
  - Gzip rule: `gzip -9 -n -c ../patch > patch-build/embed/patch.gz`.
  - `clean` now removes both `build/` and `patch-build/`.
  - `info` reports `PATCH`, `PATCH_GZ`, `PATCH_LDFLAGS`.
- `.gitignore`
  - Added `exploit/patch-build/` (generated patch-build artifacts).

## 4. Build Commands

```sh
cd exploit

# normal build (unchanged behaviour, byte-identical output)
make PROJECT=tokay-CP2A.260605.012

# patch-enabled build (preload.so with embedded patch, in patch-build/)
make PROJECT=tokay-CP2A.260605.012 PATCH=1
# or the convenience target:
make PROJECT=tokay-CP2A.260605.012 patch

# push to device
adb push patch-build/tokay-CP2A.260605.012/bin/preload.so /data/local/tmp/preload.so
```

## 5. Runtime Flow

The shared object's `.init_array` order (verified with `llvm-readelf`):

1. `init_have_lse_atomics`
2. `__init_cpu_features`
3. `load`       - preload.c constructor, runs `run_exploit()` (roots the device)
4. `patch_init` - patch.c constructor, runs AFTER the exploit

`patch_init`:
1. Skips unless `root_child_done` is set (root was actually obtained).
2. Skips unless `/data/local/tmp/su` exists (su client installed).
3. Decompresses `embedded_patch_*` -> `/data/local/tmp/patch` (0755).
4. Forks `/data/local/tmp/su -c "cd /data/local/tmp && ./patch > /data/local/tmp/patch.log 2>&1"`.
5. Waits for the patch, then prints `/data/local/tmp/patch.log` to stdout.

The patch runs with root privileges through the su daemon that the exploit
already installs. `patch_ok` (written by the patch itself) confirms the patch
executed successfully.

## 6. Verification

- Normal build `preload.so` sha256 unchanged:
  `31a03064dd20d9ee13d7864d846de7eafaa3a4d94899e9690d7b2144d3d73774`.
- Patch build `preload.so` size ~943 KB (normal is ~222 KB).
- Embedded compressed patch `patch-build/embed/patch.gz`: 718,417 bytes
  (original `patch` is 1,769,634 bytes, ~40% of original with `gzip -9`).
- `gunzip -c patch.gz | cmp - patch` -> byte-identical round trip.
- `llvm-nm -u` shows only standard libc / libz / pthread symbols
  (new: `inflate`, `inflateEnd`, `inflateInit2_` from libz.so).
- On-device (Pixel 7a / husky): root shell obtained, `patch` extracted and run,
  `/data/local/tmp/patch_ok` created containing `done`, and full patch output
  saved in `/data/local/tmp/patch.log`.

## 7. Notes / Current Behaviour

- The patch's stdout/stderr is redirected to `/data/local/tmp/patch.log` by the
  root shell, then read back and printed. On-device, the log reliably contains
  the output (DSN / PSN / "Devinfo image saved successfully.") and can be viewed
  with `cat /data/local/tmp/patch.log`.
- The patch writes some of its output asynchronously, so `patch.c` waits for
  the log size to stabilise before printing; the log file remains as a durable
  record regardless of terminal behaviour.
- Expected terminal output looks like:

  ```
  DSN set to: 0KZ0FIX7KRZBDY
  PSN set to: QUN8C6ZM43PABGJEB5ZOCIRR
  Devinfo image saved successfully.
  ```

- The patch binary at the repository root (`patch`) must exist for `PATCH=1`
  builds; the Makefile errors if it is missing.

## 8. Commands Used

```sh
# build normal + patch variants
make PROJECT=tokay-CP2A.260605.012
make PROJECT=tokay-CP2A.260605.012 PATCH=1

# verify embedded symbols and undefined symbols
llvm-nm patch-build/tokay-CP2A.260605.012/bin/preload.so | grep -E "embedded_patch|patch_init"
llvm-nm -u patch-build/tokay-CP2A.260605.012/bin/preload.so

# verify compressed blob round-trip
gunzip -c patch-build/embed/patch.gz | cmp - ../patch
```
