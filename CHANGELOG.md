# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/2.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.3.0] - 2026-09-20

First release built by the hardened pipeline: every release now ships `SHA256SUMS` covering the
executables as well as the archives, plus `BUILD-INFO-windows.txt` (commit, workflow run, runner
image and toolchain versions) and the GUI's DLL dependency/import listings. The build fails if the
removed audio component ever reappears in the executable.

### Removed

- Background music in the Windows GUI. This removes the bundled precompiled uFMOD library
  (`extern/ufmod.lib`) and its header, the embedded XM module (`resources/music3.xm`,
  `assets/late_at_morning.xm`) and the `winmm` link dependency. The patcher no longer contains
  any third-party binary code; patching behaviour is unchanged.

## [1.2.1] - 2026-06-14

1.2.1 is the first release to include a change log. The main focus of this release was a command line interface that
will work in Windows and multiple distros of Linux. The build system has been moved over to CMake and should be much
easier to build from source on different systems.

### Added

- Mutex to limit running instances to one.
- Windows CLI build. The patch can now be applied using the Windows command line.
- Linux CLI build. The release will be statically linked for a wider range of compatible distros, including Steam Deck.
- Dockerfile and scripts to help building the linux binary locally.

### Changed

- Renamed and reorganized files and directories.
- Changed build system to CMake.
- Changed from the Windows four-part version number to semantic versioning.

### Removed

- Removed Visual Studio project and solution files in favor of CMake.
