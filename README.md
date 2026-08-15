# Centered Start button with position-split taskbar icons Mod for Windhawk

Pins the Windows/Start button to the exact horizontal center of the primary
monitor, regardless of how many taskbar icons are present. Running-app
taskbar buttons are split into two groups that flank the Start button:

- A button goes to the **left** group if its window's current on-screen
  position is left of screen-center.
- A button goes to the **right** group if its window is right of
  screen-center.
- Pinned-but-not-running apps have no window to check the position of, so
  they're classified by the `leftApps`/`rightApps` name lists below, falling
  back to `unresolvedAppsDefaultSide` if unlisted. Within whichever side
  they land on, `pinnedAppsAnchor` picks whether they sit at the far edge
  or right next to Start.

When you drag a window across the center line of the screen, its taskbar
button switches sides to follow it. Side-switching is driven by a global
window-location-change listener and is best-effort: it happens shortly
after a drag/move settles, not on every intermediate pixel of the drag.

Search, Task View and Widgets can either stay at the far left edge, or move
right next to Start on whichever side you prefer.

## Known limitations (please read before reporting issues)

- **Windows 11 only.** Windows 10's taskbar has no XAML layer to hook into.
- **Primary monitor only.** Screen-center math and window-side classification
  use the primary monitor. Taskbars on secondary displays are not specially
  handled by this mod (their icons keep the default layout Windows gives
  them) - the positioning hook explicitly checks which taskbar's XAML tree
  an element belongs to and leaves anything outside the primary's alone.
- **Undocumented internals.** This mod hooks private, unversioned classes
  inside `taskbar.dll` and `Taskbar.View.dll` (via symbols resolved from
  Microsoft's public symbol server at runtime, not hardcoded offsets). A
  Windows update can change these internals and break the mod until it's
  updated. If that happens, disable the mod rather than filing against
  explorer.exe crashing.
- The "resolve which HWND a taskbar button represents" step reuses a
  technique from other taskbar-reordering mods (synchronously reporting a
  sentinel "click" to the taskbar's internal click handler, which is
  intercepted before it does anything, to read back the window handle). It
  runs on a periodic timer rather than inline during layout, and Arrange
  only ever reads whatever the timer has already cached - running it
  synchronously from inside the taskbar's own layout pass was the
  confirmed cause of an explorer.exe crash (specifically when Windows'
  "show taskbar apps on" setting is anything other than "All taskbars",
  since that's when a window moving across monitors structurally adds/
  removes taskbar buttons rather than just repositioning them).

## Changelog

**Initial build (Aug 2026)**

- Start button pinned to the primary monitor's true center, independent of
  the number of taskbar icons.
- Running-app icons split left/right of Start by live window position, with
  drag-follow (a button switches sides shortly after its window crosses the
  center line).
- `taskListOrder`: icons ordered by distance from center, or by native
  taskbar order.
- Minimized windows keep their last known side instead of jumping (a
  minimized window's reported position is off-screen nonsense, so it's
  frozen at wherever it was before minimizing).
- `leftApps`/`rightApps` name-based overrides, plus
  `unresolvedAppsDefaultSide` for pinned-but-not-running apps with no
  window to classify by.
- `systemButtonsPlacement`/`systemButtonsAdjacentSide`: Search, Task View
  and Widgets can sit at the far-left edge or adjacent to Start.

**Feature follow-ups**

- Support for "Combine taskbar buttons: Always" (grouped icons bind to a
  different internal view model than individual windows, so this needed
  its own resolution path).
- Negative caching for HWND resolution, so a pinned-but-not-running app
  (which will never resolve to a window) isn't retried on every single
  layout pass.
- Self-healing taskbar-window lookup, so the mod doesn't stay permanently
  inert if Windhawk happens to inject before the taskbar itself exists yet
  (observed right after a fresh boot).
- `unresolvedAppsDefaultSide` gained a `contralateral-to-system-buttons`
  option.
- `pinnedAppsAnchor` setting: pinned-not-running apps can sit at the outer
  edge of their side, or right next to Start.

**Stability: cross-monitor moves**

Moving a window to a second monitor repeatedly crashed explorer.exe across
several iterations, each surfacing a different unsafe call path within the
same underlying class of bug: a synchronous WinRT/XAML call made while the
taskbar's own internal button list is mid-structural-mutation - reproducible
specifically when Windows' "show taskbar apps on" setting isn't "All
taskbars," since only then does a cross-monitor move add or remove taskbar
buttons rather than just repositioning them. Root-caused via WinDbg
crash-dump analysis after a few rounds of counter-based guessing didn't
fully close it. Fixes along the way:
- Corrected an unsafe cast that could throw while a fresh secondary-monitor
  taskbar tree was still under construction.
- Removed every forced-synchronous layout pass in favor of always deferring
  to the XAML dispatcher's own next tick.
- Scoped all positioning strictly to the primary taskbar's own XAML tree (a
  process-wide hook was also seeing, and mis-positioning, secondary-monitor
  taskbars).
- Moved taskbar-button HWND resolution off the live layout pass entirely and
  onto a periodic timer.
- Added a brief "settling window" after any detected taskbar button-count
  change, during which the mod holds off on the specific operations that
  were still unsafe mid-mutation.

**Settling-window polish**

Once crash-free, the settling window itself produced two follow-on cosmetic
issues, both fixed the same day:
- The Start button no longer keeps re-centering while its neighbors are held
  still during settling (previously caused a brief icons-over-Start overlap
  on every cross-monitor move).
- Fixed a bookkeeping bug that could make the settling window continuously
  re-arm itself, making the taskbar look permanently reverted until some
  unrelated event happened to nudge it back into place - now guaranteed to
  resolve shortly after the triggering change instead.
- Instead of snapping to Windows' native layout for the duration of the
  settling window, the mod now holds every icon at its last known position
  and only lets a genuinely new button show up unpositioned - much less
  visually distracting on ordinary events like opening a new app.

 ## Disclosures
  
I am not a software developer. The present mod was developed using the Claude Code extension in VS Code. I cannot verify the external integrity of this mod on other systems and do not take responsibility for issues that may arise for its use. This mod was created for my own interests and shared for targeted development by members of the Windhawk community.
