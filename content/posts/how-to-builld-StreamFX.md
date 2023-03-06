---
title: "Build StreamFX for OBS 29"
date: 2023-03-06T18:52:43+08:00
tags: [OBS, StreamFX]
---

## Why

The pre-built binaries of [StreamFX](https://github.com/Xaymar/obs-StreamFX) are only available to sponsors now. [And the author purged all the 0.12 alpha builds from GitHub](https://web.archive.org/web/20230108104555/https://github.com/Xaymar/obs-StreamFX/releases). The only **free** option for OBS 27+ users is to build it from source. Okay, fair enough.

## How

There is a [official guide](https://github.com/Xaymar/obs-StreamFX/wiki/Building) for how to build the project. It works in general but misses some details. If you are familar with GitHub actions, it is better to check the [main.yml](https://github.com/Xaymar/obs-StreamFX/blob/root/.github/workflows/main.yml) out. 

All the following instructions are a de-templatized version of the original action, and they are Windows x64 only. That said, the idea still applies to Linux or macOS.

1. Install all the [prerequisites](https://github.com/Xaymar/obs-StreamFX/wiki/Building#1-install-prerequisites--dependencies) for your platform.
   1. You need two tarballs, named after `$OS-deps-$VERSION-$ARCH.tar.zx` and `$OS-deps-qt6-$VERSION-$ARCH.tar.zx`. Extract them into two different paths, noted as `$OBS` and `$OBS_QT6`. For a Windows build, the expected version of OBS dependencies is overrided by [obs-StreamFX/third-party/DEPS_VERSION_WIN](https://github.com/Xaymar/obs-StreamFX/blob/root/third-party/DEPS_VERSION_WIN).
   2. If you want to make a Windows installer, [Inno Setup](https://jrsoftware.org/isinfo.php) is required.
   3. [LLVM](https://releases.llvm.org/) is optional. The official pipeline uses LLVM 14. LLVM 15 works though. Make sure to install it if you don't want to change the following commands.
   4. Do make sure to check out all the submodules.
   5. You need a Windows SDK. The official pipeline uses `10.0.20348.0` which could be found [here](https://developer.microsoft.com/en-us/windows/downloads/sdk-archive/).
2. Build LibOBS (commands are Windows specific)
   1. Run `cmake -S "$ROOT/third-party/obs-studio" -B "$ROOT/build/obs" -G "Visual Studio 17 2022" -A x64 -DCMAKE_SYSTEM_VERSION="10.0.20348.0" -DCMAKE_BUILD_TYPE="Release" -DCMAKE_INSTALL_PREFIX="$ROOT/build/obs/install" -DENABLE_PLUGINS=OFF -DENABLE_UI=OFF -DENABLE_SCRIPTING=OFF -DCMAKE_PREFIX_PATH="$OBS;$OBS_QT6"` in PowerShell. `$ROOT` is where you cloned the StreamFX repository, `$OBS` and `$OBS_QT6` are paths you just extracted the two tarballs to.
   2. Run `cmake --build "$ROOT/build/obs" --config Release --target obs-frontend-api`
   3. Run `cmake --install "$ROOT/build/obs" --config Release --component obs_libraries`
3. Build StreamFX
   1. Run `cmake -S "$ROOT" -B "$ROOT/build/ci" -G "Visual Studio 17 2022" -A x64 -DCMAKE_SYSTEM_VERSION="10.0.20348.0" -DCMAKE_BUILD_TYPE="Release" -DCMAKE_INSTALL_PREFIX="$ROOT/build/ci/install" -DPACKAGE_NAME="streamfx-windows" -DPACKAGE_PREFIX="$ROOT/build/package" -DENABLE_CLANG=TRUE -DCLANG_PATH="$LLVM" -DENABLE_PROFILING=ON -Dlibobs_DIR="$ROOT/build/obs/install" -DQt_DIR="$OBS_QT6" -DFFmpeg_DIR="$OBS" -DCURL_DIR="$OBS"`. `$LLVM` is the installation of your LLVM, on Windows, it is usually `C:\Program Files\LLVM\bin\`.
   2. Run `cmake --build "$ROOT/build/ci" --config RelWithDebInfo --target install`
4. Packaging
   1. Run `& "$INNO" /V10 "$ROOT\build\ci\installer.iss"`. `$INNO` is your Inno Setup installation, on Windows, it is usually `C:\Program Files (x86)\Inno Setup 6\ISCC.exe`.
   2. Enjoy your installer. :)

