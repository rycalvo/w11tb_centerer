# Windhawk Taskbar Mod — Session Summary

**Date:** 2026-08-07
**File:** `taskbar-centered-start-split-icons.wh.cpp`
**Mod ID:** `taskbar-centered-start-split-icons`
**Platform:** Windows 11 only (requires [Windhawk](https://windhawk.net/))

## Goal

Build a custom Windhawk mod that:
1. Pins the Windows/Start button to the true horizontal center of the primary monitor.
2. Splits running-app taskbar icons into two groups flanking Start, based on which side of the screen each window currently occupies — with icons following windows live as they're moved, minimized, closed, etc.

No published Windhawk mod does this (confirmed via research at the start of the session), so it was built from scratch by reverse-engineering undocumented Windows 11 taskbar internals, grounded throughout in real technique pulled from other published mods rather than guessed symbols.

## Final feature set

- **Start button centering** — pinned to the exact horizontal center of the primary monitor's work area, independent of icon count, by hooking the taskbar's internal XAML layout pass.
- **Position-based icon split** — each running app's taskbar button is classified left or right of Start based on its window's live on-screen position relative to screen-center.
- **Live drag-follow** — dragging a window across the center line moves its icon to the other side, driven by a global `WinEventHook` on window location changes.
- **Distance-based ordering** (`taskListOrder` setting) — by default, icons are ordered by how close their window is to screen-center, so the nearest window's icon sits closest to Start; can be switched back to native taskbar order.
- **Minimize-safe** — a minimized window's icon freezes at its last known side/position rather than jumping (Windows reports nonsense off-screen coordinates for minimized windows via `GetWindowRect`).
- **Pinned/unresolvable app handling** (`leftApps`, `rightApps`, `unresolvedAppsDefaultSide` settings) — apps with no running window (nothing to check position on) are classified by matching their accessible name against user-supplied lists, falling back to a configurable default side.
- **Search / Task View / Widgets placement** (`systemButtonsPlacement`, `systemButtonsAdjacentSide` settings) — these three buttons can sit at the taskbar's far-left edge (default, matches current native-ish behavior) or immediately adjacent to one side of Start.
- **"Combine taskbar buttons: Always" support** — grouped/combined icons (multiple windows of one app collapsed into a single badge button) resolve and classify correctly, via a second resolution path alongside the individual-window path.
- **Configurable gap** (`gapPx`) between Start and each flanking icon group.

## Technical architecture

The mod hooks two DLLs loaded into `explorer.exe`:

### `taskbar.dll` (native COM/C++ layer)
- `CTaskBand`/`CSecondaryTaskBand` vtables + `GetTaskbarHost` — used to locate the taskbar's `XamlRoot` from its `HWND`, for forcing relayouts.
- `CTaskListWnd::HandleClick` — hooked to intercept a "sentinel" click value, used as a side-channel to extract native task item/group pointers (see resolution chain below).
- `CWindowTaskItem::GetWindow`, `CImmersiveTaskItem::GetWindow` (+ its `ITaskItem` vftable) — resolve a native task item to an `HWND`.
- `TaskItem::ReportClicked`, `TaskGroup::ReportClicked` (WindowsUdk implementation classes) — triggering these synchronously (with the sentinel value) causes `HandleClick` to fire with the real native pointers, which the hook captures instead of performing a real click.
- `CTaskGroup::GetNumItems` — not called normally; its address is called with a fake `this` (an array of pointers to sequential integers) to discover, via the returned value, the byte offset of the group's internal task-items array member, without needing the actual (undocumented, unstable) struct layout.

### `Taskbar.View.dll` (WinRT/XAML layer)
- `TaskbarCollapsibleLayout::ArrangeOverride` — the taskbar's own layout pass; hooked to run the mod's positioning logic on every relayout.
- A lower-level `IUIElement::Arrange` hook (installed via a discovered vtable slot, not a symbol) intercepts every individual element's arrange call within that pass, so Start/Search/Task View/Widgets/each app icon can have its position overridden individually.
- `TryGetItemFromContainer<TaskListWindowViewModel>` + `TaskListWindowViewModel::get_TaskItem` — resolve an individual (ungrouped) taskbar button to its underlying window.
- `TryGetItemFromContainer<TaskListGroupViewModel>` + `TaskListGroupViewModel::IsMultiWindow` — resolve a *grouped* taskbar button (combine-mode) to its underlying task group. Grouped buttons bind to a completely different view-model type than individual ones, which was the root cause of combine-mode not working initially.

### HWND resolution chain

Two paths, tried in order, per taskbar button:

1. **Individual path:** `TaskListButton` (XAML) → `TaskListWindowViewModel` → WindowsUdk `ITaskItem` → (sentinel click) → native `ITaskItem` → `HWND`.
2. **Group path** (fallback, needed for "Always combine"): `TaskListButton` (XAML) → `TaskListGroupViewModel` → (a *second* sentinel, piggybacked on `IsMultiWindow`'s internal call to `ITaskGroup::IsRunning`, which is hooked the same way) → WindowsUdk `ITaskGroup` → (sentinel click again) → native `ITaskGroup` → its internal task-items array (via the offset-probing trick above) → first item's `HWND`, used as the group's representative window.

Both paths are read-only reuses of techniques from taskbar-reordering and per-app-volume-control mods — nothing here performs an actual click or reorders anything natively.

Results are cached per XAML element (keyed by its ABI pointer), including a short negative-cache (2s TTL) for failures — added after logs showed pinned-but-not-running apps (whose task group legitimately has zero windows) were retrying the entire multi-sentinel resolution chain on every single arrange pass.

## Settings reference

| Setting | Options | Purpose |
|---|---|---|
| `gapPx` | number (default 12) | Gap between Start and each flanking icon group |
| `systemButtonsPlacement` | `far-left` \| `adjacent-start` | Where Search/Task View/Widgets sit |
| `systemButtonsAdjacentSide` | `left` \| `right` | Which side of Start, if adjacent |
| `leftApps` / `rightApps` | comma-separated name fragments | Force specific apps to a side (mainly for pinned-not-running apps) |
| `unresolvedAppsDefaultSide` | `left` \| `right` \| `contralateral-to-system-buttons` | Fallback for anything unmatched/unresolvable |
| `pinnedAppsAnchor` | `outer-edge` \| `adjacent-to-start` | Where pinned-not-running apps sit within their side (distance-from-center mode only) |
| `taskListOrder` | `distance-from-center` \| `taskbar-order` | How icons on the same side are ordered relative to each other |

## Debugging journey (chronological)

1. **Initial build** — compiled and enabled, but no visible effect. Root cause: `HookTaskbarViewDllSymbols` was failing outright (one bad symbol aborted the whole batch), so `Wh_ModInit` returned `FALSE` and the mod never actually initialized.
2. **Marked symbols optional** with per-symbol resolved/MISSING logging, revealing exactly which ones failed instead of one opaque `FAILED`.
3. **`TaskItem::ReportClicked` missing** — traced to being hooked from the wrong module (`Taskbar.View.dll` instead of `taskbar.dll`), confirmed by pulling the real mod source directly via `curl` rather than trusting a lossy `WebFetch` summary.
4. **Start button centered correctly**, confirming the core layout-hook mechanism — but app icons didn't split or follow windows yet.
5. **Live drag-follow not firing** — `WinEventHook`'s callback requires the *registering* thread to pump messages; `Wh_ModAfterInit` doesn't reliably run on the taskbar's actual UI thread. Fixed by marshaling registration onto the taskbar's own thread.
6. **New pins not appearing** until mod disable/re-enable — the taskbar's virtualization bookkeeping didn't reconcile with the mod's position overrides for newly-realized elements. Fixed by detecting button-count changes between passes and forcing one extra relayout.
7. **Minimized windows jumping to one side** — `GetWindowRect` returns nonsense off-screen coordinates for minimized windows. Fixed by freezing a window's last known classification while minimized.
8. **Ordering request** — added distance-from-screen-center ordering as a settings-driven alternative to native taskbar order.
9. **"Always combine" mode: icons appeared but all defaulted to one side**, unordered — diagnostic counters showed 100% of resolution attempts failing at the very first step. Root cause: grouped buttons bind to `TaskListGroupViewModel`, not `TaskListWindowViewModel`. Solved by researching (via `git clone` of the real Windhawk mods repo) and porting a complete second resolution path used by an unrelated mod (per-app volume control) that already handles both cases.
10. **Linker error** (`undefined symbol: DPA_GetPtr`) — the group-path resolution needs `comctl32.dll`'s dynamic pointer array functions; fixed by adding `-lcomctl32` to `@compilerOptions`.
11. **Negative-cache added** after logs showed a permanently-unresolvable button (a pinned-not-running app, whose task group legitimately has zero windows) retrying its full resolution chain on every single arrange pass.
12. **Mod stayed inert after a cold boot** (worked fine after a manual disable/re-enable) — `g_hTaskbarWnd` was resolved exactly once, in `Wh_ModAfterInit`, with no retry. If Windhawk injects into `explorer.exe` before `Shell_TrayWnd` exists yet (only plausible right at boot; never on a manual toggle, since the taskbar is already fully up by then), that one-shot lookup fails and `g_hTaskbarWnd` stays null forever, which silently disables all positioning. Confirmed against `taskbar-start-button-position.wh.cpp` (which this mod's XAML-hooking is based on) that its own `Arrange` hook never gates on a cached taskbar handle at all. Fixed by adding `EnsureTaskbarWnd()`, retried once per `ArrangeOverride` pass, so the mod self-heals as soon as the taskbar window appears instead of requiring a manual toggle.
13. **Added a third option to `unresolvedAppsDefaultSide`** (`contralateral-to-system-buttons`): pinned-but-not-running apps can now be placed on whichever side is opposite Search/Task View/Widgets, in addition to the existing fixed left/right choices. When those system buttons sit at the taskbar's far-left edge rather than adjacent to Start, that still counts as "their side" for this purpose, so contralateral resolves to the right. Implemented as a new `UnresolvedAppsDefaultSide` enum (distinct from the plain `Side` enum) resolved to an actual `Side` at classification time via `ResolveUnresolvedAppsDefaultSide()`, since the right answer depends on the live `systemButtonsPlacement`/`systemButtonsAdjacentSide` settings rather than being fixed at load time.
14. **Added `pinnedAppsAnchor` setting** (`outer-edge` default / `adjacent-to-start`): controls where pinned-but-not-running apps (and leftApps/rightApps-forced apps, which also have no window to rank by distance) sit within their side, relative to the running apps there. Previously they always sorted to the outer edge, hardcoded via `orderKey = +infinity` in the distance-from-center sort. `PinnedAppOrderKey()` now returns `-infinity` instead when the setting is `adjacent-to-start`, since the sort treats smaller orderKey as closer to Start - nothing beats `-infinity`, so they end up innermost. Only has an effect in `distance-from-center` ordering mode; `taskbar-order` mode doesn't rank by distance at all, so there's no outer-edge/adjacent-to-start distinction there.

## Known limitations

- **Windows 11 only** — Windows 10's taskbar has no XAML layer to hook into.
- **Primary monitor only** — screen-center math and window classification use the primary monitor; taskbars on secondary displays keep the default Windows layout.
- **No manual per-icon reordering** — only the automatic distance-based (or native) ordering exists; there's no way to pin a specific icon to a specific position within its side.
- **Undocumented internals** — all of this hooks private, unversioned classes resolved via Microsoft's public symbol server at runtime (not hardcoded offsets), but a future Windows update could still change these internals and break the mod.
- **HWND resolution runs inside the taskbar's own layout pass**, a different context than the technique it's adapted from (which normally runs from a UI event handler). No instability has been observed, but if a XAML "layout cycle" exception or similar ever appears, this is the first place to look — the fix would be moving resolution out of the `Arrange` hook entirely (e.g. onto a timer) and having `Arrange` only ever read from the cache.
