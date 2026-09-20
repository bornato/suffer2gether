# CLAUDE.md — suffer2gether

Project knowledge for Claude Code sessions. Everything marked **verified** was checked
directly against the repository at tag `v1.2.1` (commit `70d387ff849ac893026bacf558602f51be70697a`),
the published v1.2.1 release binaries, or upstream documentation, on 2026-09-20, and then
re-checked against the `remove-ufmod` branch described in §15.
Items under **Unresolved / not verified** are exactly that — do not treat them as facts.

**State of the tree after the `remove-ufmod` branch (§15):** the uFMOD library, its header, the
XM module and the `winmm` link are gone; `gdiplus` is linked explicitly; the CI workflows are
pinned, permission-scoped, record provenance and fail if the audio component reappears.
Sections 7–9 describe v1.2.1 first and then state what changed, because the v1.2.1 binaries
remain the reference for verification.

## 1. Purpose

Suffer2gether patches the Steam game *Green Hell* (Steam app id 815370) so that up to
8 players can join a co-op session instead of the default 4. It does this by rewriting
one byte of CIL in the game's managed assembly `GH_Data/Managed/Assembly-CSharp.dll`
(`ldc.i4.4` → `ldc.i4.8`) and stamping a marker so the file is recognised as already
patched. Nothing else is modified. The patch must be re-applied after every game update.

## 2. Repository layout (v1.2.1)

```
CMakeLists.txt              single CMake project; Windows (GUI + CLI) and Linux (CLI) branches
CMakePresets.json           windows-x86-{debug,release}, linux-{debug,release}, static-release (musl via zig)
CHANGELOG.md                Keep-a-Changelog; release notes are extracted from it by CI
README.md                   user docs, build docs, credits
.github/workflows/build.yaml    reusable build workflow (version → build-linux, build-windows)
.github/workflows/release.yaml  on tag push v*: calls build.yaml, packages, `gh release create`
cmake/toolchains/x86_64-linux-musl.cmake   zig cc/c++ static musl toolchain (Linux release only)
docker/                     Dockerfile + build-release.sh for the static Linux binary (local use only)
src/core/                   CPatcher (pattern search/patch), global.h (BYTE_PATTERN/REPLACEMENT), IGameLocator.h
src/platform/windows/       GameLocator: registry lookup of the Steam install path
src/platform/linux/         GameLocator: probes common Steam library paths under $HOME
src/apps/cli/main.cpp       console front-end (Windows CLI target and the only Linux target)
src/apps/windows/           MFC GUI: CApp, CMainWindow, CCustomButton, CScreen, framework.h, res/
src/apps/windows/res/       resource.h, suffer2gether.rc.in, version.rc.in, cli.rc.in (configure_file templates)
resources/                  binary assets compiled into the GUI: 8 BMPs, icon.ico (v1.2.1 also had music3.xm)
assets/                     GIMP .xcf sources for the BMPs/icon, screenshot.png (v1.2.1 also had late_at_morning.xm)
extern/                     v1.2.1 only: ufmod.h + PRECOMPILED 32-bit ufmod.lib — removed on remove-ufmod (see §7)
```

No submodules, no Git LFS, no `.gitattributes`. `out/` build trees are git-ignored.

## 3. Baseline / git facts (verified)

