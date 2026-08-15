# Windhawk Taskbar Mod — Session Summary (2026-08-08)

**File:** `taskbar-centered-start-split-icons.wh.cpp`
**Mod ID:** `taskbar-centered-start-split-icons`
**Previous session:** `20260807_VSCode_Windhawk_TaskbarMod.md` (initial build + feature set)

Two pieces of work today: a bug fix reported from real overnight use, and two new
placement settings for pinned-but-not-running apps.

## 1. Bug fix: mod stayed inert after a cold boot

### Symptom

After turning the computer on in the morning, the mod was enabled in Windhawk but
had no visible effect — Start wasn't centered, icons weren't split. Disabling and
re-enabling the mod fixed it immediately, with no other change.

### Investigation

The "disable/re-enable fixes it" detail was the key clue: it pointed at something
resolved once, early, that a manual toggle re-triggers but a fresh boot doesn't.
`g_hTaskbarWnd` (the taskbar's own `HWND`, found once via `EnumWindows` in
`Wh_ModAfterInit`) was the only such value in the mod, and the per-element
positioning hook (`IUIElement_Arrange_Hook`) explicitly bailed out whenever it was
null:

```cpp
if (!g_inTaskbarArrangeOverride || g_unloading || !g_hTaskbarWnd) {
    return original();
}
```

If Windhawk injects into `explorer.exe` before `Shell_TrayWnd` has been created yet
— plausible right at boot, never on a manual toggle since the taskbar has been
running for a while by then — that one-shot lookup fails and `g_hTaskbarWnd` stays
null forever, silently disabling all positioning.

To confirm this rather than guess, `taskbar-start-button-position.wh.cpp` (the
published mod this one's XAML-hooking technique is already based on) was pulled
via `git clone` and checked for how it handles the same situation. Its equivalent
`Arrange` hook gate never checks a cached taskbar handle at all — it only needs one
to send an explicit relayout kick, not to run the actual positioning math. That
difference confirmed the root cause and pointed straight at the fix.

### Solution

Added `EnsureTaskbarWnd()`, a self-healing resolver that:

- Returns `g_hTaskbarWnd` immediately once it's known.
- Otherwise retries `FindCurrentProcessTaskbarWnd()` and, on success, starts the
  drag-follow `WinEventHook` (previously only started once in `Wh_ModAfterInit`).

Called once per `TaskbarCollapsibleLayoutXamlTraits_ArrangeOverride_Hook` pass
(the taskbar's own recurring layout pass, which fires continuously on its own
regardless of the mod's state) instead of once at init. This makes the mod
self-heal within moments of the taskbar actually existing, with no manual toggle
required. `Wh_ModAfterInit` now calls the same function instead of duplicating the
lookup.

**Status:** fix is in the file; not yet confirmed by the user across an actual
reboot.

## 2. New setting: `unresolvedAppsDefaultSide` gains a third option

`unresolvedAppsDefaultSide` (the fallback side for pinned-but-not-running apps not
matched by `leftApps`/`rightApps`) previously only offered `left`/`right`. Added a
third option, `contralateral-to-system-buttons`, which dynamically picks whichever
side Search/Task View/Widgets are *not* adjacent to.

Edge case handled: when those system buttons sit at the taskbar's far-left edge
(`systemButtonsPlacement: far-left`) rather than next to Start, that still counts
as "their side" for this purpose, so contralateral resolves to the right.

Implementation: a new `UnresolvedAppsDefaultSide` enum (distinct from the plain
`Side` enum used everywhere else) is resolved to an actual `Side` at
classification time via `ResolveUnresolvedAppsDefaultSide()`, since the right
answer depends on the live `systemButtonsPlacement`/`systemButtonsAdjacentSide`
settings rather than being fixed at load time.

## 3. New setting: `pinnedAppsAnchor`

### Clarifying the request

The initial ask — "let users choose where pinned apps go: left of Start, right of
Start, or contralateral to Search/Task View/Widgets" — sounded at first like it
might already be fully covered by `unresolvedAppsDefaultSide` (left/right) plus the
new contralateral option above. A clarifying question narrowed it to something
more specific: not *which side* pinned apps land on, but *where within that side*
they sit relative to running apps — today they always render at the outer edge
(farthest from Start), and the user wanted the option to anchor them next to Start
instead.

### Root cause of the current "always outer edge" behavior

In the distance-from-center ordering mode, every taskbar button gets an `orderKey`
used to rank same-side icons (smaller key = closer to Start). Running apps get a
real `orderKey` from their window's actual distance from screen-center. Apps with
no window to measure — pinned-not-running apps, and anything forced by
`leftApps`/`rightApps` — got a hardcoded `orderKey = +infinity`, which always loses
that comparison and pushes them to the outer edge.

### Solution

- New `pinnedAppsAnchor` setting (`outer-edge` default / `adjacent-to-start`).
- `PinnedAppOrderKey()` returns `+infinity` (unchanged default) or `-infinity`
  (adjacent-to-start) depending on the setting. Nothing beats `-infinity`, so those
  icons always end up innermost instead of outermost.
- Applied uniformly to all three "no window" classification paths in
  `ClassifyTaskListButton`: `leftApps` matches, `rightApps` matches, and the
  `unresolvedAppsDefaultSide` fallback.
- Only has an effect in `taskListOrder: distance-from-center` mode — `taskbar-order`
  mode doesn't rank icons by distance at all, so there's no outer-edge/adjacent-
  to-start distinction to apply it to. Documented in the setting's description
  rather than silently doing nothing.

## Full settings reference (as of today)

| Setting | Options | Purpose |
|---|---|---|
| `gapPx` | number (default 12) | Gap between Start and each flanking icon group |
| `systemButtonsPlacement` | `far-left` \| `adjacent-start` | Where Search/Task View/Widgets sit |
| `systemButtonsAdjacentSide` | `left` \| `right` | Which side of Start, if adjacent |
| `leftApps` / `rightApps` | comma-separated name fragments | Force specific apps to a side |
| `unresolvedAppsDefaultSide` | `left` \| `right` \| `contralateral-to-system-buttons` | Fallback side for unmatched/unresolvable apps |
| `taskListOrder` | `distance-from-center` \| `taskbar-order` | How same-side icons are ordered |
| `pinnedAppsAnchor` | `outer-edge` \| `adjacent-to-start` | Where pinned-not-running apps sit within their side (distance-from-center mode only) |

## Process notes

- The boot-inertness bug was diagnosed by comparison against a real, published
  mod's equivalent hook (cloned locally via `git clone`) rather than by guessing
  from symptoms — the same approach that resolved the "Combine taskbar buttons:
  Always" issue in the previous session.
- Both new settings started from ambiguous requests ("pinned apps," "left/right of
  start, or contralateral to widgets") that had more than one plausible reading.
  Each was narrowed with a short clarifying question before writing any code,
  rather than guessing and risking a wasted compile/test cycle — compiling and
  verifying a Windhawk mod requires the user's own manual test pass, so getting
  the interpretation right up front matters more here than in ordinary code
  changes.
- No version bump (`@version 0.1.0` unchanged) — consistent with prior sessions,
  which iterate in place without semantic versioning until the mod is considered
  stable enough to publish.
