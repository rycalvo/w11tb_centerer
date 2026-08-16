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

![Window left of screen-center](https://raw.githubusercontent.com/rycalvo/w11tb_centerer/main/screenshots/leftside.png) \
_The crosshair-marked window sits left of screen-center - its taskbar icon
follows, landing left of Start._

![Window right of screen-center](https://raw.githubusercontent.com/rycalvo/w11tb_centerer/main/screenshots/rightdefault.png) \
_The same window moved right of screen-center - its icon moves with it,
to the right of Start._

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
  them) - the mod's positioning plan is only ever computed on the primary
  taskbar's own thread, so secondary-monitor elements are simply never
  included in it and fall through to Windows' native positioning.
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
- **Taskbar buttons can disappear when a display is deactivated** (via
  Settings, unplugging, or a third-party display on/off tool) - if
  "Taskbar behaviors > When using multiple displays, show my taskbar apps
  on" isn't set to "All taskbars", Windows itself can drop a window's
  taskbar button entirely until you switch to that window (e.g. via
  Alt+Tab), rather than migrating it to the remaining taskbar. Reactivating
  a display doesn't trigger this. This is native Explorer/taskbar.dll
  behavior upstream of anything the mod hooks into, not something this mod
  causes or can work around.

## Disclosures

I am not a software developer. The present mod was developed using the Claude Code extension in VS Code. This mod was created for my own interests and shared for targeted development by members of the Windhawk community.
