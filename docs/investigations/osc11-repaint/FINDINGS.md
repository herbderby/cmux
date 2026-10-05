# OSC 11 background sometimes not shown in a cmux pane

Source read: upstream cmux `b685a275c` (release 0.64.25, build 106, checked in
`cmux.xcodeproj/project.pbxproj`: `MARKETING_VERSION = 0.64.25`,
`CURRENT_PROJECT_VERSION = 106`), with its ghostty submodule at
`manaflow-ai/ghostty` `4a0e9e185`. All paths below are at those commits. Nothing
was run; cmux is a macOS app and this was a Linux session. "Read" means I saw
it in the code. "Inferred" means I reasoned it and did not confirm it.

## Conclusion

I did not find a single proven cause. I found one real defect that can produce
exactly the symptom, and I ruled out one hypothesis.

1. **Most likely, and a real defect: cmux throws away the OSC 11 color on any
   surface config change (a variant of hypothesis 3).** In cmux the visible
   terminal background is not drawn by ghostty; it is a plain AppKit layer
   color that cmux sets from a "color change" callback. When ghostty reports a
   surface config change (theme reload, appearance sync, font size change,
   settings change, Reload Configuration), cmux clears its stored OSC color and
   repaints the pane with the theme default. Ghostty itself keeps the OSC color
   across a config change. After any such reload, the pane shows cream while
   the terminal still thinks the background is green. (Read in code.) Whether a
   config change actually fired near Herb's sends is not known; the debug log
   described below will answer that.
2. **Possible: bytes from Claude Code landing inside the OSC (hypothesis 2).**
   Ghostty's parser silently drops a corrupted OSC 11 (read in code). Whether
   macOS can split a 12-byte `printf` write to a tty so that other output gets
   in between depends on the kernel tty code, which I did not read (inferred).
3. **Ruled out as stated: ghostty not marking the surface dirty (hypothesis 1).**
   cmux tells ghostty not to paint the default background at all, so a missing
   ghostty repaint cannot leave the wrong background on screen. On the cmux side
   the layer color is set directly with no "same as before" check. (Read in
   code.)

A pattern worth checking: in both incidents the green that failed was the
first OSC 11 the pane received, and the red that worked was the second. Every
explanation above fits that pattern only by chance. One cmux path does run
only on a pane's first OSC 11 (a lazily created "cutout" view, item 4.2 below),
but I found no way in the code for it to hide an opaque color. Open question 1
asks for a test that tells "first send fails" apart from "red helps".

## How OSC 11 travels, end to end

### 1. Ghostty parses the bytes (read)

- `ghostty/src/terminal/parse_table.zig:327-341`: inside an OSC string, bytes
  0x20-0xFF are appended to the OSC body (`osc_put`), most C0 bytes are
  ignored, BEL (0x07) ends it.
- `parse_table.zig:62-71` ("anywhere" transitions): CAN (0x18) and SUB (0x1A)
  abort to ground; ESC (0x1B) leaves the OSC and starts a new escape, which
  ends the OSC at that point.
- So if foreign bytes land inside `ESC ] 11 ; #EAF4E2 ESC \`:
  - printable text is appended to the color (`#EAF4E2abc…`), the color fails
    to parse and the command is dropped with no visible error;
  - an ESC ends the OSC early, so the color is truncated (for example `#EAF4`)
    and is usually invalid. (`#EAF` would parse as a valid three-digit color,
    so a different color could also appear.)

### 2. Ghostty updates terminal state and tells the app (read)

- `ghostty/src/termio/stream_handler.zig:1438-1466`: a "set background"
  request calls `terminal.colors.background.set(color)` and then always sends a
  `color_change` message to the surface. There is no "unchanged, skip" check.
- `stream_handler.zig:1492-1500`: OSC 111 (reset) clears the override and
  sends a `color_change` carrying the **default** color. The app cannot tell a
  reset from a set.
- `stream_handler.zig:221-237`: the surface mailbox push blocks until there is
  room; it drops messages only during a kitty-graphics replay
  (`kitty_replay_tracking`, set at `stream_handler.zig:801-805`), which does
  not apply here.
