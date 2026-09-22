# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of [N64FlashcartMenu](https://github.com/Polprzewodnikowy/N64FlashcartMenu) 0.3.4 (AGPL-3.0) that adds save states, a virtual Controller Pak, slow motion and frame step, screenshots and configurable hotkeys for the SummerCart64. It is two programs built two different ways: the menu, a libdragon C application, and the engine in `src/boot/hook/`, freestanding MIPS code that runs inside a retail game's exception handler and is embedded in the menu as generated blobs.

The fork's user docs are `docs/savestates.md` (setup, controls, the compatibility table, and a "How it works" section to read before touching `src/boot/`) and `docs/state-format.md` (the spec for the state files on the SD card). The Menu information screen (`src/menu/views/credits.c`) sends save-state bug reports to `john-steinmeyer/sc64-savestates`. `origin` is `frankjniu/sc64-savestates`.

## Build

The build needs the libdragon toolchain (`N64_INST`; the Makefile includes `$(N64_INST)/include/n64.mk`) and the submodules: `libdragon` (on its `preview` branch) and `src/libs/{libspng,minimp3,miniz,dr_libs}`. A fresh clone doesn't have them yet.

On Linux or WSL with the toolchain installed, run `git submodule update --init`, then `cd libdragon && make -j libdragon tools && make -j install tools-install && cd .. && make -j`. `make install` writes this checkout's libdragon into `$N64_INST`, so don't point `N64_INST` at a toolchain that another checkout also installs into.

On Windows without WSL, don't use a local `make`. Build in the devcontainer image instead, from PowerShell. Under Git Bash, prefix the commands with `MSYS_NO_PATHCONV=1`, because Git Bash rewrites the `-v`/`-w` paths.

```
docker build --progress=plain -t n64flashcartmenu-sc64deployer -f .devcontainer/flashcart/Dockerfile.sc64deployer .
# first time, or after a submodule change:
docker run --rm -v "${PWD}:/workspaces/N64FlashcartMenu" -w /workspaces/N64FlashcartMenu n64flashcartmenu-sc64deployer bash -lc "git submodule update --init && cd ./libdragon && make clobber -j && make libdragon tools -j && make install tools-install -j && cd .. && make all -j2"
# every other build (a --rm container starts without libdragon installed, hence the reinstall):
docker run --rm -v "${PWD}:/workspaces/N64FlashcartMenu" -w /workspaces/N64FlashcartMenu n64flashcartmenu-sc64deployer bash -lc "cd ./libdragon && make install tools-install -j && cd .. && make -j2"
```

`make` (the same as `make all`) builds `output/N64FlashcartMenu.n64` and a copy for each cart: `sc64menu.n64` is the one this fork ships, and `menu.bin`, `OS64.v64` and `OS64P.v64` come along. Other targets are `make sc64`, `make clean` and `make disassembly` (writes `build/N64FlashcartMenu.lst`).

