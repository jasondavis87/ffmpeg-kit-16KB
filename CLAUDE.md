# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a fork of `arthenica/ffmpeg-kit` whose **only** stated goal is making the Android build compatible with API 35's 16KB page-size requirement. Upstream is retired (no further releases). Do not propose adding new external libraries, new platforms, or refactors unrelated to the 16KB compatibility goal.

The 16KB workaround relies on specific NDK builds (r23 / r25) sourced from `ci.android.com` that include 16KB page-size support. Upstream's official NDK compatibility table tops out at r25 — do not "upgrade" to r27/r28 thinking it's an improvement; the upstream FFmpeg build does not support those NDKs.

## Build commands

The `*.sh` scripts at the repo root drive everything. They are not Make/Gradle wrappers — they orchestrate cross-compilation of FFmpeg + ~50 external libs + the platform wrapper, then bundle the result.

```bash
# Android — must export ANDROID_SDK_ROOT and ANDROID_NDK_ROOT first.
# The 16KB-capable NDK comes from ci.android.com (see workflow yml for the exact URL).
./android.sh -d --full --enable-gpl --disable-arm-v7a   # Main release (API 24+)
./android.sh -d --lts --full --enable-gpl --disable-arm-v7a  # LTS (API 16+)

# Apple
./ios.sh        # iOS
./macos.sh      # macOS
./tvos.sh       # tvOS
./apple.sh      # Bundles existing iOS/macOS/tvOS frameworks into an xcframework — does NOT recompile

# Linux
./linux.sh
```

Common flags (apply to all platform scripts unless noted):
- `--full` — enable every supported external library (LGPL set unless `--enable-gpl` also passed).
- `--enable-gpl` — opts into GPL libs (x264, x265, xvidcore, libvidstab, rubberband). Without this flag, `--full` only enables LGPL libs.
- `--enable-<lib>` / `--disable-lib-<lib>` — toggle a single external library.
- `--disable-<arch>` — drop an architecture (`arm-v7a`, `arm-v7a-neon`, `arm64-v8a`, `x86`, `x86-64`).
- `--lts` — switch to the LTS variant (Android API 16+, older iOS/macOS/tvOS minimums, frameworks instead of xcframeworks).
- `-d` / `--debug` — debug build.
- `--api-level=N` (Android) — override min SDK.
- `--rebuild-<lib>` / `--reconf-<lib>` / `--redownload-<lib>` — force a single library to redo that step (useful when iterating on one dep).
- `-h` / `--help` — full option list (much longer than this — defer to `--help` rather than guessing).

Build output lands under `prebuilt/`:
- Android: `prebuilt/bundle-android-aar/ffmpeg-kit.aar` (or `bundle-android-aar-lts/`).
- Apple: `prebuilt/bundle-*-xcframework/` or `bundle-*-framework/` for LTS.
- Per-arch intermediate libs in `prebuilt/<platform>-<arch>/`.

All script output is redirected to `build.log` at repo root. **When a build fails, read `build.log` first** — the terminal output is intentionally sparse. For FFmpeg-configure failures specifically, check `src/ffmpeg/ffbuild/config.log`.

There are no unit tests in this repo. Functional testing is done via the separate `arthenica/ffmpeg-kit-test` project.

## Architecture

### Script layering (the build is shell-driven, not gradle-driven)

```
android.sh / ios.sh / linux.sh / macos.sh / tvos.sh   ← entrypoint, parses flags, sets defaults
        │
        ├── scripts/variable.sh                ← global arrays: ENABLED_ARCHITECTURES, ENABLED_LIBRARIES (~62 slots)
        ├── scripts/function.sh                ← shared helpers
        ├── scripts/function-<platform>.sh     ← platform-specific helpers (toolchain, arch names, gradle/xcodebuild glue)
        │
        └── per-arch loop → scripts/main-<platform>.sh
                                   │
                                   └── for each enabled library: scripts/<platform>/<lib>.sh
                                                                  (builds that one external lib for the current arch)
                                                                  └── finally scripts/<platform>/ffmpeg.sh
                                                                  └── then ffmpeg-kit wrapper (gradle/xcodebuild)
```