- `ghostty/src/Surface.zig:1339-1363`: on the app thread, `color_change`
  becomes `performAction(.color_change)` with kind `background`.

### 3. Ghostty does not paint the default background in cmux (read)

- cmux forces `macos-background-from-layer = true` into every config it builds
  (`Sources/GhosttyTerminalView.swift:1238-1251`, and the fallback config at
  `:964-969`).
- With that flag, ghostty's renderer sets the background alpha to zero
  (`ghostty/src/renderer/generic.zig:1750-1751`) and skips the fullscreen
  background fill (`generic.zig:1991-2003`). Cells with the default background
  get alpha 0 (`generic.zig:3403-3431`), so they are transparent.
- Result: what Herb sees as the pane background is a CALayer color owned by
  cmux, not anything ghostty draws.

### 4. cmux applies the color (read)

- `Sources/GhosttyTerminalView.swift:3390-3413` handles
  `GHOSTTY_ACTION_COLOR_CHANGE`. For a background change it schedules, on the
  main queue: `surfaceView.backgroundColor = newColor`, then
  `applySurfaceBackground()`, then `applyWindowBackgroundIfActive()`.
- `applySurfaceBackground()` (`:4173` onward) builds a
  `TerminalSurfaceBackgroundFillPlan`
  (`Packages/macOS/CmuxAppKitSupportUI/.../TerminalSurfaceBackgroundFillPlan.swift:59-89`).
  With a stored override, the owner is the pane's own host layer and
  `clearsSharedWindowBackdrop` is true, because
  `Workspace.usesWindowRootTerminalBackdrop()` always returns `true`
  (`Sources/Workspace.swift:3595-3597`).
- `GhosttySurfaceScrollView.setBackgroundColor` (`:10632-10646`) sets
  `backgroundView.layer.backgroundColor` inside a `CATransaction` with no
  comparison against the previous value. Core Animation commits it at the end
  of the main run loop turn.

**4.2 The first-override-only path (read).** `setBackgroundColor` calls
`synchronizeSharedBackdropCutout(visible:)` (`:10649-10658`). On a pane's
first OSC override it creates a view whose Core Image compositing filter
(`TerminalSharedBackdropCutoutFilter`, `:3587-3615`, a destination-out blend)
punches the pane's area out of the shared window backdrop
(`makeSharedBackdropCutoutView`, `:10664-10676`). Its comment says AppKit needs
`layerUsesCoreImageFilters` set before display, which is why it is created
lazily. This is the only code I found that runs on the first override and not
later. It sits below the colored layer, so with an opaque background it should
not matter (inferred). With `background-opacity` below 1 it decides what shows
through, and a pale green over cream could look like cream (inferred).

### 5. cmux clears the color on any surface config change (read)

- `Sources/GhosttyTerminalView.swift:3414-3427`, `GHOSTTY_ACTION_CONFIG_CHANGE`:

  ```swift
  DispatchQueue.main.async { [self] in
      if let staleOverride = surfaceView.backgroundColor {
          surfaceView.backgroundColor = nil
          ... logBackground("surface override cleared ... source=action.config_change.surface")
          surfaceView.applySurfaceBackground()
          surfaceView.applyWindowBackgroundIfActive()
      }
  }
  ```

- Ghostty sends that action at the end of every `Surface.updateConfig`
  (`ghostty/src/Surface.zig:2418-2423`).
- Ghostty keeps the OSC override across that same update:
  `ghostty/src/termio/Termio.zig:603` assigns only
  `terminal.colors.background.default`; the override field lives separately
  (`ghostty/src/terminal/color.zig:332-353`, `get()` returns
  `override orelse default`).
- cmux calls `ghostty_surface_update_config` from at least:
  the incremental config apply (`Sources/GhosttyApp+ConfigurationApply.swift:189-207`),
  surface reloads (`Sources/GhosttyApp+SurfaceConfigurationReload.swift:11-35`),
  and the surface `RELOAD_CONFIG` action that ghostty raises when a surface's
  light/dark scheme changes (`GhosttyTerminalView.swift:3446-3470`,
  `ghostty/src/Surface.zig:6829-6848`). Reload triggers include appearance sync
  (`GhosttyTerminalView.swift:2300-2330`, including a deferred sync drained
  later at `:2334`), settings changes (`:1066-1086`), font magnification
  (`AppDelegate.swift:14249-14262`), theme notifications
  (`AppDelegate.swift:19039-19071`), and the Reload Configuration menu item.