- `FLAGS` adds compiler flags; CI (`.github/workflows/build.yml`) sets `FLAGS=-DNDEBUG`. The Makefile compiles in ROM autoload through `N64_CFLAGS`, not `FLAGS`. Keep it there: a build that sets `FLAGS` replaces its default value, so autoload would silently disappear from those builds.
- If `MENU_VERSION` has the form `<base>-ss<release>` (for example `0.3.4-ss1.7`), the Makefile fills the release and base-version fields on the Menu information screen from it. Without `-ss`, the screen says "(development build)". The build doesn't track flags as dependencies, so run `make clean` before changing them: `make clean && MENU_VERSION=0.3.4-ss1.7 FLAGS=-DNDEBUG make all`.
- Don't run `make format`. The tree isn't clang-format clean (upstream's `rom_info.c` alone has hundreds of over-length lines), so it would reflow upstream files wholesale and bury the fork's diff.

### Regenerating the hook blobs

`src/boot/hook_blob.c` and `src/boot/hook_blob.h` are generated files, checked in, and the Makefile only compiles them. After any change in `src/boot/hook/` (hook.c, ss_overlay.c, entry.S, state.S, lowpage.S, monitor.S, the `.ld` scripts, ss_font.h or pakcrc.inc), regenerate them with `src/boot/hook/build.sh`. Otherwise the menu keeps shipping the old engine:

```
docker run --rm -v "${PWD}:/workspaces/N64FlashcartMenu" -w /workspaces/N64FlashcartMenu n64flashcartmenu-sc64deployer bash src/boot/hook/build.sh
```

`build.sh` needs bash, python3 and the MIPS toolchain (from `$N64_INST`, or `/opt/libdragon` when that's unset). It writes the two files straight into `src/boot/`. Its intermediates (`*.o`, `*.elf`, `*.bin`, `hook.map`) stay in `src/boot/hook/`, and `.gitignore` covers them. The script fails when a piece outgrows its fixed home:

- the hook must stay within 128 KiB with an empty `.bss`. It is about 119 KiB now, so there isn't much room left.
- the monitor must stay within 16 KiB, with its sections pinned at +0x1200, +0x2200 and +0x3000.
- each vector-page fragment must fit its hole.

Not every toolchain reproduces the committed blob. With GCC 14.4.0 and unchanged sources, the hand-written parts come out byte for byte (the monitor and the vector-page fragments), but the compiled hook doesn't: register allocation and inlining differ all through it. So a regenerated blob is new machine code. Commit one only together with a hook change, and test it on hardware just as you would the change itself.

## Testing

There are no automated tests. `src/libs/acutest` is vendored but unused, and the Makefile's `test` target is commented out. To check a change, build it without warnings, then run it on real hardware: a SummerCart64 plus an Expansion Pak, because the engine depends on the cart. The developer guide says Ares and gopher64 run the menu "without most flashcart features".

`make run`, `run-debug`, `run-debug-reboot` and `run-debug-upload` upload the ROM to a real SC64 with `sc64deployer`. On Windows, after a container build, you can call `localdeploy.bat` (with `/d`, `/dr` or `/dur`) directly. It needs `tools/sc64/sc64deployer.exe`. Inside the container, `remotedeploy.sh` connects to `sc64deployer server` on the host through `REMOTE=host:9064`. The debug variants stream the `debugf()` output. `run-debug-upload` also writes `sc64menu.n64` to the SD card and reboots. `docs/99_developer_guide.md` has the details.

The fork added hooks for testing from a PC:

- USB commands in `src/menu/usb_comm.c`: `ping` and `load-rom <path>@`, plus some development-only ones behind `SC64SS_HOOK_DEV`.
- `hook_cfg.pc_req` and `pc_pad`, which let a PC trigger a save, load, panel or screenshot and inject pad input.

The PC-side tooling isn't in this repo. Neither is the `build_ce.sh` that the comments mention. That script sets the diagnostic switches (`SC64SS_BISECT`, `SC64SS_PDPROBE`, `SC64SS_DIAG_WATCHEPC`, `SC64SS_HOOK_DEV`), which are all off by default.

## Architecture

### The base menu

`main.c` calls `menu_run()` (`src/menu/menu.c`) and then `boot()` (`src/boot/boot.c`). The menu never returns. `menu_run` runs a loop once per frame over the table of views in `menu_views[]`. Each view in `src/menu/views/` has a `view_<name>_init` and a `view_<name>_display` function, and changes screens by setting `menu->next_mode`. `MENU_MODE_BOOT` ends the loop. A new view needs four things:

- a `MENU_MODE_*` value in `menu_state.h`
- declarations in `views/views.h`
- an entry in `menu_views[]`
- a line in the Makefile's `SRCS`

`src/flashcart/flashcart.c` picks a driver, which is a table of function pointers, with the register-level code in the `*_ll.c` files. If no cart is recognized it falls back to a dummy driver. Gate cart-specific behaviour on `flashcart_has_feature()`. The Makefile's rules convert the assets in `assets/` into `filesystem/`, which is packed into the ROM's DFS.

### The save-state engine

1. **Launch** (`src/menu/views/load_rom.c`, the boot path at the end of the file). The menu:
   - decides what to arm: states, the virtual pak, the screenshot button and cheat codes. All of them need `is_memory_expanded()`.
   - picks the placement (see step 3).
   - lays out the state slots in cart SDRAM above the ROM (`sc64ss_slot_table`). The bigger the ROM, the fewer the slots.
   - preallocates the SD files (the state slots, the pak image and `screenshots/pending.bin`) and writes their sector run tables to the cart. The engine can write into an existing file by sector, but it can't create one.
   - copies the hook blob to its staging area on the cart and patches the staged `hook_cfg`.
   - writes the monitor to the cart.
   - collects the `boot_patches`: NOPs over the game's `mtc0` writes to WatchLo, and the compressed-code scrub stub.
2. **Boot** (`src/boot/boot.c`, `src/boot/cheats.c`). The engine rides on upstream's Datel cheat engine, which the patched boot installs on the game's exception vector through a CP0 watchpoint. With no cheat codes, the engine sits in the vector page at 0x80000090. With codes, it sits at its classic address, 0x807C5C00. A libdragon ROM's boot code (IPL3) clears all of RAM, so for those ROMs the single `jr a0` hand-off in the boot code on the cart is pointed at a stub on the cart instead (`boot_libdragon_stage`).
3. **In the game**, the engine uses one of two placements:
   - **Borrowed** (the default). Gate fragments in the exception-vector page (`lowpage.S`) send VI and SI interrupts to the monitor (`monitor.S`). The monitor runs in place from cart SDRAM. It reads the pad from the game's own joybus buffer and answers the virtual pak. For a save, load, panel or screenshot, it stashes the top 128 KiB of RAM to the cart, pulls in the hook from staging, runs it, and then restores that RAM. Only the gate stays in RAM, which is why games that use all 8 MiB still work.
   - **Resident**. The hook stays in the top of RAM (0x807D0000) for the whole run. This placement is used for three cases: the per-game "Slow motion" option (`hook_borrowed=0`, which slow motion and frame step need), any launch with cheat codes, and every libdragon ROM.
   - The hook itself: `entry.S` saves the registers and switches to the hook's own stack, then calls `hook_tick()` in `hook.c`. `hook.c` handles state capture and restore, the SD mirror, the VI service that holds the picture steady during a freeze, and the virtual pak's patch into libultra's `__osException`. `ss_overlay.c` draws the panel, messages, thumbnails and screenshots, and is `#include`d by hook.c rather than compiled on its own. `state.S` saves the CPU context, the FPU and the TLB.
4. **Back in the menu**. `sc64ss_shots_finish()` files the screenshots from `pending.bin` into per-ROM folders. `menu_init()` copies the vector page that a hung game left behind into the cart buffer, where a PC can read it for a post-mortem.

### Contracts that span files

- **Cart SDRAM layout and slot geometry.** The `#define`s in `hook.c` and `ss_overlay.c` are the single source of truth. The map is the comment block near the top of hook.c. `build.sh` extracts them with a regex into `hook_blob.h` as `SC64SS_*`, and the menu uses only those names. Keep the definitions as plain `#define NAME 0x…u` lines, or the extraction breaks.
- **`struct hook_cfg`** (hook.c). Its offsets are mirrored in two places:
  - `monitor.S`, as `.set CFG_*` values, checked by `_Static_assert`s in hook.c.
  - `load_rom.c`, which writes the staged copy by word index, and nothing checks it. For example, `sc64ss_cfg[10]` is `combo_save`, and `sc64ss_cfg[27]` (`spare`) holds flag bits: bit 2 means the virtual pak is in, bit 5 means borrowed, and bits 12–13 hold the pak's port.

  Change all three together.
- **The vector page** (0x80000000–0x800003FF). The header of `lowpage.S` maps who owns which words, and some games claim words there:
  - Banjo-Kazooie's copy protection reads 0x1D8.
  - GoldenEye keeps a TLB handler at 0x000–0x07F.
  - Excitebike 64 checks the IPL3 epilogue at 0x200.
  - Perfect Dark's boot stack reaches down to 0x35C.

  Read that header and the comments in `cheats.c` before you place anything there.
- **State files.** `docs/state-format.md` is the spec, and `HOOK_VERSION` in hook.c records what a state carries. The user docs promise that later builds will keep loading older states. So a format change has to bump `HOOK_VERSION`, keep old states loading, and update the spec.
- **The libdragon hand-off search** is implemented twice: once on the cart in `boot.c` (`boot_libdragon_handoff`), and once on the file in `rom_info.c` (`rom_libdragon_handoff`). The two must agree.
- **Per-title special cases** all live in `load_rom.c`, keyed by game code:
  - `sc64ss_titles[]`: the watchpoint on writes only, the monitor's scratch moved to 0x2A8, or the resident placement forced.
  - `sc64ss_full_ram_title()`: games that use all 8 MiB. They are forced to the borrowed placement and refused Slow motion.
  - `scrubs[]`: games whose compressed code hides their WatchLo writes.

  Test results go in the compatibility table in `docs/savestates.md`.
- **Settings.** Each ROM's own options live in its `.ini` file, handled in `rom_info.c`. The every-game hotkeys live under `[savestates]` in `sd:/menu/config.ini`, handled in `settings.c`. `docs/savestates.md` lists the key names.

### Rules for code in `src/boot/hook/`

This code runs with EXL=1 inside the game's exception handler, and it must never fault or disturb the game:

- No libc.
- No FPU. The code is built with `-msoft-float`, and `state.S` opts in explicitly where it has to.
- No division or modulo, and no jump tables.
- No `.bss`: give every static an initializer, because nothing zeroes that RAM.
- The build uses `-Werror`.
- Touch the PI bus only when it's idle, always with timeouts, and put back the game's PI address registers afterwards.
- Code that runs from the cart (the monitor) can't start a PI DMA, and can't follow a cart write with an instruction fetch. Those operations go through RAM routines in `lowpage.S`.
- The monitor may use only t0–t3, a0–a3, v0, ra and k0/k1. Every other register holds live game state.

## Conventions

- Mark every change to an upstream file with an `SC64SS` comment: `// SC64SS: <why>`, or `/**< SC64SS: … */` on a struct member. Symbols that exist only in the fork use the `sc64ss_` or `SC64SS_` prefix. This keeps the fork's changes findable with `grep -rn SC64SS src`, for example when moving to a new upstream release.
- The comments record the hardware evidence behind each decision: which game broke and how, or what was measured on the console. Keep that evidence when you change the code, because most of it can't be rediscovered without the hardware.
- A release updates three places:
  - the top section of `CHANGELOG.md`, headed `## 0.3.4-ssX.Y (date)`
  - the "New in X.Y" paragraph at the top of `README.md`. Everything below the README's "unchanged upstream README" line stays as upstream wrote it.
  - `docs/savestates.md`

  Release commits have been titled `Save states X.Y: <summary>`.
- `.github/copilot-instructions.md` is upstream's rulebook for AI agents. In short:
  - Keep diffs minimal and local: no refactors, helper extraction, renames, file moves or style-only rewrites unless asked.
  - Use C and libdragon only, and leave `src/libs/` alone.
  - Menu features must degrade gracefully on a 4 MiB console.
  - Treat `src/flashcart/`, `src/boot/` and the Makefile's rules as risky.
