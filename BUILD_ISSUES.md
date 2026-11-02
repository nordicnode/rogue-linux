# Build Issues Found During README Testing

This document lists issues encountered when following the README.md build instructions on macOS (darwin 25.0.0, Apple Silicon), plus the Linux port notes.

## Current Status

Both issues below are fixed in the tree, and the game builds and runs on
modern Linux. Verified on CachyOS (Arch Linux, x86_64) with GCC 16.2.1,
ncurses 6.6 and GNU make 4.4.1 from a clean tree:

- `./configure && make` completes without errors.
- `./rogue` starts, plays, saves to `~/.rogue.save` (mode 0400), restores, and
  lists scores with `./rogue -s`.
- `make install DESTDIR=...` produces a correct system layout, and `makepkg`
  builds an Arch package from this tree (see `PKGBUILD`).

See the Linux port notes at the end of this document for what changed.

## Date
Testing performed: Following README.md Quick Start instructions

## Environment
- **OS**: macOS (darwin 25.0.0)
- **Architecture**: Apple Silicon (arm64)
- **Compiler**: GCC/Clang (available)
- **make**: Available
- **ncurses**: Installed via Homebrew at `/opt/homebrew/opt/ncurses`

## Issues Found

### 1. Compilation Error: Incomplete Type 'WINDOW'

**Location**: `main.c:241-242`

**Error Message**:
```
main.c:241:11: error: incomplete definition of type 'WINDOW' (aka 'struct _win_st')
  241 |     curscr->_cury = oy;
      |     ~~~~~~^
main.c:242:11: error: incomplete definition of type 'WINDOW' (aka 'struct _win_st')
  242 |     curscr->_curx = ox;
      |     ~~~~~~^
```

**Root Cause**:
The code attempts to access internal ncurses structure members (`curscr->_cury` and `curscr->_curx`) which are not part of the public API in modern ncurses libraries. These internal members are not accessible in both the system ncurses and Homebrew ncurses.

**Context**:
The problematic code is in the `tstp()` function (lines 241-242), which handles stop/start signals (SIGTSTP). After resuming from a stop signal, the code tries to manually restore the cursor position by directly accessing internal ncurses structure members.

**Affected Code**:
```c
void
tstp(int ignored)
{
    // ... code ...
    getyx(curscr, y, x);
    mvcur(y, x, oy, ox);
    fflush(stdout);
    curscr->_cury = oy;  // ERROR: Internal member access
    curscr->_curx = ox;  // ERROR: Internal member access
}
```

**Impact**:
- **Severity**: CRITICAL - Build fails completely
- The game cannot be compiled on modern systems with standard ncurses
- This affects all platforms using modern ncurses (not just macOS)

**Suggested Fix**:
1. Remove the direct structure member access (lines 241-242)
2. The `mvcur()` call on line 239 should be sufficient for cursor positioning
3. If cursor position tracking is needed, use public ncurses API functions like `getyx()` and `move()` instead

**Workaround Attempted**:
- Tried using Homebrew ncurses with explicit include/library paths as suggested in README troubleshooting section
- Same error occurred with both system ncurses and Homebrew ncurses
- Tried manual build using `Makefile.std` as documented in README (Alternative build method)
- Same error occurred with manual build method
- This confirms it's a code compatibility issue, not a configuration or build system issue

### 2. Compiler Warnings (Non-blocking)

**Location**: Multiple files

**Warning Messages**:
- `main.c`: Multiple warnings about deprecated function prototypes (C23 compatibility)
- Function declarations without prototypes in `extern.h` conflicting with definitions

**Impact**:
- **Severity**: LOW - Warnings only, do not prevent compilation
- Code compiles but may have issues with future C standards (C23)

**Note**: These warnings are expected for legacy C code and don't prevent the build from succeeding once the critical error is fixed.

## What Works

1. ✅ **Configure script**: Runs successfully
   - Detects compiler, headers, and libraries correctly
   - Creates `config.h` and `Makefile` properly
   - Works with both system ncurses and Homebrew ncurses paths

2. ✅ **Prerequisites**: All required tools are available
   - C compiler (gcc/clang)
   - make
   - ncurses library (via Homebrew)

3. ✅ **Build system**: Autotools configuration works correctly
   - `configure` script exists and is executable
   - No need for `autoreconf` (as documented in README)

## Recommendations for README Updates

1. **Add Known Compatibility Issue Section**:
   - Document that the codebase has compatibility issues with modern ncurses
   - Note that direct access to internal ncurses structures (`curscr->_cury`, `curscr->_curx`) will fail on modern systems
   - Provide information about which ncurses versions are known to work (if any)