- The suppression flag `suppressGhosttyReloadActions` (`:2191-2199`) gates only
  `RELOAD_CONFIG` (`:2393-2400`), not `CONFIG_CHANGE`, so the clear still runs
  during cmux's own applies (read).
- I found no path where an OSC color change itself triggers a config reload.
  The only observer of `.ghosttySurfaceThemeDidChange`, which the color handler
  posts, is the mobile mirror (`Sources/Mobile/MobileTerminalRenderObserver.swift:93-102`),
  and it only schedules a theme resend to phones (read).
- cmux's shell integration does not send OSC 11 or 111 (searched
  `Resources/shell-integration` and `Resources/bin`; read).

## The hypotheses, ranked

| Rank | Hypothesis | What the code shows | Status |
|---|---|---|---|
| 1 | 3, as "cmux drops the color on config change" | Any surface config change clears the visible override while ghostty keeps it (§5). | Defect read in code. Whether it fired during Herb's sends is unknown. |
| 2 | 2, interleaved output | Parser drops a corrupted OSC silently (§1). | Parser read; tty write splitting inferred. |
| 3 | 4, first-override cutout | Code that runs only on the first override exists (§4.2). | Read; no failure mechanism found for opaque colors. |
| 4 | 1, no repaint / cached compare | Ghostty doesn't paint the default background in cmux; cmux sets the layer directly with no compare (§3, §4). | Ruled out as stated (read). |

## Reproduction recipe for Herb's Mac

### Setup: turn on the background log

The release build reads a user default. This needs a restart of cmux because
the flag is read once (`GhosttyTerminalView.swift:594-602`). The log goes to
`/tmp/cmux-bg.log` unless `CMUX_DEBUG_BG_LOG` is set (`:574-592`).

```sh
defaults write com.cmuxterm.app cmuxDebugBG -bool YES
# quit cmux fully, relaunch it
: > /tmp/cmux-bg.log
```

In each test pane, run `tty` before starting anything and note `/dev/ttysNNN`.
Set `T=/dev/ttysNNN` in the controlling terminal (a different pane or
Terminal.app).

What each log line means:

- `action event target=surface action=color_change` and
  `surface override set ... override=#EAF4E2`: the OSC reached cmux.
- `surface background applied ... source=surfaceOverride ... color=#EAF4E2`:
  cmux painted it.
- `surface override cleared ... cleared=#EAF4E2 source=action.config_change.surface`:
  cmux threw it away (hypothesis 3 variant).
- No `color_change` line for a send: the OSC never reached cmux (hypothesis 2,
  or the bytes were never written).

For each screenshot, sample a pixel in an empty area of the pane (Digital
Color Meter, or `sips`/Python on the PNG) and record the hex. Pale green
`#EAF4E2` is close to many creams, so eyeballing is not enough.

### Test A: clear-on-config-change (deterministic, checks rank 1)

1. Fresh pane, plain shell (no Claude Code). `printf '\033]11;#EAF4E2\033\\' > $T`.
2. Screenshot, sample pixel. Expect green.
3. In cmux, choose Reload Configuration from the app menu (or toggle macOS
   light/dark appearance).
4. Screenshot, sample pixel. **If it is now cream and the log shows
   `surface override cleared`, the defect in §5 is confirmed.**

### Test B: idle versus busy (separates hypothesis 2 from 1)

Run each case in 10 **fresh** panes, sending the green **once**:

- B1 idle: pane at a shell prompt, nothing running.
- B2 busy, controlled: in the pane run
  `while :; do printf '\033[H\033[2J'; seq 1 3000; done`, then send the green.
  This gives heavy output with many escape sequences, like a TUI.
