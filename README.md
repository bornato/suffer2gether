# Green Hell Multiplayer Patch - More Suffering Together
**Suffer2gether** is an open-source multiplayer patch for the survival game *Green Hell*.
The tool modifies the game's core files to expand the default 4-player co-op limit,
allowing up to **8 players to survive together** in the same game session. 

Suffer More; Suffer Together!

The application is distributed in three distinct formats, **Windows GUI, Windows CLI, and Linux CLI**. All core features,
usage rules, multiplayer guidance, and removal instructions apply identically to every version.

![main patch window](assets/screenshot.png?raw=true "main patch window")

---

### Core Features & Compatibility
* **Universal Version Support:** The patch dynamically searches for specific byte patterns to expand the lobby.
Because of this design, the patch **works across different versions of Green Hell** without requiring 
constant code updates.
* **No Save Game Risks:** Applying or using the patch **does not cause any issues with your save games**.
* **Cross-Platform Compatibility:** The tool functions identically across both Windows and multiple Linux distributions,
including the Steam Deck, using a unified core codebase.

### Essential Usage & Multiplayer Guidance
* **Mandatory for All Players:** Every player in your multiplayer session 
**must apply the patch to their own game files**, regardless of whether they use the GUI or CLI versions. 
* **Reapplication Required:** You must re-run the tool to reapply the patch 
**every time Green Hell receives an official game update**, as updates will overwrite your patched files.
* **Nameplate Bug:** If a player joins without the patch, 
**player nameplates will not show up over their heads**, making it incredibly difficult to locate each other in the jungle.
* **Joining Larger Lobbies:** The in-game user interface will **not** show that more than 4 players can be added.
To add more than 4 players, you must **invite friends directly via the Steam/GoG overlay/interface**, 
or have your friends **join your active game session through the Steam/GoG friends list**.

### Performance & Networking Note
* **Peer-to-Peer Architecture:** Green Hell relies on peer-to-peer (P2P) multiplayer, 
meaning the host's computer bears the full burden of calculating and syncing everything
for every player in the session.
* **Host Performance Impact:** Because of this P2P nature, adding more players 
increases the strain on the host's hardware and network. More players can lead to performance 
issues like freezing, crashing, lag, or invisible players.
* **Not Mod-Related:** These performance bottlenecks are inherent to the base game's P2P networking 
design and are **not caused by the Suffer2gether patch** itself. A strong host PC and stable 
connection are highly recommended for large lobbies.

### Uninstallation & Patch Removal
* **Steam/GoG Verification:** To completely remove the patch and return your game to its vanilla state, 
simply use Steam's or Gog's **"Verify integrity of game files"** feature.
* **Automatic Restoration:** Steam/GoG will automatically detect the modified file 
and re-download the original version, seamlessly reverting the game.

### Repository & Security Note
* **Verify Hashes:** Release file hashes will be on every release; 
be sure to verify these hashes to assert no one tampered with the binaries.
* **Review Code:** The full source code and build process is available for review.
* **False Positives:** A few antivirus engines on VirusTotal may flag the tool using heuristic scans.
This is a **false positive**; for example, automated scanners mistakenly flag an embedded `.xm` tracker music 
file as a Windows MSI installer. The tool is entirely safe to use.

## Usage
### Windows  
Download the latest windows patch from the release page or build from source. 
The patch will detect if and where Green Hell is installed so all you have to do is click "PATCH". If the patcher is unable to detect the game, it will prompt you to
locate Green Hell's `Assembly-CSharp.dll` file.
Once the patch says you're ready to play, exit the patch and start playing Green Hell.

### Linux
Download the linux patch and run the file in the terimal. It will attempt to detect the location of Green Hell and the patch. If the patcher is unable to detect the location, you can specify
the directory containing the `Assembly-CSharp.dll` file.

The Linux binary release in this repo is statically linked using Musl and should work on a wide variety of Linux distros, include Arch and Steam Deck.