Library enable/disable state lives in **positional arrays** (`ENABLED_LIBRARIES[42]=1`). Every library has an integer ID; helpers like `get_library_name <id>`, `is_gpl_licensed <id>` translate. When adding or modifying a library you must touch the array sizing in `variable.sh` and the dispatch loops in `android.sh`/`apple.sh`/etc. — they iterate hardcoded ranges (e.g. `for library in {0..61}`).

### Wrapper library structure

The "FFmpegKit wrapper" is a thin platform-specific layer that exposes FFmpeg/FFprobe to the host language:

- `android/ffmpeg-kit-android-lib/src/main/java/com/arthenica/ffmpegkit/` — Java API (`FFmpegKit`, `FFprobeKit`, `FFmpegSession`, `FFmpegKitConfig`, …).
- `android/ffmpeg-kit-android-lib/src/main/cpp/` — JNI glue + the `fftools_*.c` files (FFmpeg's CLI extracted as a library).
- `apple/src/` — Objective-C API (mirrors the Java one) + the same `fftools_*.c` files.
- `flutter/flutter/` — Dart wrapper that delegates to Android/iOS/macOS native code.
- `react-native/` — JS/TS wrapper that delegates the same way.
- `linux/src/` — C++ API.

The `fftools_*.c` files are vendored copies of FFmpeg's CLI sources, modified so `ffmpeg` / `ffprobe` can be invoked as a library function instead of a `main()`. They appear in both `android/.../cpp/` and `apple/src/`. **These must be kept in sync with the FFmpeg version being built** — patches under `tools/patch/` are typically what re-applies any local changes after `src/ffmpeg/` is freshly cloned.

### `tools/android/build.gradle` vs `android/ffmpeg-kit-android-lib/build.gradle`

The build script **copies** `tools/android/build.gradle` (or `build.lts.gradle`) over `android/ffmpeg-kit-android-lib/build.gradle` at the top of `android.sh`. Editing the file under `android/ffmpeg-kit-android-lib/` directly will be silently overwritten. Edit the template under `tools/android/` instead.

### Source download

`src/ffmpeg/` and `src/cpu-features/` are populated by the build script (`download_gnu_config`, `downloaded_library_sources`) — they are not committed. `.gitignore` excludes `prebuilt/`, `*.log`, `.tmp/`. Network access to `github.com` is required for a fresh build.

## CI / release flow

`.github/workflows/android-build-scripts-16kb.yml` is the workflow specific to this fork. It downloads the 16KB-compatible NDK from a ci.android.com URL (currently r23-based, despite being labeled "r25" in the workflow name — the URL is the source of truth). It runs `--full --enable-gpl --disable-arm-v7a` for both main and LTS variants and uploads the AARs as artifacts; `workflow_dispatch` runs additionally cut a prerelease GitHub release. The pre-signed Google Cloud Storage URL in that workflow expires — if the NDK download starts 403'ing, the URL needs to be re-fetched from ci.android.com.

The other `*.yml` workflows (`android-build-scripts.yml`, `ios-`, `macos-`, etc.) are inherited from upstream and may not be actively used by this fork.

## Pitfalls specific to this fork

- **GPL flag validation**: `--full` plus `--enable-gpl` enables GPL libs; `--full` alone silently skips them. If you want GPL libs explicitly via `--enable-x264` etc. without `--enable-gpl`, the build aborts with "Invalid configuration detected".
- **64-bit Android API floor**: `arm64-v8a` and `x86_64` force `API=21` even if `--api-level` is lower (Android NDK doesn't support 64-bit ABIs below 21). The script saves and restores `ORIGINAL_API` around this.
- **arm-v7a is currently disabled** in the CI workflow (`--disable-arm-v7a`). If re-enabling, expect 16KB-page-size verification to need re-checking on that arch.
- **`build.gradle.tmp`** under `android/ffmpeg-kit-android-lib/` is a backup the script writes; do not commit it.
