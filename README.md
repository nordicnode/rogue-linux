# Rogue 5.4.4 for Linux

[![License](https://img.shields.io/badge/license-BSD-blue.svg)](LICENSE.TXT)

**Rogue** is the original graphical dungeon-crawling adventure game that spawned an
entire genre: procedurally generated dungeons, turn-based combat, unidentified
potions and scrolls, and one very famous Amulet.

This repository takes the original 1999 release (5.4.4), gets it building and
installing cleanly on current Linux toolchains, and packages it for Arch Linux
and CachyOS. Gameplay, balance and mechanics are untouched — the work here is
portability, packaging and documentation.

- **Original authors:** Michael Toy, Ken Arnold, Glenn Wichman
- **Sources derived from:** [Davidslv/rogue](https://github.com/Davidslv/rogue), an archive of the 5.4.4 release
- **Port target:** Arch Linux / CachyOS, x86_64

## Status

Verified on CachyOS (Arch Linux, x86_64) with GCC 16.2.1, ncurses 6.6 and GNU
make 4.4.1:

- clean `./configure && make` with no warnings
- `make install DESTDIR=...` produces a correct system layout
- `makepkg` builds an installable package with a shared scoreboard
- the game starts, generates levels, saves and restores, and reads and writes
  the scoreboard

The portability changes are listed in [Changes from upstream](#changes-from-upstream)
and written up in detail in [BUILD_ISSUES.md](BUILD_ISSUES.md).

## Install

### Arch Linux / CachyOS

A [PKGBUILD](PKGBUILD) is included. From the repository root:

```bash
makepkg -si
```

This installs:

| Path | Contents |
| --- | --- |
| `/usr/bin/rogue` | the game, setgid `games` |
| `/usr/share/man/man6/rogue.6.gz` | manual page |
| `/usr/share/applications/rogue.desktop` | desktop entry |
| `/usr/share/icons/hicolor/32x32/apps/rogue.png` | icon |
| `/usr/share/doc/rogue/` | game documentation and license |
| `/var/games/roguelike/rogue.scr` | shared, site-wide scoreboard |

Both `/var/games` and the `games` group come from Arch's `filesystem` package.
The setgid binary lets any local user post a score while the scoreboard itself
stays read-only for everyone else.

Remove it with `sudo pacman -R rogue`. The scoreboard is marked as a backup
file, so your scores survive reinstalls.

### Build and play from source

Requirements:

```bash
# Arch / CachyOS
sudo pacman -S base-devel ncurses

# Debian / Ubuntu
sudo apt-get install build-essential libncurses5-dev

# Fedora
sudo yum install gcc make ncurses-devel
```

On Debian/Ubuntu the ncurses package is `libncurses5-dev` or `libncurses-dev`
depending on release.

Then:

```bash
./configure
make
./rogue
```

A build tree run this way keeps its scoreboard at `./rogue.scr`, which is fine
for playing locally but not for a shared install.

### System-wide install by hand

```bash
./configure --prefix=/usr \
    --enable-setgid=games \
    --enable-scorefile=/var/games/roguelike/rogue.scr \
    --enable-lockfile=/var/games/roguelike/rogue.lck
make
sudo make install
```

`make install` creates the scoreboard directory, applies `DESTDIR` correctly,
and can be reversed with `sudo make uninstall`.

## Playing

```bash
rogue                       # start a new game
rogue ~/rogue.save          # restore a saved game
rogue -s                    # show the scoreboard
rogue -d                    # print a demo death screen
```

The terminal must be at least **24x80**. Progress is saved to
`rogue.save` in your passwd home directory.

### Controls

| Key | Action | Key | Action |
| --- | --- | --- | --- |
| `h` `j` `k` `l` (or arrows) | move left / down / up / right | `<` `>` | go up / down stairs |
| `y` `u` `b` `n` | move diagonally | `.` | rest for a turn |
| `H` `J` `K` `L` `Y` `U` `B` `N` | run in a direction | `s` | search for a trap or secret door |
| `f` + direction | fight until death or near death | `^` + direction | identify a trap |
| `t` + direction | throw something | `z` + direction | zap a wand |
| `m` + direction | move onto a square without picking it up | `,` | pick something up |
| `i` | inventory | `I` | inventory a single item / identify |
| `q` | quaff a potion | `r` | read a scroll |
| `e` | eat food | `w` | wield a weapon |
| `W` | wear armor | `T` | take armor off |
| `P` | put on a ring | `R` | remove a ring |
| `d` | drop an object | `c` | call an object something |
| `a` | repeat the last command | `S` | save the game |
| `D` | recall what you have discovered | `o` | examine / set options |
| `@` | current stats | `)` `]` `=` | current weapon / armor / rings |
| `^R` | redraw the screen | `^P` | repeat the last message |
| `?` | help for a key or `*` for all | `Q` | quit |

### Options

Game options can be set before play:

```bash
export ROGUEOPTS="name=YourName,terse,jump"
```

Press `o` in-game to examine or change options. Wizard mode is
compiled out by default; `./configure --enable-wizardmode` enables it along
with the `SEED` environment variable for reproducible dungeons.

## Changes from upstream

These are the fixes required to build and install 5.4.4 on a current system.
None of them alter gameplay.

1. **ncurses private fields** — `tstp()` wrote `curscr->_cury` / `curscr->_curx`,
   which are not part of the public API and no longer visible with modern
   headers. It uses `wmove(curscr, oy, ox)` instead.
2. **C23 default** — the code is K&R-era C, where `void f()` means "unspecified
   arguments". GCC 15+ defaults to C23, where `()` means "no arguments", and the
   daemon/fuse callback calls stop compiling. The makefiles pass `-std=gnu89`
   explicitly so the build does not depend on the compiler's default.
3. **Stale objects after reconfiguring** — objects had no dependency on
   `config.h`, so a second `./configure` with different options could reuse
   objects built against the old configuration (this silently shipped a binary
   with the wrong scoreboard path). Objects now depend on `config.h` and
   `Makefile`.
4. **Setgid scoreboard permissions** — installs used mode `0464` for the
   scoreboard. A setgid process is only in group `games`, and `0464` is group
   read-only, so `open_score()` could never write a score. Now `0664`, matching
   what the RPM spec already used.
5. **Install rules** — the scoreboard's directory is created instead of assumed;
   `DESTDIR` is applied when installing the manual page; disabling the
   scoreboard or lockfile no longer leaves stray files in the package root; and
   `make uninstall` removes every installed document.
6. **Arch packaging** — `PKGBUILD` added for a setgid install with a shared
   `/var/games` scoreboard, desktop entry and icon.

## Project layout

All sources live in the repository root, as in the original distribution.

| Group | Files |
| --- | --- |
| Entry point and loop | `main.c`, `command.c`, `io.c` |
| Dungeon generation | `rooms.c`, `passages.c`, `new_level.c` |
| Combat and monsters | `fight.c`, `weapons.c`, `armor.c`, `monsters.c`, `chase.c` |
| Items | `potions.c`, `scrolls.c`, `rings.c`, `sticks.c`, `things.c`, `pack.c` |
| State and support | `init.c`, `state.c`, `save.c`, `move.c`, `daemon.c`, `daemons.c`, `rip.c`, `list.c`, `misc.c` |
| Platform layer | `mdport.c` (POSIX/Windows abstraction), `mach_dep.c` (scoreboard, signals, environment) |
| Headers | `rogue.h`, `extern.h`, `score.h` |
| Build system | `configure.ac`, `Makefile.in`, `Makefile.std`, `config.h.in` |
| Packaging and docs | `PKGBUILD`, `rogue.spec`, `rogue.desktop`, `BUILD_ISSUES.md`, `LICENSE.TXT` |

The code is deliberately pre-C99: `register`, old-style function definitions,
custom control-flow macros (`when`, `otherwise`, `on()`), and linked lists for
the object pack. New code should match that style.

### Manual build without Autotools

`Makefile.std` builds `rogue54` directly with `gcc`:

```bash
make -f Makefile.std
make -f Makefile.std clean
```

The repository also keeps the original Visual Studio project
(`rogue54.sln`, `rogue54.vcproj`) and an RPM spec, but only the Linux build is
maintained and tested here.

## Troubleshooting

**`configure: error: curses library not found`**
Install the ncurses development package for your distribution (see
[Build and play from source](#build-and-play-from-source)).

**`error: incomplete definition of type 'WINDOW'`**
You are building a snapshot that predates the `tstp()` fix; build this tree
instead, or replace the `curscr->_cury` / `_curx` writes with `wmove()`.

**`Sorry, the screen must be at least 24x80`**
Resize the terminal; the game refuses to start on a smaller one.

**`Could not open /var/games/roguelike/rogue.scr for writing: Permission denied`**
For a shared scoreboard, the binary must be setgid `games` and the scoreboard
must be group-writable (`0664`, owned `root:games`). Reinstall through the
PKGBUILD or with `--enable-setgid=games`.

## License and credits

Rogue is distributed under the BSD license; see [LICENSE.TXT](LICENSE.TXT).
The original copyright belongs to Michael Toy, Ken Arnold and Glenn Wichman.

This port keeps the original sources intact and adds portability and packaging
changes only. Changes are documented in [BUILD_ISSUES.md](BUILD_ISSUES.md).
