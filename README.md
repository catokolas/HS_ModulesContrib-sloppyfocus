# hs._ckol.sloppyfocus

A small Hammerspoon native module that focuses a window without raising it
(X11-style "sloppy focus" on macOS). Designed to be called from the
[`FocusFollowsMouse`](https://github.com/catokolas/HS_SpoonsContrib/blob/main/FocusFollowsMouse.spoon) spoon's `_maybeFocus`
hook, but usable from any Lua code that has an `hs.window`.

## Why this exists

[AutoRaise](https://github.com/sbmpost/AutoRaise) does the same thing as a
standalone app, and that's the simpler option for most people. It has one
limitation though: it doesn't reliably focus Chromium-based PWAs (Brave/Chrome
"Install as App" windows). That's the gap this module fills — `hs.window`
identifies PWAs correctly, so passing its pid and window id straight into the
same SkyLight calls AutoRaise uses gets us focus-without-raise that works on
native apps *and* PWAs.

## How it works

Uses three private macOS APIs (resolved via `dlopen`/`dlsym` so a future
macOS that removes them just makes us degrade to a no-op):

- `_SLPSSetFrontProcessWithOptions(psn, wid, flags)` — tells SkyLight to make
  a process front for a specific window. Passing the real window id (not 0)
  is what avoids the raise.
- `SLPSPostEventRecordTo(psn, bytes)` — used to post synthetic key-window
  events (the byte layout was lifted from
  [yabai](https://github.com/koekeishiya/yabai) via AutoRaise).
- `GetProcessForPID` from ApplicationServices — to get a PSN from a pid.

The full recipe is documented inline in `internal.m`; it's a verbatim
port of [`AutoRaise.mm` lines 165-221](https://github.com/sbmpost/AutoRaise/blob/17e0c9abe8155fd2cf2fc0486fc6e1f2caebe978/AutoRaise.mm#L165-L221)
with the gating logic stripped out (we let `hs.window` decide what to
focus).

## Install without compiling (pre-built universal binary)

Each release ships a `sloppyfocus-<version>-macos-universal.zip` containing
a fat `internal.so` (arm64 + x86_64) plus `init.lua`, built against
`-mmacosx-version-min=13.0`. Pull the latest release artifact and unzip
straight into `~/.hammerspoon`:

```bash
# 1. Download (replace <version> with whatever's current on the Releases page).
curl -L -o sloppyfocus.zip \
  https://github.com/catokolas/HS_ModulesContrib-sloppyfocus/releases/latest/download/sloppyfocus-<version>-macos-universal.zip

# 2. macOS may quarantine a downloaded .so; clear the flag so dlopen accepts it.
xattr -dr com.apple.quarantine sloppyfocus.zip 2>/dev/null || true

# 3. Unzip into ~/.hammerspoon. The archive's top-level is hs/, so this
#    lands at ~/.hammerspoon/hs/_ckol/sloppyfocus/.
unzip -o sloppyfocus.zip -d ~/.hammerspoon

# 4. Quit and relaunch Hammerspoon (Reload Config will NOT pick up a fresh .so).
#    Then verify in the Console:
#      require("hs._ckol.sloppyfocus")
```

If `dlopen` still complains about the quarantine after step 2, repeat the
`xattr` after step 3 on the unpacked `.so`:
`xattr -dr com.apple.quarantine ~/.hammerspoon/hs/_ckol/sloppyfocus`.

## Build & install from source

```bash
cd sloppyfocus
make install       # copies into ~/.hammerspoon/hs/_ckol/sloppyfocus/
# or for development:
make link          # symlinks instead, so future `make` picks up automatically
```

Then **quit and relaunch Hammerspoon** (Reload Config does not refresh native
modules — already-loaded `.so` files stay pinned in `package.loaded`).

To produce a release artifact (universal binary zip) yourself:

```bash
cd sloppyfocus
make dist VERSION=0.1     # → dist/sloppyfocus-0.1-macos-universal.zip
```

## Usage

```lua
local sloppy = require("hs._ckol.sloppyfocus")

-- Focus the window under the cursor without raising it:
local win = hs.window.windowsForApplication(app)[1]
sloppy.focusWithoutRaise(win)

-- When switching between two windows of the same app, also pass the
-- currently-focused window so SkyLight performs the extra deactivate/
-- activate dance it needs:
local current = hs.window.focusedWindow()
sloppy.focusWithoutRaise(target, current)
```

## Logging

This module emits no log output of its own and intentionally avoids
`NSLog` in `internal.m` — focus-without-raise runs on every mouseover
in the calling Spoon, so per-call logging would be very noisy. All
diagnostic output is the responsibility of the calling Spoon (e.g.
`FocusFollowsMouse.spoon` exposes a `logger` variable; see its README
for how to set the level).

For native-side debugging, build a debug copy and `printf`/`NSLog`
ad-hoc:

```bash
cd sloppyfocus
make clean && make DEBUG_CFLAGS="-g -O0"
# add NSLog(@"...") calls in internal.m, rebuild, quit & relaunch HS.
# Output lands in Console.app under the Hammerspoon process.
```

## API

### `focusWithoutRaise(win [, currentlyFocused]) -> boolean`

Gives keyboard focus to `win` without changing its position in the Z-order.

- `win` — an `hs.window`
- `currentlyFocused` — optional `hs.window` currently holding focus. Required
  only when switching between two windows of the same app (e.g. two iTerm
  windows); if omitted, same-app focus changes may appear to no-op.
- returns `true` on success, `false` if the SkyLight symbols couldn't be
  resolved, `GetProcessForPID` failed, or SLPS returned an error.

## Acknowledgments

- [AutoRaise](https://github.com/sbmpost/AutoRaise) — the focus-without-raise
  recipe, including the same-process dance, is a direct port of its logic.
- [yabai](https://github.com/koekeishiya/yabai) — the original
  `make_key_window` byte layout came from there.

## License

MIT — see [`LICENSE`](LICENSE) (sibling file at the repo root). Compatible
with the upstream projects this module derives from (AutoRaise and yabai are
both MIT).