### Windows and Linux Terminal
The Linux binary is a standalone terminal CLI. The Windows version also comes with a standalone CLI version. Both have the same usage syntax:
```shell
Windows:
  suffer2gether-cli
  suffer2gether-cli "d:\games\steam\steamapps\common\Green Hell"

Linux:
  suffer2gether
  suffer2gether "~/.local/share/Steam/steamapps/common/Green Hell"
```

**This patch will need to be reapplied every time Green Hell is updated.**

**NOTE:** Every player in your multiplayer session will need to apply this patch. If you don't, any player joining after the 4th will have no name plate above their head making it very difficult to see them.

---

## Building From Source
### Requirements

#### Windows
* Windows 10/11
* [Visual Studio Build Tools 2022 or later (requires MFC component)](https://aka.ms/vs/stable/vs_BuildTools.exe)
* [Cmake 4.33+](https://cmake.org/download/)
* [Ninja build system](https://ninja-build.org/)
* [Windows SDK](https://learn.microsoft.com/en-us/windows/apps/windows-sdk/downloads)

#### Linux
* GCC or Clang for native Linux builds
* OR [zig](https://ziglang.org/) for static releases
* [Cmake 4.33+](https://cmake.org/download/)
* [Ninja build system](https://ninja-build.org/)

For musl static builds (release binaries distributed via GitHub Releases), ensure [zig](https://ziglang.org/) is installed and available in PATH.
A Dockerfile and small shell script are available in the `docker` directory. The script will build the image if needed and build the static linux release binary.

---

## Build Overview
This project produces different targets depending on platform:

### Windows
* suffer2gether.exe - GUI application (MFC-based)
* suffer2gether-cli.exe - CLI application (no MFC)

### Linux
* suffer2gether - CLI application

---

## Building on Windows
CMake commands must be run from the project root.

### Release Build
```console
cmake --preset windows-x86-release -B out
cmake --build out
```

### Debug Build
```console
cmake --preset windows-x86-debug -B out
cmake --build out
```

**NOTE**: You can also use Visual Studio 2022+ to open the CMake project and build from within the IDE.

## Building on Linux
CMake commands must be run from the project root.

### Release Build
```bash
cmake --preset linux-release -B out
cmake --build out
```

If you want a release binary that can run on many different Linux distros, use the static release build:
```bash
cmake --preset static-release -B out
cmake --build out
```

### Debug Build
```bash
cmake --preset linux-debug -B out
cmake --build out
```


---
## Build Presets Summary
|Preset|Platform|Type|Notes|
|------|--------|----|-----|
|windows-x86-debug|Windows|Debug|MFC GUI + CLI|
|windows-x86-release|Windows|Release|MFC GUI + CLI|
|linux-debug|Linux|Debug|Native dynamically linked CLI|
|linux-release|Linux|Release|Native dynamically linked CLI|
|static-release|Linux|Release|musl toolchain CLI|

## Editing Resources
If you want to edit the buttons or font you will need to download and install the fonts listed in the [3rd Party Credit](#3rd-party-credit) section of this readme.

## 3rd Party Credit
**Background image:** Repurposed from the Green Hell game files and modified by me for use as a window background. All credit for the walkie talkie images go to Creepy Jar for some fantastic work.

**Screen Font:** Neon Pixel-7 version 1.0 created by [styleseven.com](http://www.styleseven.com/php/get_product.php?product=Neon%20Pixel-7).

**Button Font:**
This font is copyright (c) Jakob Fischer at [www.pizzadude.dk](http://www.pizzadude.dk), all rights reserved. Do not distribute without the author's permission.
Use this font for non-commercial use only! If you plan to use it for commercial purposes, contact me before doing so!

**uFMOD:** [uFMOD](http://ufmod.sourceforge.net/) is used for playing the XM music file.

**Music:** Late at Morning created by Andreas Rohdin (MrGamer / Gamermachine). He has a lot of [great music](https://soundcloud.com/gamermachine). This module was downloaded from [modarchive](https://modarchive.org/index.php?request=view_by_moduleid&query=52842) and is distributed in this repository under the modarchive ["upload agreement"](https://modarchive.org/index.php?faq-licensing).