- `v1.2.1` is a **lightweight** tag pointing at `70d387f` (merge of PR #12, 2026-06-14).
  The tag was pushed three times to different commits (`3390036`, `f31f40d`, `70d387f`);
  release run #20 (workflow run 27521663254) on `70d387f` produced the published assets.
- `main` (`e22b51a`) is v1.2.1 plus four commits touching only `README.md` and the Zig
  download URL/path in `.github/workflows/build.yaml` (Zig `0.17.0-dev.813+…` nightly →
  `0.16.0` stable). **All source, CMake, resource and workflow-Windows-job content is
  byte-identical between v1.2.1 and main.**
- Older tags: `v1.1.0.0` (2020, Visual Studio .vcxproj era), `v1.0.0.0`.
  The byte pattern, replacement, magic marker/offset and registry key have not changed
  since v1.1.0.0.
- Published v1.2.1 assets and their SHA-256 (verified against the release's SHA256SUMS):
  - `suffer2gether-v1.2.1-windows-x86.zip` `5396ba43…cb41e` containing
    `suffer2gether.exe` (1,981,952 B, sha256 `18fa4dd4b8ec9afe80877247ef4f15c06ad172069fdfe1bef17a2e646d648f49`)
    and `suffer2gether-cli.exe` (413,184 B, sha256 `6a863307e3c54095ece8258e7e9451cdfd6c85bd92e520d89a1c7002c97bccf3`).
  - `suffer2gether-v1.2.1-linux-x86_64.tgz` `a438be1d…0c720`.
  - SHA256SUMS lists only the archives, not the inner executables.

## 4. Core patching flow (src/core) — verified by reading and by an ASan harness

`CPatcher` opens the target with `std::fstream(in|out|binary)` (needs write access, no truncation).

- `IsPatched()`: reads a `uint32_t` at file offset `0x88` and compares to `0xDEADC0DE`.
  Offset 0x88 is the COFF `TimeDateStamp` field **assuming** `e_lfanew == 0x80`, which
  is true for compiler-produced .NET assemblies. The code does not validate PE structure.
- `Find(BYTE_PATTERN, &pos, 512)`: scans from offset 512 in 4096-byte chunks, seeking back
  22 bytes between chunks so a match may straddle a boundary. Pattern (nibble wildcards `?`):
  `1A 80 ?? ?? ?? ?? 17 80 ?? ?? ?? ?? 73 ?? ?? ?? ?? 80 ?? ?? ?? ?? 2A`
  = CIL `ldc.i4.4; stsfld; ldc.i4.1; stsfld; newobj; stsfld; ret`. First match wins.
- `Patch(pos, "1E")`: rewrites the nibbles of one byte at `pos` (0x1A → 0x1E = `ldc.i4.8`)
  and writes `0xDEADC0DE` little-endian at 0x88. Total change = exactly 5 bytes.
- Both front-ends do: locate DLL → `Load` → `IsPatched` → `Find` → `Patch`.

Known defects in this code (confirmed, not yet fixed — see §11).

## 5. Game location

- Windows (`src/platform/windows/GameLocator.cpp`): reads
  `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Steam App 815370\InstallLocation`
  (64-bit view first, then 32-bit view; `KEY_QUERY_VALUE` only — read-only), appends
  `GH_Data\Managed\Assembly-CSharp.dll`, returns it if it exists, else an empty path.
  **Returns the DLL path.** Includes `framework.h` (MFC `afxwin.h`) just to get Win32 headers.
- Linux: probes `~/.local/share/Steam/…`, `~/.steam/steam/…`, `~/games/steam/…` for
  `Green Hell/GH_Data/Managed/Assembly-CSharp.dll` using `getenv("HOME")`.
  **Returns the game directory, not the DLL path** (inconsistent with Windows; see §11).

## 6. Front-ends

### GUI (`suffer2gether.exe`, MFC, target `suffer2gether` on WIN32)

Execution flow (verified by reading):
1. MFC `WinMain` → `CApp::InitInstance`: `OpenMutex`/`CreateMutex` on
   `SUFFER2GETHER_1acbf36d-…` (second instance exits silently via `return FALSE`);
   `CWinApp::InitInstance`; `GdiplusStartup`; **Release builds only** (`#ifndef _DEBUG`):
   `uFMOD_SetVolume(13)` + `uFMOD_PlaySong(MAKEINTRESOURCE(IDR_MUSIC), m_hInstance, XM_RESOURCE)`;
   `new CMainWindow()` assigned to `m_pMainWnd`.
2. `CMainWindow` ctor: `new CScreen` (GDI+ back buffer + bitmap font `IDB_FONT`), `Create`,
   two owner-drawn `CCustomButton`s (`ID_PATCH`, `ID_EXIT`, bitmaps `IDB_PATCH*`/`IDB_EXIT*`),
   60 Hz `SetTimer(ID_RENDERTIMER)`. `PreCreateWindow`: borderless `WS_POPUP`, `WS_EX_TOPMOST | WS_EX_LAYERED`,
   colour-key RGB(0,255,0), window sized/centred from `IDB_BACKGROUND`; class brush = background bitmap.
3. `OnPatch` (PATCH button): if no path yet → `GameLocator::GetGamePath()`; if empty →
   "GAME FILES NOT DETECTED" + `CFileDialog` filtered to `Assembly-CSharp.dll` (cancel returns).
   Then `m_Patcher.Load(path)` (**return value ignored**), `IsPatched` → "GAME IS ALREADY PATCHED",
   `Find` fail → "ERROR MISSING OPCODES", `Patch` fail → "PATCH FAILED", success → "GAME IS READY TO PLAY".
4. `OnExit` → `DestroyWindow` → MFC posts `WM_QUIT` → `CApp::ExitInstance`: delete main window,
   `GdiplusShutdown`, **Release only** `uFMOD_StopSong()`, `ReleaseMutex`.
   `CMainWindow::PostNcDestroy` frees timer/brush/buttons/screen but does not `delete this`.

Screen text is drawn from a bitmap font that only covers `A–Z`, `0–9` and space; all messages are
hard-coded upper-case. Dragging works via `WM_NCLBUTTONDOWN/HTCAPTION`.

### CLI (`suffer2gether-cli.exe` on Windows; `suffer2gether` on Linux; `src/apps/cli/main.cpp`)

`argv[1]` = **game root directory** (the code appends `GH_Data/Managed/Assembly-CSharp.dll`);
otherwise auto-detect. Prints outcome, returns 0 on success/already patched, 1 on any error.
(The usage string printed on failure says `<path_to_Assembly-CSharp.dll>`, which is wrong.)

## 7. uFMOD / music — the opaque dependency (verified; removed on `remove-ufmod`)

Facts about v1.2.1, kept because the v1.2.1 binaries are the verification baseline:

- `extern/ufmod.lib`: `!<arch>` archive with one member `ufmod.obj` — COFF **i386 only**,
  timestamp 2008-03-22, `.text` 6,411 B, `.bss` 74,350 B. Unchanged since the initial commit
  (2020-05-03; moved from `libs/`). sha256 `9dc678ada2a44bb947cd63e07e557e803d3f225819f6effd589e7ecde1a3f00b`.
  Exports (stdcall): `uFMOD_PlaySong@12, SetVolume@4, Pause@0, Resume@0, Jump2Pattern@4,
  GetStats@0, GetRowOrder@0, GetTime@0, GetTitle@0`.
  Imports: kernel32 `CreateThread, SetThreadPriority, WaitForSingleObject, Sleep, CloseHandle,
  HeapCreate/HeapAlloc/HeapDestroy, FindResourceA/LoadResource/SizeofResource,
  CreateFileA/ReadFile/SetFilePointer` and **winmm** `waveOutOpen/Close/Write/Reset/
  GetPosition/PrepareHeader/UnprepareHeader`. No networking, no process/registry APIs.
- Used by exactly one file, `src/apps/windows/CApp.cpp`: `uFMOD_SetVolume`, `uFMOD_PlaySong`
  and the macro `uFMOD_StopSong()` (= `uFMOD_PlaySong(0,0,0)`), all inside `#ifndef _DEBUG`.
  `CApp.cpp` also includes `<mmsystem.h>` solely because `ufmod.h` uses `HWAVEOUT`.
- Linked only into the GUI target via `add_library(ufmod_lib STATIC IMPORTED)` +
  `target_link_libraries(suffer2gether PRIVATE ufmod_lib winmm)`; include dir `extern`.
  The CLI does not link it and the shipped CLI imports no WINMM.
- The music is `resources/music3.xm` (byte-identical to `assets/late_at_morning.xm`,
  FastTracker II module "Late at Morning"), embedded as `IDR_MUSIC RCDATA` (ID 311) via
  `suffer2gether.rc.in`; `IDR_MUSIC` is defined in `res/resource.h`.
- In the shipped v1.2.1 GUI EXE: `WINMM.dll` is imported with exactly those seven `waveOut*`
  symbols and nothing else; the XM is present verbatim as RCDATA 311 (15,888 B); uFMOD's
  code is the first object in `.text` (file offset 0x400). Nothing else in the project
  uses winmm or the XM. Removing uFMOD does not affect GUI initialisation, shutdown, patching,
  detection or any resource other than `IDR_MUSIC`.

Other committed binaries: BMPs/ICO (image data only), `.xcf` (GIMP sources), `screenshot.png`.
`ufmod.lib` was the **only** precompiled code in the repository.

What the removal commit ("Remove bundled uFMOD music playback from the Windows GUI") did:
`CApp.cpp` lost the three includes and the two `#ifndef _DEBUG` blocks; `suffer2gether.rc.in`
lost the `IDR_MUSIC` line; `resource.h` lost `#define IDR_MUSIC 311` (no other ID renumbered);
`CMakeLists.txt` lost the `ufmod_lib` IMPORTED target, the `ufmod_lib`/`winmm` link inputs and
the `extern` include directory; `extern/ufmod.lib`, `extern/ufmod.h`, `resources/music3.xm` and
`assets/late_at_morning.xm` were deleted; README credits and CHANGELOG updated. `src/core`,
`src/platform`, `src/apps/cli` and `CMainWindow.cpp` are byte-identical to v1.2.1. A follow-up
commit adds `target_link_libraries(suffer2gether PRIVATE gdiplus)` to make the real GDI+
dependency visible (no binary change expected). After these commits the tree contains no
third-party code at all; the remaining link inputs are the MSVC CRT, MFC and Windows SDK
import libraries.

Expected effect on the GUI EXE (to be confirmed on the first CI build): no `WINMM.dll` in the
import table, no `RCDATA 311`, no `Extended Module:` bytes, none of uFMOD's code; everything
else (imports, resources, version info, behaviour) unchanged from v1.2.1.

## 8. Build system (verified)

- `cmake_minimum_required(VERSION 4.3.3)`; `CXX_STANDARD 26` (MSVC gets `/std:c++latest`;
  GCC 13 silently decays to `-std=gnu++23`). Version comes from `version.txt`
  (`VERSION_MAJOR n` … lines) if present, else 1.0.0; CI writes it from the tag name.
  `if(EXISTS "version.txt")` uses a relative path (CMake only defines this for full paths);
  it works because CI runs cmake from the repo root.
- **Windows** (`if(WIN32)`): Ninja generator, `cl.exe`, **x86 (32-bit)** via the preset's
  `architecture: x86 / strategy: external` — i.e. the environment must already be a 32-bit
  MSVC environment (`vcvars32.bat`). Both targets: `MSVC_RUNTIME_LIBRARY MultiThreaded[Debug]`
  (static CRT `/MT`), C++26.
  - `suffer2gether` (`add_executable(... WIN32)`): `CApp.cpp CCustomButton.cpp CMainWindow.cpp
    CScreen.cpp platform/windows/GameLocator.cpp core/CPatcher.cpp` + generated
    `suffer2gether.rc` + `version.rc`. v1.2.1 linked `ufmod_lib winmm` and included `extern`;
    on `remove-ufmod` it links `gdiplus` only and includes `src/core src/platform/windows
    src/apps/windows`.
  - `suffer2gether-cli`: `cli/main.cpp core/CPatcher.cpp platform/windows/GameLocator.cpp` +
    generated `version.rc` + `cli.rc`. `target_link_options(-static)` — meaningless to MSVC
    `link.exe` (expected to produce warning LNK4044); the static CRT comes from `/MT`.
  - `USE_MFC TRUE` is **not a CMake property** (inert). MFC is linked by `afx.h`'s
    `#pragma comment(lib, "nafxcw.lib")` (static, **MBCS/ANSI** — no `_UNICODE`), plus
    `libcmt`, kernel32/user32/gdi32/msimg32/comdlg32/winspool/advapi32/shell32/comctl32/
    shlwapi/uxtheme/windowscodecs. `gdiplus.lib` is not named anywhere in CMake; in the MFC 14.0
    sources inspected, every MFC static-library object carries `/DEFAULTLIB:gdiplus.lib`
    (MFC's own PCH includes `afxtoolbarimages.h` → `atlimage.h`), which is the mechanism that
    lets the v1.2.1 build link without an explicit gdiplus entry (VS 2026 headers not inspected
    directly). Those MFC headers do **not** force winmm (only `afxsound.obj` names it, and that
    object is not pulled in: the shipped GUI imports no `PlaySound`, the shipped CLI imports no WINMM).
  - Because `platform/windows/GameLocator.cpp` includes `framework.h` (`afxwin.h`), the
    "no-MFC" CLI also links static MFC (shipped CLI imports OLEACC/WINSPOOL/ole32/OLEAUT32).
  - `.rc.in` files bake `@CMAKE_SOURCE_DIR@\\resources\\…` paths; only the data is embedded.
  - Debug vs Release: `_DEBUG` (from `/MTd`) enables `DEBUG_NEW` (and, in v1.2.1, disabled the music calls);
    Release adds `NDEBUG`, `/O2`.
- **Linux** (`elseif(UNIX)`): one target `suffer2gether` = CLI sources + `platform/linux/GameLocator.cpp`.
  `static-release` preset uses `zig cc/c++ -target x86_64-linux-musl -O2 … -static -s -Wl,--gc-sections`.
- Building the Windows targets requires MSVC + MFC (x86, MBCS); it cannot be cross-compiled
  with MinGW. The Linux CLI configures and builds cleanly with GCC 13 + CMake 4.4.3 (verified).

Commands (from repo root): `cmake --preset windows-x86-release -B out && cmake --build out`
(inside a `vcvars32` shell); Linux: `cmake --preset linux-release -B out && cmake --build out`.

## 9. CI / release pipeline (verified from the workflow files and the v1.2.1 run summary)

### 9a. As of v1.2.1 / main

- `build.yaml`: triggers `pull_request` and `workflow_call` (inputs `upload-artifacts`, `version`).
  Job `version` parses `vX.Y.Z`; `build-linux` (`ubuntu-latest`) downloads **unpinned-by-hash**
  CMake 4.3.3, Ninja 1.13.2 and a Zig **nightly** (v1.2.1) / Zig 0.16.0 (main) with `curl`, no
  checksum verification, caches them, builds `static-release`; `build-windows`
  (`runs-on: windows-2025-vs2026`, `shell: cmd`) writes `version.txt`, calls the hard-coded
  `C:\Program Files\Microsoft Visual Studio\18\Enterprise\VC\Auxiliary\Build\vcvars32.bat`,
  then `cmake --preset windows-x86-release -B out && cmake --build out`. Downloads nothing.
  Uploads `out/suffer2gether.exe` and `out/suffer2gether-cli.exe` as separate artifacts.
- `release.yaml` (name is empty; run-name "Build and Release"): on `push` of tags `v*`, calls
  `build.yaml`, then job `package-and-release` (`permissions: contents: write`) downloads
  artifacts, extracts the matching CHANGELOG section, zips the two Windows EXEs, tars the Linux
  binary, writes `SHA256SUMS` (archives only) and runs `gh release create` with `GITHUB_TOKEN`.
- Actions are referenced by major tag only: `actions/checkout@v6`, `actions/cache@v5`,
  `actions/upload-artifact@v7`, `actions/download-artifact@v8`. No third-party actions. No
  secrets beyond `GITHUB_TOKEN`. `build.yaml` sets no `permissions:` block.
- v1.2.1 release run: Windows job 46 s, artifacts and logs now expired. Linker version in the
  EXE is 14.51 (MSVC 19.51, VS 2026). GitHub's `windows-2025-vs2026` label is documented as a
  **testing** label that aliases to `windows-2025` after the June 2026 migration; the VS 2026
  image (README `Windows2025-VS2026-Readme.md`, image 20260907) ships VS Enterprise 2026 18.9 at
  `C:\Program Files\Microsoft Visual Studio\18\Enterprise`, CMake 4.4.3, Ninja 1.13.2, component
  `Microsoft.VisualStudio.Component.VC.ATLMFC`.
- The release job depends on the whole `build` workflow, so a failing Linux job blocks the Windows
  release as well.
- `pull_request` runs upload no artifacts in v1.2.1 (`inputs.upload-artifacts` is empty on that
  event), so a PR-built EXE could not be inspected.

### 9b. As of the `remove-ufmod` branch (two CI commits; `actionlint` clean)

- Both workflows pin `actions/checkout`, `actions/cache`, `actions/upload-artifact` and
  `actions/download-artifact` to the commit SHAs behind v6.1.0 / v5.1.0 / v7.0.1 / v8.0.1
  (tag in a trailing comment). `permissions: contents: read` at workflow level; only
  `package-and-release` has `contents: write`.
- `build.yaml` also runs on `workflow_dispatch` (inputs default to uploading artifacts and
  version `v0.0.0`) and uploads artifacts on `pull_request` runs.
- Windows job: `runs-on: windows-2025`; step "Locate Visual Studio" uses `vswhere -latest
  -products * -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64
  Microsoft.VisualStudio.Component.VC.ATLMFC -property installationPath` and exports `VSDIR`;
  "Build release" calls `%VSDIR%\VC\Auxiliary\Build\vcvars32.bat`, fails on any failed command
  (cmd does not stop by itself), writes `out\build-windows.log` (verbose Ninja) and checks that
  both EXEs exist. Artifact names/paths are unchanged (`out/suffer2gether.exe`,
  `out/suffer2gether-cli.exe`) because release.yaml depends on them.
- "Record build provenance" writes `out\BUILD-INFO-windows.txt` (repository, commit, ref,
  workflow, run URL, `ImageOS`/`ImageVersion`, VS path, `VCToolsVersion`, `WindowsSDKVersion`,
  cmake/ninja/cl/link versions) and `dumpbin /dependents` + `/imports` listings for both EXEs.
  cmd pitfall handled there: redirections are written before `echo` so values ending in a digit
  are not parsed as handle redirections.
- "Verify audio component is absent and hash binaries" (pwsh) fails the job if
  `suffer2gether.exe` depends on `WINMM.dll`, contains `waveOut` or `Extended Module:`, or
  contains the 32-byte uFMOD code window `85db0f8588000000395eec8b4edc7562…` (`ufmod.obj`
  `.text+0x400`); then writes `out\SHA256SUMS-windows.txt` (sha256sum format, LF). These files
  are uploaded as the "Windows build provenance" artifact; the build log as "Windows build log".
- `release.yaml`: `sha256sum -c SHA256SUMS-windows.txt` after download (artifact integrity),
  SHA256SUMS now lists the executables as well as the archives, the release notes end with the
  commit and run URL, and `BUILD-INFO-windows.txt` + the GUI's dependents/imports listings are
  attached to the release. Linux job: only the action pins and the Zig cache key (`0.16.0`,
  matching the download) changed.
- Not yet run on GitHub at the time of writing: the first PR/dispatch run on the fork is the
  acceptance test for these workflows (see §12).

## 10. Security-relevant behaviour (verified — nothing else exists in the tree)

| Capability | Where | Why |
|---|---|---|
| Registry **read** (HKLM Steam uninstall key) | windows/GameLocator.cpp | locate the game |
| `getenv("HOME")` | linux/GameLocator.cpp | locate the game |
| Read/write of one user-chosen file | core/CPatcher.cpp | the patch itself (5 bytes) |
| Named mutex | CApp.cpp | single instance |
| Topmost layered borderless window | CMainWindow.cpp | skinned UI |
| GDI+ startup/shutdown | CApp.cpp / CScreen / CCustomButton | drawing |
| Audio thread + waveOut (inside ufmod.lib) | extern/ufmod.lib (v1.2.1 only; removed) | background music only |
| `curl` downloads of toolchains | build.yaml (Linux job), docker/Dockerfile | Linux static build only |

No sockets/HTTP/telemetry/update checks, no `CreateProcess`/`ShellExecute`/`system`, no
`LoadLibrary`/`GetProcAddress`, no process injection APIs, no persistence, no credential or
browser data access, no registry writes, no build-time scripts other than the workflows.

## 11. Known defects / code-quality findings (verified unless marked)

1. `CPatcher::Find` never scans the final chunk of the file: the loop condition
   `while (m_File.read(buf, 4096) && …)` is false on a short read, so a pattern inside the last
   partial chunk (and, because of the 22-byte back-seek, almost always the last < 4 KB) is
   not found. Confirmed with a harness. Harmless for the real DLL (pattern is in the IL body).
2. `CPatcher::Find` reads up to 22 bytes past the 4096-byte heap buffer (`m_Buffer[i + j]`
   with no bound on `i + j`). Confirmed heap-buffer-overflow READ with ASan (CPatcher.cpp:65).
3. `CMainWindow::OnPatch` ignores `Load()`'s return value; if the DLL cannot be opened for
   writing (permissions, file in use) the GUI reports "ERROR MISSING OPCODES". `IsPatched()`
   on a closed/short stream compares an uninitialised `uint32_t`.
4. No PE validation before writing at 0x88; no error checking on `write`/`flush` in `Patch`.
5. Linux `GameLocator::GetGamePath` returns the directory while `main.cpp` passes it straight
   to `Load()`, so Linux auto-detection cannot work; CLI usage text contradicts the code.
6. Windows `GameLocator.cpp` depends on MFC's `framework.h` for Win32 headers, dragging static
   MFC into the CLI. `-static` link option is not an MSVC option.
7. ANSI/MBCS build: paths with characters outside the system code page are lossy
   (registry value, file dialog, `argv`). `RegQueryValueEx` does not check `type`/termination.
8. `if(EXISTS "version.txt")` relative path; silently builds as 1.0.0 if cwd ≠ source dir.
9. `MUTEX` handle is released but never closed (cosmetic). `CScreen` font indexing assumes
   `A–Z`/`0–9`.
10. `version.rc.in` labels the CLI as `suffer2gether.exe`; `cli.rc.in` uses an undefined
    `IDI_ICON1` symbol (becomes a *named* resource) — cosmetic.

## 12. Testing / validation notes

- No tests exist in the repository. A throwaway ASan harness exercising `CPatcher` on synthetic
  files was used during the study (not committed); it is a good seed for a CTest.
- The unchanged v1.2.1 **CLI** (sha256 `6a863307…bccf3`) is a byte-exact oracle for the patch:
  any modified GUI must produce a DLL identical to what the CLI produces (exactly 5 bytes differ
  from the original: one at the pattern offset, four at 0x88).
- Binary checks that need no Windows: `llvm-readobj --coff-imports/--coff-resources`, `strings`,
  a search for uFMOD `.text` byte windows (e.g. `ufmod.obj` .text[0x400:0x430]). The CI guard
  (§9b) automates the import/XM/code checks on every build.
- Acceptance of the `remove-ufmod` branch: (1) CI run green on the fork, guard step prints
  "OK"; (2) download "Windows MFC artifact" + "Windows build provenance"; confirm the imports
  listing equals v1.2.1's minus WINMM, no RCDATA 311 (`llvm-readobj --coff-resources`), size
  roughly 22 KB smaller; (3) on Windows: start → detect → patch → "GAME IS ALREADY PATCHED" on
  second click → foreign DLL gives "ERROR MISSING OPCODES" → second instance exits silently →
  EXIT terminates; patched DLL byte-identical to one patched by the v1.2.1 CLI.
- GUI behaviour checks need a real Windows machine (registry detection, dialog, mutex).
- Workflows can be linted locally with `actionlint` (pip package `actionlint-py`).

## 13. Rules for future sessions editing this repository

- Do not change `BYTE_PATTERN`, `BYTE_REPLACEMENT`, the magic marker, `MAGIC_OFFSET`, the
  512-byte start offset or the registry key unless the task is explicitly about patch logic;
  keep patch-logic changes in separate commits from build/CI/asset changes.
- Keep the Windows build x86 + static CRT + static MBCS MFC unless explicitly asked to change it.
- Do not add third-party binaries, downloaded dependencies, or third-party GitHub Actions.
- Do not "fix" the Linux/Docker paths as a side effect of Windows work.
- Any CMake change must keep `cmake --preset windows-x86-release -B out` and the artifact
  paths `out/suffer2gether.exe`, `out/suffer2gether-cli.exe` working (release.yaml depends on them).
- Resource IDs are shared across namespaces in `resource.h` (e.g. 311 is `ID_EXIT`, and was also
  `IDR_MUSIC`); removing an ID must not renumber the others.
- Record verification evidence (imports, resources, hashes) when touching the GUI link graph.
- Update `CHANGELOG.md` (`[Unreleased]`) for user-visible changes; the release job extracts it.
- Never re-introduce precompiled code or an audio dependency; the CI guard (§9b) is meant to
  fail if that happens — do not weaken it to make a build pass.
- `README.md` and `CHANGELOG.md` are CRLF; source, CMake and workflow files are LF. Preserve
  each file's line endings (edit CRLF files with a script that keeps `\r\n`).
- When pinning or bumping actions, resolve the tag to its commit with
  `git ls-remote --tags https://github.com/<owner>/<action>.git` and keep the tag comment.

## 14. Unresolved / not verified

- `extern/ufmod.lib` has **not** been byte-compared with the upstream uFMOD 1.25.2a Win32
  package (SourceForge download blocked in the study session). Its member timestamp
  (2008-03-22) is consistent with that release's date; `ufmod.h` matches a third-party copy
  modulo whitespace.
- The exact warnings emitted by the Windows build (e.g. LNK4044 for `-static`) — logs expired.
- Whether `windows-2025-vs2026` still resolves as a label today (GitHub says it aliases to
  `windows-2025`; not re-tested).
- Whether the Zig nightly URL in the v1.2.1 workflow still exists (`ziglang.org` unreachable
  from the study session; `main` replaced it, which suggests it did not).
- Default `GITHUB_TOKEN` permissions of the repository (a repo setting; not visible).
- Green Hell's current `Assembly-CSharp.dll` still contains the pattern (game updates change it).
- The `remove-ufmod` workflows have been linted (`actionlint`) and the cmd/pwsh steps reviewed,
  but not yet executed on a GitHub runner; the vswhere lookup, the `for /f` version capture and
  the pwsh guard are the parts most likely to need a small fix on first run.

## 15. Change log of Claude's work on this repository

- `phase1-study` (pushed to `bornato/suffer2gether`): `v1.2.1` + this file.
- `remove-ufmod` (based on `main@e22b51a` = v1.2.1 code + README + Zig pin): cherry-pick of the
  CLAUDE.md commit, then "Remove bundled uFMOD music playback from the Windows GUI", "Link
  gdiplus explicitly for the GUI target", "CI: pin actions, declare permissions, locate Visual
  Studio with vswhere", "CI: record build provenance and fail if the audio component
  reappears", and this CLAUDE.md update. Deferred by decision: `/Brepro`, build attestation,
  and every item in §11 (each to be its own PR).