2. **Update macOS Troubleshooting**:
   - The current macOS troubleshooting section mentions Homebrew ncurses paths, but this doesn't solve the compilation error
   - Add a note that even with correct ncurses paths, the build may fail due to code compatibility issues

3. **Add Build Status**:
   - Consider adding a "Known Issues" or "Build Status" section to the README
   - Indicate that the codebase may need patches for modern ncurses compatibility

## Next Steps

1. **Code Fix Required**: The `main.c` file needs to be updated to remove direct access to internal ncurses structure members
2. **Testing**: After fix, verify the game builds and runs correctly
3. **Documentation**: Update README with any workarounds or fixes applied

## Additional Notes

- The README's Quick Start instructions are clear and easy to follow
- The build system (Autotools) is properly configured
- The issue is specifically with code compatibility, not with the build system or documentation
- This appears to be a known issue with porting old curses-based code to modern ncurses

---

## Linux (CachyOS / Arch Linux)

Verified November 2026 on CachyOS (Arch Linux, x86_64), GCC 16.2.1, ncurses
6.6, GNU make 4.4.1.

### Fixes required

1. **ncurses private `WINDOW` fields** (`main.c`, `tstp()`): the resume-from-
   suspend path wrote `curscr->_cury` / `curscr->_curx`, which are not part of
   the public API. It now calls `wmove(curscr, oy, ox)`, which updates the same
   state through the public interface.

2. **GCC 15+ defaults to C23** (`Makefile.in`, `Makefile.std`): this is K&R-era
   C, where `void f()` means "unspecified arguments" and daemon/fuse callbacks
   are invoked through unprototyped function pointers with arguments. Under C23
   `()` means "no arguments", so the build fails. Both makefiles now pass
   `-std=gnu89`, the dialect the code was written for. Passing it explicitly
   keeps the build independent of the compiler's default, which matters as
   distributions keep moving to newer GCC releases.

3. **Setgid scoreboard permissions** (`Makefile.in`): the install rule
   chmod'ed the scoreboard to `0464`. A setgid binary runs with effective
   group `games`, and `0464` grants that group read-only access, so
   `fopen("r+")` in `open_score()` fails and no score is ever recorded.
   Changed to `0664`, matching what the RPM spec's `%attr` already used.

4. **Stale objects after reconfiguring** (`Makefile.in`): the object files had
   no dependency on `config.h`, so running `./configure` with different
   options and then `make` reused objects built against the old configuration.
   `makepkg` hit this: it reconfigured for `/var/games/roguelike/rogue.scr`,
   `make` reported "'rogue' is up to date", and the package shipped a binary
   with the previous build's scoreboard path compiled in. Objects now depend
   on `config.h` and `Makefile`. (The rule sits below the link rule so the
   executable remains make's default goal.)

### Install layout

Arch's `filesystem` package already provides `/var/games` and the `games`
group (`/usr/lib/sysusers.d/arch.conf`, gid 50), so the classic shared-
scoreboard layout works unchanged:

```bash
./configure --prefix=/usr \
    --enable-setgid=games \
    --enable-scorefile=/var/games/roguelike/rogue.scr \
    --enable-lockfile=/var/games/roguelike/rogue.lck
make
make install DESTDIR="$pkgdir"
```

`Makefile.in` was also fixed so that `install`:

- creates the scoreboard's parent directory instead of assuming it exists
  (`install-sh -d`), and skips the scoreboard entirely when it is disabled;
- skips lockfile handling when no lockfile is configured. With an empty
  path, `$(DESTDIR)$(LOCKFILE)` collapsed to `$(DESTDIR)` and the rule
  installed its temporary `test` file into the package root;
- applies `DESTDIR` when testing for the `man6` directory, so the manual page
  lands in `<destdir>/usr/share/man/man6` rather than the host's
  `/usr/share/man`;
- no longer drops the scoreboard at `<destdir>rogue.scr`. The old
  `$(DESTDIR)$(SCOREFILE)` concatenation produced `<destdir>` + `rogue.scr`
  with no path separator when the scorefile was configured as a relative path.

`uninstall` used `$(DESTDIR)$(docdir)$(PROGRAM)` (missing separator) and
removed only the `.doc` file. It now removes every installed document and
`rmdir`s the right directory.

### Packaging

`PKGBUILD` packages this tree for Arch/CachyOS (`makepkg -si`). Because no
tagged upstream tarball contains the local changes, it builds in place from
`$startdir`, so run it from the repository root.

### Remaining notes

- `md_gethomedir()` reads the home directory from `getpwuid()`, not `$HOME`,
  so save files always go to the passwd home directory. That is original
  behaviour rather than a Linux bug, but it is surprising when testing with a
  temporary `$HOME`.
- `make install` run as a normal user cannot `chgrp games`; a setgid install
  needs root or fakeroot (which `makepkg` provides).

