# Changelog

Development history and design rationale for `taskbar-centered-start-split-icons.wh.cpp`. The mod itself carries only the comments that matter for a maintainer touching the code today; this file has the "how we got here" story that used to live inline as comments.

## Architecture

The mod hooks two undocumented pieces of the Windows 11 taskbar:

- `IUIElement::Arrange` (a process-wide vtable slot, taskbar.dll/Taskbar.View.dll's XAML tree) - overrides the X position of Start, Search/Task View/Widgets, and each task list button.
- `TaskbarCollapsibleLayoutXamlTraits::ArrangeOverride` - the taskbar's own top-level layout pass, used as the trigger to (re)compute a full layout plan once per pass, before any nested `Arrange` calls happen.

Positioning math lives in `RecomputeLayoutPlan`/`PlanTaskListButtons`; per-window HWND resolution (needed to classify which side of the screen a taskbar button's window is on) lives in a separate subsystem built around a "click-sentinel" technique borrowed from other Windhawk taskbar mods (`taskbar-volume-control-per-app`, `taskbar-reorder-right-drag`, `taskbar-start-button-position`): dispatch a real click through the taskbar's internal click handler and intercept it before it does anything, reading back the native window handle from the intercepted call.

## The crash-debugging arc (2026-08-12 through 2026-08-15)

The mod went through eight distinct crash root-causes before reaching a stable design, all sharing the same fault signature: `STATUS_STOWED_EXCEPTION` (`0xC000027B`) inside `Windows.UI.Xaml.dll`. This is WinUI's own internal "an exception was stowed somewhere and I'm failing the process for it" mechanism (`CXcpDispatcher::Tick -> DirectUI::ErrorHelper::ReportUnhandledError`), reported on a *later* composition tick than whatever actually threw - which meant a crash dump's stack trace, no matter how many times re-analyzed, only ever showed the generic bucket, never the real culprit. Each of the eight was a genuinely different bug that happened to produce the identical fault:

1. **`.as<T>()` vs `.try_as<T>()`** - a null-source cast during construction of a *secondary* monitor's XAML tree, in the `Arrange` hook (the only `.as<>()` call in the file at the time).
2. **Unguarded reentrant `UpdateLayout()`** - a WinEventHook-driven invalidate calling `UpdateLayout()` synchronously while a burst of ~340 events/sec was landing, occasionally reentering an already-active layout pass.
3. **Count-change self-correction reentrancy** - the `ArrangeOverride` hook's "button count changed, re-check" logic called a forced synchronous relayout from a point that could itself be nested inside a larger layout walk.
4. **Cross-monitor structural churn** - moving a window between monitors while Windows' "show taskbar apps on" setting wasn't "All taskbars" restructures which taskbar's repeater owns a button, not just its coordinates; a forced synchronous `UpdateLayout()` anywhere in that path was unsafe. This is the origin of the still-current rule: `InvalidateTaskbarLayout` never calls `UpdateLayout()`, only `InvalidateArrange()`/`InvalidateMeasure()`, letting the XAML dispatcher pick up the relayout on its own next tick.
5. **Wrong-taskbar arranging** - the process-wide `Arrange` hook was applying primary-monitor-relative math to elements that had moved into a *secondary* monitor's XAML tree. (Superseded later the same day - see below.)
6. **A self-inflicted repeat of #5** - the fix for #5 itself made an unmarshaled cross-apartment WinRT call from the wrong thread under specific timing.
7. **The real root cause, found via WinDbg** (`cdb -y ... -c "!analyze -v"` against `%LOCALAPPDATA%\CrashDumps\explorer.exe.<pid>.dmp`, which Windows captures automatically) - the click-sentinel HWND-resolution technique was being invoked *synchronously from inside `Arrange`*, exactly the moment a button could be mid-insertion into a repeater. This is why HWND resolution now runs entirely off a periodic timer (`ResolvePendingButtonHwnds`), never inline during layout - `Arrange`'s hook only ever reads a cache the timer already populated.
8. **A settling-window compression side effect** (see below) - fixed the same day as the architectural rewrite that closed the whole crash class for good.

**Lesson learned across this arc, worth restating:** once a crash dump's stack trace comes back identical twice, further `!analyze -v` runs are a dead end - what actually cracked case #7 was the user noticing the crash correlated with a specific Windows *setting* ("show taskbar apps on"), not a fourth dump. A behavioral discriminator beats a repeated dump once the bucket is generic.

## The settling-window mitigation, then its replacement (2026-08-14 to 2026-08-15)

Before the architectural rewrite below, the crash class from incident #7 was worked around with a "settling window": when the `ArrangeOverride` hook detected the task-list button count had changed, it would suppress all repeater-touching logic (falling back to native/frozen positions) for ~1 second, on the theory that a full second was enough for any in-flight structural churn to finish. This actually worked for crash-avoidance, but produced a cascade of its own cosmetic bugs, each fixed in turn:

- Start and the icons using inconsistent gating produced a visible overlap during the settling window (fixed by gating Start too).
- The recovery-after-settling relied on incidental subsequent events, so the taskbar could visibly "look disabled" for 30+ seconds after a legitimate change (fixed with a dedicated recovery-pending flag checked by the resolve timer).
- The settling detector's own synthetic zero-counts, read back on the *next* pass, looked like a second structural change and re-armed settling indefinitely (fixed by not feeding gated-pass counts back into the detector).
- Even once crash-free and cycle-free, falling back to native positions during settling was itself a visible "flash" on every single button-count change (not just cross-monitor moves) - opening any app would flash the whole taskbar to native layout and back. Fixed by holding each element's *last known good X* during settling instead of falling through to native, so a settling pass looks like a brief pause rather than a layout swap.

## The precomputed-plan rewrite (2026-08-15/16)

The first AI review round on the windhawk-mods submission made the case that the entire settling-window/recovery/re-arm system was treating a symptom, not the disease: the crash-prone operation was repeater traversal happening *inside* a nested `Arrange` call, and settling only ever changed the *timing* of that risk, never eliminated the unsafe operation itself.

The actual fix: compute the whole layout plan once per `ArrangeOverride` pass, entirely *before* calling into XAML's own layout (before any nested `Arrange` call happens at all), and make the per-element `Arrange` hook a pure map lookup with zero traversal. This is the architecture the file has today (`RecomputeLayoutPlan` -> `g_lastArrangedX` -> `IUIElement_Arrange_Hook`'s lookup), and it closed the crash class outright rather than mitigating it - the entire settling-window apparatus, and the `g_primaryXamlRootIdentity`/secondary-monitor-scoping mechanism from incident #5/#6, both became unnecessary and were deleted. Since a secondary-monitor element's `Arrange` call is never even part of the primary-only plan, it simply falls through to native positioning with no special-casing needed.

## HWND-resolution hardening (multiple rounds, 2026-08-17 through 2026-08-21)

The click-sentinel technique dispatches a *real* click, relying entirely on a hook (`CTaskListWnd_HandleClick_Hook`) intercepting it before it reaches the taskbar's actual click handler. Because this mod (unlike the reference mods it borrowed the technique from) fires the probe unattended from a background timer rather than only on a genuine user gesture, several rounds of hardening went into making sure a broken interception can never turn into Explorer spontaneously activating/launching windows on its own:

- A `TryGetItemFromContainer_...` failure vs. a `ReportClicked` miss are different signals; only the latter (a probe that actually reached the dispatch point with zero capture) is evidence about the interception mechanism itself.
- The confirmed/broken state is tracked **per resolution path** (individual-item vs. grouped-button), not as one shared flag - the two paths reach the taskbar's click handler through different internal call chains, so a Windows update could break only one of them, and a shared flag would let a still-working path mask a broken one. This mattered most for the grouped path, since a pinned-but-not-running app's group can never resolve and used to retry forever.
- A group with no running windows now short-circuits via an `ITaskGroup::IsRunning` check *before* ever dispatching a click at it, closing the "retried forever" case at the source rather than just detecting it after the fact.
- Latching a path "broken" requires a few consecutive unconfirmed misses (not one), since a single miss can also mean the window closed mid-probe or the item was mid-teardown during a taskbar rebuild - not necessarily a broken interception. An early false positive on the always-used individual-item path used to risk silently degrading every taskbar button to the default-side fallback for the rest of the session.
- Negative-result caching, identity-mismatch detection (an `ItemsRepeater` can rebind an already-realized element to a different item without destroying it), and capped exponential backoff (2s to 32s, with a 30-minute ceiling specifically to bound synthetic-click frequency against pinned-not-running apps) were all added incrementally as each was found to be needed.

## Notification/marshaling evolution

Early rounds used `RunFromWindowThread` (a `SetWindowsHookEx(WH_CALLWNDPROC)`-based marshal) for every cross-thread notification, which blocks the caller until the target thread's message loop processes it - expensive on hot paths like drag-follow invalidation, which can fire several times a second. A later round (following a maintainer-cited precedent, `taskbar-auto-hide-when-maximized.wh.cpp`) subclassed the taskbar window directly, letting the hot paths use a non-blocking `PostMessage` instead, with `RunFromWindowThread` kept only as a fallback for the rare case the subclass installation itself failed.

That subclass-based rewrite shipped once without ever running through a local compiler (no local Windhawk clang/WinRT toolchain was available that session), and broke CI: the subclass callback used comctl32's 6-parameter `SUBCLASSPROC` signature instead of Windhawk's actual 5-parameter one, plus `RemoveWindowSubclassFromAnyThread` needs no separate id parameter the way raw `SetWindowSubclass`/`RemoveWindowSubclass` do (a mismatch here left a dangling subclass across teardown, a crash-on-every-disable). A later session found and documented a working local `clang++ -fsyntax-only` invocation (Windhawk bundles its own MinGW/clang toolchain at `C:\Program Files\Windhawk\Compiler`), and every round since has run it on both `x86_64-w64-mingw32` and `aarch64-w64-mingw32` targets before shipping.

## Submission and review

Submitted to the official windhawk-mods catalog as [PR #5111](https://github.com/ramensoftware/windhawk-mods/pull/5111) on 2026-08-15, with an automated AI review loop (`/ai-review`, limited to 3 requests per 24 hours) iterating through 16+ rounds since. Recurring themes across rounds: exception-safety at raw ABI/callback boundaries (several `.try_as` vs `.as`, RAII guards for flags that must reset on every exit path, atomics for cross-thread-read globals), the click-sentinel hardening described above, and - not yet resolved as of this writing - whether the Start-centering half of this mod's feature set should eventually move into the existing `taskbar-start-button-position.wh.cpp` mod instead of staying duplicated.

## Known open items

- Multi-monitor: only the primary taskbar is positioned; every window's side is decided against the *primary* monitor's center line, so windows on a secondary monitor read as a fixed left/right regardless of where they sit on that monitor.
- A user-reported observation (2026-08-21): with "Combine taskbar buttons" set to "Always", multiple windows of the same app were observed not combining into one button. This mod has no code path that could cause it (grouping is a native decision made before any of this mod's hooks see a button) - not yet confirmed whether it reproduces with the mod disabled.
- The overlap-with-`taskbar-start-button-position` scope question, raised across several review rounds with an increasingly specific proposed resolution (fold Start-centering into that mod as an option), remains an open decision.