- B3 Claude Code idle: Claude Code open, waiting for input.
- B4 Claude Code busy: while Claude Code is streaming a long answer.

For B2, stop the loop with Ctrl-C before taking the screenshot (the loop does
not change the background, so the color should persist; `clear` does not
reset OSC 11).

Record per send: pixel hex at +1 s and +10 s, and whether a `color_change`
line appeared. Readings:

- Missing `color_change` lines only in busy cases: hypothesis 2.
- `color_change` present but pane cream, with a later `override cleared`:
  rank 1.
- `color_change` present, `surface background applied ... #EAF4E2`, no clear,
  pane cream: a compositing problem (rank 3); report `background-opacity` too.
- Everything green in all 40: the trigger is something these tests lack; go to
  test D.

### Test C: drop rate under load (sharper test of hypothesis 2)

With the B2 loop running in the pane:

```sh
for i in $(seq 1 100); do
  if [ $((i % 2)) = 0 ]; then printf '\033]11;#EAF4E2\033\\' > $T
  else printf '\033]11;#FF0000\033\\' > $T; fi
  sleep 0.05
done
grep -c 'action=color_change' /tmp/cmux-bg.log
```

Clear the log first. Fewer than 100 `color_change` events (after subtracting
any from other panes) means sends were lost on the way in.

### Test D: first send versus "red helps"

Fresh pane with Claude Code, idle. Send green once. If it fails, send the
**same green again** (no red). If the second green works, the pattern is "first
OSC 11 in a pane fails", not "a different color first helps". Check the log
for the first send in either case.

### Afterwards

```sh
defaults delete com.cmuxterm.app cmuxDebugBG
```

## Proposed fix (patch only, not built)

`proposed-fix.patch` in this directory, against `b685a275c`, changes
`Sources/GhosttyTerminalView.swift` only:

1. On `CONFIG_CHANGE`, keep `surfaceView.backgroundColor` and just repaint, so
   cmux's visible color matches ghostty's terminal state (which keeps the
   override, §5).
2. On a background `COLOR_CHANGE`, store `nil` when the new color equals the
   current default. Ghostty reports OSC 111 as a change to the default color
   (§2); without this, a later theme switch would leave the old default pinned
   as an "override". I infer this is why the clear in step 1 was added.

Caveats:

- An OSC 11 that sets exactly the theme default is treated as "no override",
  so after a theme change that pane follows the new theme. That matches what
  it showed before the change.
- A cleaner fix is to have ghostty mark resets in `ghostty_action_color_change_s`
  (a fork change), so cmux does not compare colors.
- This addresses rank 1 only. It does nothing for hypothesis 2. If test C shows
  drops, the fix is outside cmux: send the OSC through the program that owns
  the pane, or accept retries.
- Not compiled; no tests added. A regression test would drive a
  `CONFIG_CHANGE` after a `COLOR_CHANGE` and assert the host-layer color.

## Open questions

1. In each failed case, was the green the pane's first OSC 11 since it opened?
   Test D answers whether "first send fails" is the real pattern.
2. Was the failed green checked by pixel sampling, or by eye? `#EAF4E2` is
   close to cream.
3. What are Herb's `background` and `background-opacity` settings? Below 1,
   the first-override cutout view (§4.2) decides what shows through.
4. Did anything reload config near the failed sends: an appearance change,
   wake from sleep, a settings edit, a theme switch, or cmux startup work after
   the reboot? The log answers this.
5. Does Claude Code ever send OSC 111 (reset background) or OSC 11 itself? A
   reset from Claude Code would arrive as a change to cream (§2) and look
   exactly like this bug. Checkable by recording Claude Code's output with
   `script` and searching for `]111` and `]11;`.
6. Can macOS split a single 12-byte write to a tty when the output queue is
   full? I believe the BSD tty layer can sleep mid-write once the queue passes
   its high-water mark, letting another writer in, but I did not read XNU.
   Test C measures it directly.
7. Why did 0.64.22 drop the BEL form while the ST form worked? The ghostty
   parser at this commit treats BEL as a terminator (`parse_table.zig:341`).
   I did not trace the 0.64.22 source.
