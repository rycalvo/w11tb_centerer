# RATIONALE.md

Design rationale, history, and edge-case reasoning that used to live inline
in `taskbar-centered-start-split-icons.wh.cpp`'s comments. The source now
keeps only "what this does" plus load-bearing constraints; anything about
*why*, prior incidents, or subtle edge cases that were cut out during a
comment-trimming pass live here instead, organized in the same order as the
file's own section headers. Comments in the source that say "see
RATIONALE.md" point back to a heading below.

## Settings

**Pinned-app ordering.** `PinnedAppsAnchor`/`PinnedAppOrderKey`/
`ResolveUnresolvedAppsDefaultSide` govern where a pinned-but-not-running
app's icon sits, since it has no window to measure a real position from.
`ResolveUnresolvedAppsDefaultSide`'s "contralateral" mode mirrors how
Search/Task View/Widgets place themselves: opposite
`systemButtonsAdjacentSide` when they sit next to Start, or the right if
they're pinned at the taskbar's far-left edge (still their "side" for this
purpose, since that edge is unambiguously left of everything else).
`PinnedAppOrderKey` returns +/-infinity rather than a real distance so
`PlanTaskListButtons`' distance-from-center sort (smaller orderKey = closer
to Start) always puts these innermost or outermost with nothing able to
beat the infinity.

**`gapPx` clamped to >= 0**: a negative value from the settings UI would
pull the flanking icon groups toward Start instead of away from it.

**`LoadSettingsFromStore` vs `g_settings`**: kept separate so
`Wh_ModSettingsChanged` can read every setting (no XAML/COM dependency, any
thread) on its own calling thread, and marshal only the final struct
assignment onto the taskbar thread via `ApplyLoadedSettings`.

## Globals / threading

**Never traverse the taskbar's XAML tree from inside a nested
`IUIElement::Arrange` call.** XAML's layout system can be
mid-structural-mutation of the same repeater there —
`STATUS_STOWED_EXCEPTION` was observed when a window moves across monitors
while Windows' "show taskbar apps on" setting isn't "All taskbars," since
that structurally adds/removes taskbar buttons. `RecomputeLayoutPlan` does
the entire tree traversal once per `ArrangeOverride` pass, up front, writing
every element's target X into `g_lastArrangedX`, so `IUIElement_Arrange_Hook`
itself is a pure map lookup with no tree access at all.

**`g_hTaskbarWnd` is `atomic<HWND>`, not plain `HWND`**: written from
`EnsureTaskbarWnd` (the taskbar thread, or `Wh_ModAfterInit`'s thread at
startup) and read from the dedicated WinEventHook thread (`WinEventProc`,
`InvalidateTaskbarLayout`, `ButtonHwndResolveTimerProc`). A plain `HWND`
here was an oversight caught while auditing this file, not a considered
decision — `g_taskbarWndSubclassed` needed the identical treatment for the
identical reason. Beyond the formal data race, a stale read that passes a
null-check right before `EnsureTaskbarWnd` clears it to `nullptr` has a real
consequence: `PostMessage(nullptr, ...)` is *not* a no-op — it posts to the
**calling thread's own queue** and returns `TRUE`, silently misdirecting the
message instead of failing loudly. Every caller with more than one read of
this value snapshots it into a local first (`InvalidateTaskbarLayout`,
`ButtonHwndResolveTimerProc`, `ApplyLoadedSettings`, `RecomputeLayoutPlan`)
so all reads in one call agree, even if a concurrent write lands
mid-function.

**`g_taskbarWndSubclassed`**: this is the SOLE way `InvalidateTaskbarLayout`
/ `ButtonHwndResolveTimerProc` / `ApplyLoadedSettings` reach the taskbar
thread. There is no fallback marshal if the subclass never installs
(`SetWindowSubclassFromAnyThread` failing is rare — see `EnsureTaskbarWnd`'s
own log line). If it never installs, live drag-follow, HWND resolution, and
live settings updates are simply unavailable until a later
`EnsureTaskbarWnd` call succeeds. Core centering/splitting is unaffected
either way, since that's computed synchronously inside `RecomputeLayoutPlan`
on whatever `ArrangeOverride` passes XAML triggers on its own regardless of
the subclass.

**`g_lastDragFollowInvalidate`** is hoisted out of what used to be a
function-local static in `WinEventProc`, specifically so
`DragFollowTrailingTimerProc` can update it too: a trailing-timer fire is
itself an invalidate, so the next `WinEventProc` event needs to see it as
"now," not still throttle against whatever raw event last landed outside
the 150ms window before the trailing timer took over.

**`g_lastShowEventArm`**: leading-edge throttle mirroring the
LOCATIONCHANGE throttle. A top-level window becoming visible fires
`EVENT_OBJECT_SHOW` just as often system-wide as a move fires
LOCATIONCHANGE, and a single `arm(0)` call is enough to catch every button
currently pending regardless of how many SHOW events land in the same
burst. Unlike drag-follow, there's no "final position" that specifically
needs the trailing event, so no trailing timer is needed for SHOW.

## XamlRoot resolution (`XamlRootFromTaskbarHostSharedPtr`)

The shared_ptr's ref-count block (`taskbarHostSharedPtr[1]`) must be
released on every path, not just a fully successful call — this runs on
every `ArrangeOverride` pass, so a leaked reference here is one per pass for
the life of the process; a `DecrefGuard` RAII wrapper handles that. The
top-level null check only rules out *both* slots being null — slot 0 can
still be null while slot 1 isn't, and slot 0 is dereferenced further down,
so there's a second bail-out after the guard is already constructed (so [1]
still gets released even on that path).

The XAML element pointer's offset inside `TaskbarHost` isn't exposed by any
symbol, so it's read out of the prologue of a neighboring function known to
access it at a fixed offset. `#if defined(_M_X64)` also covers ARM64 in
practice — explorer.exe is a predefined shell process, so this mod is built
and loaded natively as ARM64 there too, and the two architectures need
separate opcode patterns. The code bails out on a pattern mismatch rather
than proceeding with a guessed offset, since a wrong offset means
dereferencing whatever happens to live there and calling a virtual method
through it.

`GetTaskbarXamlRoot`'s null check on `taskBand`: null while the taskband
window exists but its extra data isn't populated yet (e.g. mid-creation
after a taskbar recreate). This runs on every dirty arrange pass and from a
periodic timer, so skipping here matters — a null deref is a raw access
violation, not a C++ exception, so no try/catch in any caller can contain
it.

## HWND resolution (click-sentinel technique)

**Overview.** Two different call chains, depending on whether "Combine
taskbar buttons" groups multiple windows onto one button:

- Individual (ungrouped) button: `TaskListButton` (XAML) ->
  `TaskListWindowViewModel` -> WindowsUdk `ITaskItem` -> a sentinel "click"
  through the taskbar's own click handler, intercepted before it acts on it
  -> native `ITaskItem` -> HWND.
- Grouped button ("Combine taskbar buttons: Always" — each button can
  represent multiple windows of one app, and isn't bound to a
  `TaskListWindowViewModel` at all, so the individual path fails at its very
  first step): `TaskListButton` (XAML) -> `TaskListGroupViewModel` -> a
  second sentinel, piggybacked on `TaskListGroupViewModel::IsMultiWindow`'s
  internal call to `ITaskGroup::IsRunning`, intercepted the same way ->
  WindowsUdk `ITaskGroup` -> another sentinel click, this time on the group
  -> native `ITaskGroup` -> its internal task-items array (located by
  exploiting `CTaskGroup::GetNumItems`' known-trivial implementation — see
  `GetTaskItemsArray`) -> first item's HWND, used as the group's
  representative position.

Both techniques are the same ones `taskbar-reordering` and
`per-app-volume-control` use to map a button back to a window/windows; here
they're reused read-only, purely to find out where a window currently is on
screen.

**Why probing unattended is riskier here than in reference mods.** The
click-sentinel resolution technique dispatches a REAL click through
`TaskItem::ReportClicked`/`TaskGroup::ReportClicked`, relying entirely on
`CTaskListWnd_HandleClick_Hook` recognizing the sentinel and swallowing it
before the taskbar's real `HandleClick` ever runs. Reference mods using the
same technique only fire it in response to an actual user gesture on one
specific button, so a broken interception there just means the user's own
click happens twice — annoying but self-limiting. This mod instead fires it
unattended, from a background timer, against every button it hasn't
resolved yet. If a future Windows update changes `ReportClicked`'s internal
call path so it stops reaching the hook, every one of those probes silently
becomes a genuine click — for the buttons actually retried
(pinned-but-not-running apps), that means Explorer spontaneously launching
them on a timer, forever, with no user action and no visible sign anything
is wrong (an unintercepted probe looks identical to an ordinary resolution
failure).

**Why the broken-latch is per-path, and requires several misses.** The
interception either works on this Windows build or it doesn't — it isn't a
per-button thing. It's latched per path (item vs. group) rather than one
shared flag because the two paths reach `CTaskListWnd::HandleClick` through
different internal call chains (`TaskItem::ReportClicked` vs.
`TaskGroup::ReportClicked`), so a Windows update could break only one of
them. A shared flag would let a still-working item-path confirmation mask a
broken group path — which matters most because the group path is the one a
pinned-but-not-running app's button keeps retrying. Each `*Broken` latch
permanently gates its own path in `ResolveHwndFromTaskListButton`: once set,
that path never calls `ReportClicked` again, rather than retrying a
mechanism now known to dispatch real clicks.

A single unconfirmed miss is NOT proof the interception is broken — a probe
can also fail to reach `HandleClick` because the window closed between
resolving the task item and dispatching the click, `ReportClicked` failed
internally, or the item was mid-teardown during a taskbar rebuild. Latching
a whole path dead on the first miss of a session risked exactly that: an
early unlucky miss on the item path (the common path, used by every
ungrouped button) would silently degrade every button on the taskbar to
`unresolvedAppsDefaultSide` for the rest of the session, with only a log
line as the symptom. `kClickSentinelMissesBeforeBroken` requires a few
misses (still a bounded cost if the mechanism really is broken) before
actually latching.

This bound only holds pre-confirmation: `NoteUnconfirmedClickSentinelMiss`
returns immediately once a path is confirmed, by design — a
post-confirmation miss is routinely innocent (see above), so counting it
would risk latching a working path dead over an unrelated timing hiccup.
So "at most `kClickSentinelMissesBeforeBroken` real clicks per path, per
session" is the guarantee *before* first confirmation, not a session-wide
cap. Also note `ResolveHwndFromTaskListButton` tries the item path first
and falls through to the group path on a miss, so one unresolvable button
can burn a probe on BOTH paths in a single attempt — the worst case before
both latches trip is `2 * kClickSentinelMissesBeforeBroken` (6, at the
current constant) dispatched clicks, not
`kClickSentinelMissesBeforeBroken`.

**`g_clickSentinelProbingGroup`**: which path's probe is in flight on this
thread when a sentinel click might land in `CTaskListWnd_HandleClick_Hook`
— the hook only sees the click itself, not which resolve function
dispatched it, so this is what lets it credit the right path's `*Confirmed`
flag. Set immediately before dispatching a probe, alongside the existing
reset-before/read-after `g_clickSentinel_TaskItem`/`_TaskGroup` pattern.

**`.exchange` instead of plain assignment** on the confirmed flags: purely
so the confirmation log fires exactly once, the first time each path is
ever confirmed — giving a concrete, observable answer (useful for the PR
record and future debugging) to "does the group path actually get
exercised/intercepted on this build," which resolve-stats alone can't
distinguish from the item path.

**`ScopedCaptureTaskGroup`** is RAII, not a plain set/clear pair: if the
`IsMultiWindow` call between them ever exited non-locally, a plain
set/clear would leave `g_captureTaskGroup` stuck true, and
`ITaskGroup_IsRunning_Hook` would then answer every subsequent *real*
`IsRunning` call with a hardcoded `false` instead of forwarding to
`Original` — a wrong answer to a question the taskbar itself uses for real
decisions (running indicators, click behavior), for the rest of the
session. Same reasoning as `ScopedArrangeOverrideFlag` elsewhere in this
file.

**`CTaskGroup::GetNumItems` self-reporting probe.** Its entire body is just
`return DPA_GetPtrCount(this->taskItemsArray);` — i.e. reading one `int` off
`this` at a fixed-but-undocumented offset. Calling it with a fake `this`
that's actually an array of pointers-to-sequential-ints turns that read
into a self-reporting probe: whatever offset it reads becomes the returned
value, revealing the real offset without needing the struct layout.
Adapted from the per-app volume control mod, which needs the same task
group -> item list access for an unrelated reason.
`kTaskItemsArrayProbeSize` is shared with `GetTaskItemsArray`'s bounds
check below — the probe can never legitimately return an offset at or past
this size.

**`GetTaskItemsArray`'s offset-0 check**: offset 0 is the vtable slot, so
the probe can never legitimately land there — either `GetNumItems`'s
implementation stopped being the trivial form this technique relies on, or
the probe array was too small to reach the real offset. Either way,
trusting an out-of-range offset means dereferencing whatever happens to be
there and handing it to `DPA_GetPtrCount` as if it were a real array — a
wild read on a background timer, not a user gesture.

**`ResolveStats`**: diagnostics-only counter of how often the HWND
resolution chain succeeds vs fails, combined across both the individual and
grouped-button paths. `LayoutPlanStats`' own resolved/total count already
gives the steady-state health signal; this mainly confirms the chain is
running at all. A stage-by-stage breakdown (which specific step failed) was
useful while first bringing this technique up against a new Windows build —
worth re-adding locally for that kind of investigation rather than carrying
it permanently.

**Why `CTaskListWnd_HandleClick_Original` is checked explicitly** in both
`ResolveHwndFromIndividualTaskItem` and `ResolveHwndFromTaskGroup`: it's
what proves the interception hook is actually installed. It's optional in
`HookTaskbarDllSymbols`, so on a build where it didn't resolve,
`CTaskListWnd_HandleClick_Hook` never gets installed and the `ReportClicked`
call below would be a genuine click on a live window rather than an
intercepted sentinel. Checked here specifically (not just implied by the
other symbols) since this is the one way the probe can be unsafe that's
knowable up front, rather than only discoverable via
`NoteUnconfirmedClickSentinelMiss`'s runtime miss-counting.

**Why `GetTaskItemsArrayOffset()` is validated before dispatching a click,
not after** (the round-23 fix in `ResolveHwndFromTaskGroup`):
`GetTaskItemsArrayOffset()` is a memoized, side-effect-free probe — safe to
validate up front. Without this, a build where `CTaskGroup::GetNumItems`
stopped being the trivial form the offset probe relies on would still
dispatch a click on every single group probe (the sentinel interception
itself is unaffected, so the click-sentinel latch never trips — it's
`GetTaskItemsArray`'s own bounds check further down that always fails
instead), turning every group resolution attempt into a pure-waste
`ReportClicked` into the taskbar's click machinery, forever, with nothing
to bound it.

**The `-1` pointer adjustment** before calling
`TaskListGroupViewModel_IsMultiWindow_Original`: `IsMultiWindow`'s
implementation happens to call `ITaskGroup::IsRunning` internally, which is
hooked to capture its `this` (the native WindowsUdk task group) instead of
really answering the question. The `-1` adjusts from the interface pointer
`QueryInterface` handed back to the adjacent vtable `IsMultiWindow` actually
needs — a fixed ABI detail of this object, not a magic number specific to
this mod.

**Bailing out on `IsRunning() == false` before dispatching a group
click**: a group with no running windows (a pinned-but-not-running app) can
never yield an HWND — its task items array is legitimately empty, so
`GetTaskItemsArray`'s check always fails for it anyway — and it's exactly
the group that would otherwise get re-probed forever at the backoff ceiling
for the life of the session. Bailing out before dispatching a click at all
avoids that. `IsRunning` is a sibling method on the same interface already
captured above; the "consume" calling convention it was hooked under
(`ITaskGroup_IsRunning_Hook`'s `*(void**)pThis` capture) expects a pointer
to a variable holding the interface pointer, not the interface pointer
itself — hence `&windowsUdkTaskGroup`, matching how it was captured.

## Screen-position / window classification

**Minimized-window freeze**: a minimized window's `GetWindowRect` returns a
nonsense off-screen position (classically around `(-32000,-32000)`), which
would otherwise always classify it as "left." Instead,
`g_lastKnownWindowClassification` freezes at the last known classification
from before it was minimized. Not pruned on window close in the same
function — see the note on `GetButtonHwnd`'s cache and
`ResolvePendingButtonHwnds`' pruning pass for why that's an acceptable
tradeoff (a separate pruning pass keyed by HWND liveness handles it).

**No prior classification** (a window minimized before this mod ever saw
it — e.g. an app that auto-launches straight to a minimized/tray state,
common among startup apps right after login): `GetWindowPlacement`'s
`rcNormalPosition` still reports the window's restored position while
minimized (unlike `GetWindowRect`, which is why the freeze branch exists at
all — it returns nonsense off-screen coordinates for a minimized window),
so this can classify it correctly on first sight instead of permanently
defaulting.

**`workOffsetX` adjustment**: `rcNormalPosition` is in workspace coordinates
(relative to `rcWork`, which excludes the taskbar/appbars), while
`screenCenterX` is computed from `rcMonitor`. These are identical on a
normal bottom-docked taskbar (`rcWork.left == rcMonitor.left`), but a
left-docked appbar shifts `rcWork.left` right of `rcMonitor.left` — without
this offset, a window near the boundary could classify to the wrong side.

**Classification priority and the accessible-name skip**: explicit user
override by app name, then live window position, then the configured
default (mainly hit by pinned apps that aren't running and weren't listed
in `leftApps`/`rightApps`). The accessible-name lookup (an
`AutomationProperties::GetName` call plus a lowercasing string copy) is
skipped entirely when there's nothing to match it against —
`leftApps`/`rightApps` are both empty by default, and this runs for every
task-list button on every single `ArrangeOverride` pass.

**`SystemButtonContentWidth`**: Search, Task View and Widgets can each be
individually hidden/shown using a negative-margin collapse trick —
`ActualWidth()` includes that margin, so it never drops below the collapsed
width and grows on every layout pass once a transition starts. This reads
the content child's `DesiredSize` instead (matching
`taskbar-start-button-position.wh.cpp`'s fix for the same button types),
falling back to `ActualWidth()` if no child is realized yet.

Deliberately NOT used for Start or task list buttons: Start is never
hidden/shown, so it was never exposed to this bug (and swapping its width
source anyway understated it — see the `RecomputeLayoutPlan` Start-width
note below); task list buttons have no evidence of the same mechanism and
are this file's hottest path, so paying for the extra indirection there
would be pure cost.

## Layout (`PlanTaskListButtons` / `RecomputeLayoutPlan`)

**`ChildInfo`**: a repeater child paired with its `SystemButton`
classification, computed once per child per `RecomputeLayoutPlan` pass and
reused across the Start-finding, cluster-width, and system-button-placement
loops rather than re-deriving `IdentifySystemButton` (a
`winrt::get_class_name` call) separately in each.

**`g_lastLeftSystemClusterWidth`/`g_lastRightSystemClusterWidth`**:
computed unconditionally every `RecomputeLayoutPlan` pass so an empty
cluster (all three system buttons hidden) reads as genuinely 0 rather than
keeping a stale nonzero value that would permanently reserve dead space.

**`PlanTaskListButtons` overall shape**: computes every task list button's
target X in one O(n) pass, classifying each button exactly once. Every
entry is written into `outPlan` and stays in this mod's own coordinate
system — it never falls through to native Arrange, since Start is forced to
true center regardless and mixing forced- and native-position elements
would produce a visible overlap. Instead, an overflowing side's inter-icon
spacing compresses (icons stay full width) to fit within
`leftBoundLocal`/`rightBoundLocal` — the taskbar's left edge past the
system-button cluster, and the system tray's left edge, respectively.

Each side is walked innermost-first from Start's own edge outward, using
each icon's *unscaled* width for its own placement (only the running
reference point advances by the scaled amount) — this is what guarantees
the innermost icon's edge never drifts into Start under compression.
Scaling the placement itself looks equivalent at scale=1 but breaks that
guarantee; don't reintroduce it.

**Sort tie-breaking** in DistanceFromCenter mode: ties (e.g. multiple
pinned/overridden buttons, which all share the same +/-infinity `orderKey`)
break on taskbar index instead of an ABI-pointer value — equally stable
frame to frame, but a predictable order instead of an arbitrary one.

**TaskbarOrder mode's `left` reversal**: preserves taskbar order, read
left-to-right across the whole split layout the way native taskbar order
reads. On the left side, the earliest entry sits at the outer (leftmost)
edge and later entries sit progressively closer to Start; on the right side
it's the opposite — the earliest entry sits innermost (right next to Start)
and later entries sit progressively farther out. Together these reproduce
native left-to-right order across the whole taskbar once Start is inserted
in the middle. `left`/`right` are already in taskbar order (`entries` was
built that way); the right side's walk (accumulating outward from Start)
already puts its earliest entry innermost, but the left side needs
reversing first — walking it in original taskbar order would put the
*earliest* entry innermost instead, backwards from what preserves reading
order there (earliest belongs at the outer edge on the left side).

**Compression scale math**: scale <1 compresses inter-icon spacing (not
each icon's own rendered width) so the outermost icon's own edge never
passes `leftBoundLocal`/`rightBoundLocal`, at the cost of icons overlapping
each other once a side is too full for natural spacing. A side with plenty
of room keeps scale at 1. The outermost icon's own width
(`left.back()`/`right.back()` — the walk loop always processes them last)
is excluded from both the total and the available space fed into the ratio:
that icon is placed at full, unscaled width with nothing beyond it to
compress against, so folding its width into the compressible pool would
double-count it and let its own edge overshoot the bound by
`width*(1-scale)`.

**The zero-width placement skip (and the round-23 fix to it)**: each icon's
own placement uses its unscaled width (`x - entry->width`, not
`x - entry->width * leftScale`), so the innermost icon's edge lands exactly
at `leftInnerX`/`rightInnerX` regardless of scale — only `x` itself (the
reference point carried to the next icon) advances by the scaled amount.
Scaling the placement itself instead would drift the innermost icon into
Start as compression increases.

A just-realized button reports `ActualWidth() == 0` for one pass (only the
*previous* arrange pass ever sets it). The original (round 20) guard
checked `entry->width > 0` to skip giving such a button a plan entry that
pass — but `entry->width` is `FullFootprintWidth`, which includes
`Margin()`, and margin comes from style rather than layout, so it's
essentially never actually zero. That guard was therefore close to a no-op.
The round-23 fix checks `entry->element.ActualWidth() > 0` directly instead
(matching the pattern the Start branch already used), which actually
detects the brand-new, zero-width case.

Giving a zero-width button a plan entry would place it exactly on top of
its neighbor for one frame. Leaving it out of `outPlan` entirely instead
falls through to native positioning for that one pass (the same fallback
every element this mod doesn't plan gets) — and since it's then also
missing from `g_lastArrangedX`, `RecomputeLayoutPlan`'s own staleness check
(the hash-lookup miss, not even the child-count backstop) forces a real
recompute on the very next pass once its real width is available, so this
is a one-frame artifact, not a persistent one. Doesn't affect `x`'s advance
either way — a zero-width entry doesn't move the reference point regardless
of whether it gets a plan entry.

**`g_planDirty`**: set whenever something might have changed that
`g_lastArrangedX` doesn't reflect yet — a window moving, a button's
HWND/side resolving, the `ArrangeOverride` hook's button-count-change check,
a settings change, or mod startup — each setting it at the point that
specific change actually becomes visible on the taskbar thread (see
`InvalidateTaskbarLayout`'s own note below for why it can't be set any
earlier, at the calling thread's call site). Starts `true` so the first
`ArrangeOverride` pass always computes a real plan. `RecomputeLayoutPlan`
clears it only after a genuinely successful recompute — left set on an
exception so a later pass retries rather than freezing on a broken plan.
Every read and write now happens on the taskbar thread only, so a plain
`bool` would be sufficient; kept `atomic` as low-risk insurance rather than
downgrading it while touching this area for an unrelated fix. This flag
skips `RecomputeLayoutPlan`'s full traversal (a taskbar.dll vtable scan, a
`VisualTreeHelper` walk, a classification per child) on the very common
passes where nothing this mod cares about changed — a pure short-circuit
around already-correct plan-computation logic, not a rewrite of it.

**`kMaxPlanStalenessMs` staleness backstop**: some real triggers for a
layout change (Search/Task View/Widgets visibility, taskbar geometry/DPI/
monitor changes, a button appearing/disappearing between two dirty passes
with nothing else invalidating in between) aren't guaranteed to call
`InvalidateTaskbarLayout`. Rather than track every such trigger
individually, `RecomputeLayoutPlan` forces a real recompute at least this
often regardless of `g_planDirty`, bounding how long the plan can disagree
with reality instead of only reacting to known triggers. This bound only
holds while `ArrangeOverride` keeps getting called at least this often — a
change landing in a skipped pass on an otherwise-idle desktop (e.g. a
pin/unpin) could sit stale indefinitely otherwise.

The staleness-check itself does a hash lookup first, `IsTaskListButton`
(`winrt::get_class_name` — a `GetRuntimeClassName` round trip plus an
`HSTRING` allocation) second, short-circuited via `&&` — every element the
plan covers (Start and the system buttons too, not just task list buttons)
is already a key in `g_lastArrangedX`, so the common "nothing changed" case
never pays for a single class-name lookup. `IsTaskListButton` only runs at
all on a hash-lookup miss, i.e. a genuinely new child. A child
*disappearing* (unpin, app closed, or a system button's visibility
changing) adds nothing new to `g_lastArrangedX`'s coverage — every
remaining live child is still a key in it — so the hash-lookup loop alone
can't catch that direction; the count comparison against `g_planChildCount`
(every child kind, not just task list buttons) catches it instead, for free,
off the same walk.

**Start-width read (bare `ActualWidth()`, not `SystemButtonContentWidth`)**:
Start is never hidden/shown the way Search/Task View/Widgets are, so it was
never exposed to `ActualWidth()`'s collapse-margin growth problem (see
`SystemButtonContentWidth` above) — swapping to the content child's
`DesiredSize()` there instead would understate Start's true width, since
Start's own visual-tree shape doesn't match what that technique was
validated against. `g_lastStartWidth` only updates when `ActualWidth()` is
actually positive — `ActualWidth()` still reflects the previous arrange
pass and is 0 for a just-realized element, so a freshly (re)created Start
button keeps the last known-good width instead of collapsing the whole
taskbar around a zero-width Start for one pass.

**Subtracting `Margin().Left` from Start's target X**: the X handed to
`Arrange` sets the layout slot's left edge; XAML then insets the rendered
content by the element's own `Margin.Left`. Without subtracting it, Start's
rendered icon would center at `startCenterX + Margin.Left` instead of true
screen-center — a no-op when the margin happens to be 0, but wrong
otherwise.

**`RecomputeLayoutPlan`'s exception path**: `g_lastArrangedX` is left as
whatever the last successful pass produced —
`IUIElement_Arrange_Hook`'s lookup-or-fall-through handles a stale/
incomplete plan exactly like it already handles a brand-new not-yet-planned
element. `g_planDirty` deliberately stays `true` here (unlike the success
path), since this pass didn't actually produce a plan reflecting current
state, so a later pass should retry rather than treat this as "up to date"
and skip forever.

**`ArrangeOverride` hook's button-count self-correction**: the button count
can change without any window moving (new pin, app launched/closed). If
`RecomputeLayoutPlan`'s repeater walk ran before XAML had realized a
just-inserted button yet, this pass's plan has no entry for it and it
renders at its native position for one pass. Self-correcting by
invalidating whenever the observed count changes, and arming an immediate
HWND-resolve attempt, matters because a newly-inserted button is exactly
what `ResolvePendingButtonHwnds`' cache-only view (`NextResolveDelayMs`)
can't see coming on its own.

## XAML hooks (`EnsureTaskbarWnd`, `GetCachedTaskbarRepeater`,
`ResolvePendingButtonHwnds`)

**Why `EnsureTaskbarWnd` self-heals on every pass instead of a one-shot
`Wh_ModAfterInit` lookup**: `g_hTaskbarWnd` was originally resolved once, in
`Wh_ModAfterInit`. But if Windhawk injects into explorer.exe before
`Shell_TrayWnd` has been created yet — observed after a fresh boot, never
after a manual mod disable/re-enable since the taskbar is already fully up
by then — that one-shot lookup fails and `g_hTaskbarWnd` stays null forever,
which silently disables all positioning (`IUIElement_Arrange_Hook` and the
position math both require it). Retrying once per `ArrangeOverride` pass
instead lets the mod self-heal as soon as the taskbar exists, rather than
needing a manual toggle to re-run `Wh_ModAfterInit`. Shell_TrayWnd can in
principle be recreated without explorer.exe itself restarting (not
something this mod's own code causes, but not something it can rule out
either) — without the `IsWindow` staleness check at the top, a stale
`g_hTaskbarWnd` would silently stop all positioning forever with no
self-healing path.

**Why the subclass-install block was moved out of the one-shot resolve
block (round-23 fix — this is the bug the reviewer found in round 22's
`RunFromWindowThread` removal)**: the original code returned early
(`if (g_hTaskbarWnd) { return g_hTaskbarWnd; }`) before ever reaching the
subclass-install attempt, so the install only ran on the single pass that
first resolved `g_hTaskbarWnd`. If `SetWindowSubclassFromAnyThread` failed
on that one attempt — rare, but possible — the subclass would never be
retried again for the life of the process, since every subsequent call hit
the early return first. That silently and *permanently* disabled live
drag-follow, HWND resolution, and live settings updates (core
centering/splitting still worked, since that's computed synchronously). The
fix restructures this so the install is attempted on every call, gated only
on `!g_taskbarWndSubclassed && !g_unloading`, so a failed attempt gets
retried on the very next `ArrangeOverride` pass instead of being permanent.
This is cheap: once `g_hTaskbarWnd` is resolved, this runs on the taskbar's
own thread, so `SetWindowSubclassFromAnyThread` takes its same-thread fast
path (no actual marshal) on every call where the subclass is already
installed.

**The `g_unloading` re-check immediately after a successful install, and
the crash it closes**: `Wh_ModBeforeUninit`'s own subclass-removal pass only
runs once, gated by the same `g_taskbarWndSubclassed.exchange(false)` latch
— if it already ran with `g_hTaskbarWnd` still null (reachable: right after
a `Shell_TrayWnd` recreate, or a fresh-boot startup where the one-shot
lookup failed, is exactly when `EnsureTaskbarWnd` can still be resolving
this for the first time as unload begins), it can never run again. A
subclass installed here *after* that point would stay wired into
`Shell_TrayWnd` with no later removal call, and Windhawk unmaps this
module's code shortly after `Wh_ModUninit` returns — a use-after-free the
next time that window procedure runs. Re-checking `g_unloading` immediately
after install and undoing it right there closes that window.
`StartWinEventHook` needs no equivalent recheck since its own start/stop
pair is already serialized by `g_winEventThreadMutex`/
`g_winEventThreadStopped`.

**Why a newly-successful subclass install kicks the resolve timer and a
relayout**: with no fallback marshal, a subclass that installs successfully
on a *later* call (e.g. after an earlier attempt failed and the taskbar has
since recreated) needs an explicit kick — `ButtonHwndResolveTimerProc`'s own
self-rearming stops entirely while unsubclassed, so nothing else would
restart it. Harmless to call unconditionally on the very first successful
resolve too, alongside `Wh_ModAfterInit`'s own `InvalidateTaskbarLayout()`
call right after `EnsureTaskbarWnd` returns.

**`GetCachedTaskbarRepeater`**: caches the resolved repeater across calls
instead of every caller redoing `GetTaskbarXamlRoot`'s resolution chain
(taskband HWND -> vtable scan -> `TaskbarHost::FrameHeight` prologue parse
-> `QueryInterface`) plus `FindTaskbarFrameRepeater`'s three-level tree walk
from scratch — this used to run independently in `RecomputeLayoutPlan`,
`InvalidateTaskbarLayout`, and `ResolvePendingButtonHwnds` on every single
call. A `weak_ref` alone isn't a reliable "still live" signal: the
repeater's own layout, `ItemsSourceView`, and the animation machinery can
each hold a reference of their own, so if any of those outlives the
taskbar's tree even briefly (a taskbar recreate, a DPI/monitor change), the
`weak_ref` still resolves to a now-detached element — one with a null
`XamlRoot`, since that's only ever non-null while actually attached to a
tree. `GetCachedTaskbarRepeater` checks that explicitly rather than
trusting the `weak_ref` by itself. On a detached hit, every caller
eventually dereferences `XamlRoot()` (or relies on the repeater actually
being the live one) anyway, so returning it would just move the null deref
further down the call chain rather than avoiding it — clearing the weak ref
and falling through to a fresh resolution is strictly better.

**`ResolvePendingButtonHwnds`'s `g_unloading` gate, in detail**: the
click-sentinel probe only stays safe while `CTaskListWnd_HandleClick_Hook`
is installed to intercept it, and that hook (like all of this mod's hooks)
can be gone before `Wh_ModUninit` actually stops the timer that calls this
function. Without the gate, a probe landing in that window reaches the
taskbar's real `HandleClick` with a garbage `LauncherOptions` pointer (the
sentinel string reinterpreted as a vtable) — an access violation plus a
genuinely dispatched click. `g_unloading` is set at the top of
`Wh_ModBeforeUninit`, before the hooks are removed, so this closes the
window regardless of exactly when the timer's next tick lands. There's also
a secondary, defense-in-depth reentrancy guard against running while nested
inside an active Arrange pass on this thread (not the primary protection).

**The reentrancy guard is RAII, not plain set/clear**: this function runs
dispatched from the posted `ResolveButtonHwndsMsg` via
`TaskbarWndSubclassProc`, and it mutates `g_buttonHwndCache`/
`g_lastKnownWindowClassification` with erase-while-iterating prune loops. It
calls into taskbar.dll and WinRT, either of which could pump messages — if
another posted `ResolveButtonHwndsMsg` gets dispatched reentrantly while a
call is already in progress, a nested call's inserts would invalidate the
outer loop's iterator: undefined behavior, not just wasted work. RAII
(rather than a plain set/clear) is needed because this function has several
early returns a plain clear at the end would never reach.

**Why `ScheduleNextResolveTick` computes the delay in its own destructor
rather than in `ButtonHwndResolveTimerProc`**: it needs to run on every
return path, always on this thread, right after this pass's own cache
reads/writes are done. Computing it from `ButtonHwndResolveTimerProc`
instead would read `g_buttonHwndCache` concurrently with this function's
own insert/erase calls on the taskbar thread — a use-after-free.
`NextResolveDelayMs`'s `INFINITE` answer is only trustworthy once a pass
has actually confirmed the live button set — an empty `g_buttonHwndCache`
from a pass that bailed out before enumerating anything (every early-return
guard, or no repeater yet) looks identical to a genuinely empty taskbar
otherwise, and the timer would then never be armed again except by
incidental luck. `enumerated` tracks whether the walk actually ran to
completion; the destructor falls back to `kIdleResolveTickMs` instead of
trusting `INFINITE` when it didn't.

**Raw Win32 callback boundary try/catch**: `ResolvePendingButtonHwnds` runs
dispatched via `TaskbarWndSubclassProc`, a raw Win32 callback boundary with
no C++/WinRT exception translation on the other side, same as
`IUIElement_Arrange_Hook`/`RecomputeLayoutPlan`/
`PerformTaskbarLayoutInvalidate`. Every WinRT property access inside
(`Content()`, `ItemsSourceView()`, `get_class_name`, `GetName`, `XamlRoot()`)
can throw `winrt::hresult_error`, most likely exactly when the taskbar tree
is being recreated — an uncaught throw here crosses the boundary and
fail-fasts explorer.exe. Same reasoning applies to `IUIElement_Arrange_Hook`
(a raw ABI boundary — a process-wide vtable slot) and to
`TaskbarCollapsibleLayoutXamlTraits_ArrangeOverride_Hook`.

**The `needsResolve`/`anyChanged` bookkeeping in the resolve walk**: a
brand-new button in the live set — even if it fails to resolve an HWND
right away (e.g. a just-pinned app with no window yet) — still needs
`anyChanged = true`, since its mere appearance means `g_lastArrangedX` has
no entry for it and the plan needs rebuilding regardless of whether
`ResolveAndCacheButtonHwnd`'s own HWND-changed check fires. Identity
mismatch (checked on both the resolved and negatively-cached branches, not
just the resolved one) means `ItemsRepeater` rebound this element to a
different item since it was last resolved — a rebind onto a
negatively-cached element (e.g. drag-reordering a pinned-not-running app
past a running one) is just as real a rebind, and without checking there
too it would inherit the old entry's accumulated backoff instead of
resolving. For an already-resolved entry, the old HWND is still a perfectly
valid window, just no longer this element's, so `IsWindow()` alone can't
catch a rebind — identity has to be compared too.

The `forceResolve` bypass: a negatively-cached element's own
`consecutiveFailures` keeps accumulating for as long as its button exists
(which, for a pinned-but-not-running app, is the entire session) — by the
time it's actually launched, the backoff could be minutes away, silently
defeating `EVENT_OBJECT_SHOW`'s whole purpose of triggering an immediate
resolve. It's bounded to `consecutiveFailures < kMaxForcedRetryFailures` —
see the `g_forceResolveUnresolved` entry below for why an unconditional
force was a real bug, not just belt-and-suspenders.

**Pruning `g_buttonHwndCache`**: XAML routinely destroys and recreates
`TaskListButton`s (unpin, app close, virtualization), and the allocator can
reuse a destroyed element's address for a later, unrelated one — without
pruning, that new element would silently inherit the destroyed one's cached
HWND (the cache is keyed by raw ABI pointer with no reference held, so
there's no way to detect this other than checking against a fresh
enumeration like this one). `g_lastArrangedX` doesn't need the same
treatment — it's rebuilt from scratch by `RecomputeLayoutPlan` on every
genuine recompute (bounded to `kMaxPlanStalenessMs`, not immediate — see
`g_planDirty` above), so a stale entry there can never outlive that.

**Pruning `g_lastKnownWindowClassification`**: same idea one level up — it's
keyed by HWND, which Windows also recycles, and it's only ever consulted
for a minimized window's frozen side. Without pruning, a brand-new window
could inherit a closed window's stale classification.

**Rebuilding `g_resolvedHwnds`**: unconditionally, not just when
`anyChanged`, since it's cheap and needs to reflect this pass's pruning even
when nothing else about the plan changed. This is the set `WinEventProc`
reads (cross-thread, hence the separate mutex) to filter drag-follow's
`EVENT_OBJECT_LOCATIONCHANGE` events — a window that never resolved to a
taskbar button can't change this mod's layout, so there's no reason to pay
for a relayout on its account.

**Why `ResolveAndCacheButtonHwnd` is ONLY ever called from the resolve
timer**, never from `GetButtonHwnd` or an active Arrange pass: the
click-sentinel technique interacts with the taskbar's internal
click-handling machinery, and running it while a button is being
structurally inserted/removed from an `ItemsRepeater`'s data source reaches
`STATUS_STOWED_EXCEPTION` — confirmed via crash-dump analysis — which
happens whenever Windows' "show taskbar apps on" setting isn't "All
taskbars" and a window moves across monitors.

**`g_forceResolveUnresolved`, and why unconditional forcing was a real
bug**: set by `WinEventProc`'s `EVENT_OBJECT_SHOW` branch and the
`ArrangeOverride` hook's count-change check — both call
`ArmButtonHwndResolveTimer(0)` to make the next resolve pass run
immediately, but arming the timer sooner doesn't by itself bypass a
negatively-cached entry's own backoff gate (that backoff can be
long-lived — see `ButtonHwndCacheEntry` below). Consumed once per pass in
`ResolvePendingButtonHwnds` to force a negatively-cached entry to retry
regardless of backoff, but only up to `kMaxForcedRetryFailures` consecutive
failures. A group with no running windows can never yield an HWND and bails
at `ResolveHwndFromTaskGroup`'s `IsRunning` check before ever dispatching a
click — the original, unconditional version of this force assumed THAT is
the only reason a negatively-cached entry keeps failing (a pinned app that
just launched, still catching up to its own `EVENT_OBJECT_SHOW`). But a
RUNNING app's button can also fail to resolve for reasons that don't bail
out early — `ReportClicked` failing internally, a task item mid-teardown,
`GetTaskItemsArray` coming back empty — and for those, forcing every retry
unconditionally meant dispatching a real `ReportClicked` on both paths on
every forced pass, indefinitely, at up to the ~7Hz `EVENT_OBJECT_SHOW` can
arm this at — exactly the runaway-real-clicks scenario the backoff schedule
exists to bound. Capping the force-bypass to entries that have only failed
a few times keeps the "pinned app just launched" fast path intact (that
case is still failing 0 times when `EVENT_OBJECT_SHOW` first fires) while
letting the normal backoff schedule take back over for an entry that's
failed repeatedly, the same way it already would with no force at all.

## `ButtonHwndCacheEntry`

Per-button HWND cache, keyed by the XAML element's ABI pointer. Avoids
re-running the resolution chain every pass, and negatively caches failures
(a pinned-but-not-running app's task group legitimately has zero windows,
so resolution fails until it's actually launched).

`consecutiveFailures` drives capped exponential backoff (2s up to the
30-minute `kResolveBackoffCeilingMs`) rather than a fixed retry: the
resolution chain ends in a synthetic click against the taskbar's real click
handler, and `ItemsRepeater` recycles the same element/cache entry for a
given index rather than creating a new one, so a hard stop would
permanently break side-following for a pinned app that's later launched. A
negatively-cached element's failures accumulate for as long as its button
exists, so a long-idle session's backoff can be close to the ceiling by the
time the app is actually launched — `g_forceResolveUnresolved` exists
specifically to bypass that when there's real evidence (a window just
appeared) that a retry is worth trying regardless of the schedule.

`identity` (the button's accessible name at resolve time) catches a
different case: `ItemsRepeater` can rebind an already-realized element to a
different item (e.g. a drag-reorder) without destroying it. The old HWND
stays valid, just no longer this element's, so an `IsWindow()` check alone
can't detect it — `ResolvePendingButtonHwnds` compares identity on every
check (both the resolved and negatively-cached branches) and forces a
re-resolve on mismatch.

## Live drag-follow

**`g_inInvalidateTaskbarLayout` reentrancy guard, in detail**: guards
against reentering the invalidate body itself on the taskbar's own thread.
`WinEventProc` can fire in rapid bursts (thousands of raw events within
seconds have been observed while something on screen is spamming
`EVENT_OBJECT_LOCATIONCHANGE`), each one posting another
`InvalidateTaskbarLayoutMsg` dispatched via `TaskbarWndSubclassProc` — a
nested WinRT/taskbar.dll call inside this function's own body
(`GetCachedTaskbarRepeater`, `InvalidateArrange`/`InvalidateMeasure`) could
pump messages and let a second posted message land reentrantly on this same
thread before the first call returns.

**Why `InvalidateTaskbarLayout`/`PerformTaskbarLayoutInvalidate` never call
`UpdateLayout()`**: forcing a synchronous `UpdateLayout()` call from a
caller that includes a raw OS callback (`WinEventProc`) that can fire while
already nested inside XAML-internal layout activity reenters WinUI layout
and fails fast with `STATUS_STOWED_EXCEPTION` — a raw SEH `RaiseException`,
not a C++ exception, so no try/catch anywhere in this file can contain it.
Don't reintroduce a forced call here without a fundamentally different
(genuinely async, post-unwind-only) mechanism.

**Why `InvalidateTaskbarLayout` doesn't set `g_planDirty` itself**: the
marshal is usually asynchronous (`PostMessage`), so a natural
`ArrangeOverride` pass could land on the taskbar thread between this call
setting the flag and the posted message being processed, see it already
true, recompute with stale state, and clear the flag — leaving the posted
invalidate with nothing to do. Each real caller marks dirty itself instead,
at the point its own state change becomes visible on the taskbar thread:
`PerformTaskbarLayoutInvalidate`, and the `SettingsChangedMsg` dispatch case
in `TaskbarWndSubclassProc` (the plan the previous recompute produced was
built from the settings that just got replaced, so it has to be marked
stale exactly there, not by the separate `InvalidateTaskbarLayout()` call
`Wh_ModSettingsChanged` also makes).

**`TaskbarWndSubclassProc` — why it's the sole path, no fallback**:
installed on `g_hTaskbarWnd` by `EnsureTaskbarWnd` via
`SetWindowSubclassFromAnyThread`. A subclass proc only ever runs on the
thread that owns the window, which is what lets `InvalidateTaskbarLayout`/
`ResolvePendingButtonHwnds`/`ApplyLoadedSettings` use a plain, non-blocking
`PostMessage` — no per-call `SetWindowsHookEx`/`SendMessage`/
`UnhookWindowsHookEx` dance needed (an earlier design,
`RunFromWindowThread`, did exactly that dance as a blocking fallback; it was
removed entirely — see "Removal of `RunFromWindowThread`" below). This is
also the SOLE way any of the three private messages reach the taskbar
thread; if this subclass never installs, there is no fallback. Everything
other than these private messages is forwarded to `DefSubclassProc`, which
is also what lets comctl32 clean this subclass up automatically via
`WM_NCDESTROY` if the window is ever destroyed out from under the mod
without `Wh_ModBeforeUninit`'s explicit removal call running first.

**`InvalidateTaskbarLayout`'s hot-path note**: called from `WinEventProc`'s
drag-follow throttle at up to ~7 times/sec while any window on the system is
being dragged, plus every resolve tick, so this is a hot path — the
non-blocking `PostMessage` matters specifically because it doesn't block on
Explorer's UI thread pumping it.

**`EVENT_OBJECT_SHOW` handling, in detail**: a real top-level window
becoming visible is what a pinned-but-not-running app launching looks like
— nudge the resolve timer to run right away, and force it to ignore each
negatively-cached entry's own backoff (see `g_forceResolveUnresolved`
above), instead of leaving it to the backoff schedule. That schedule is
kept as a fallback — this is a fast path on top of it, not a replacement
for it, in case a launch is ever reached through a code path that
legitimately doesn't produce this event. Leading-edge throttled the same
way the location-change branch is: this event fires for every top-level
window becoming visible anywhere on the system, unthrottled, so an app that
opens several windows at once (or a burst of unrelated launches) would
otherwise schedule a full resolve pass — a blocking marshal onto Explorer's
UI thread plus a repeater walk and an `AutomationProperties::GetName` per
button — for every single one. No trailing timer is needed the way
drag-follow has one: unlike a window's final drop position, `arm(0)` just
needs to run once to pick up every currently-pending button, so missing the
last event in a burst costs nothing.

**Drag-follow trailing timer, in detail**: the throttle is leading-edge
only, so the final location-change event of a drag/move — the one carrying
its actual release position — is routinely the one that lands inside the
throttle window and gets dropped, since a drag generates a continuous event
stream right up to release. Without a trailing timer, the icon can be left
classified by a stale mid-drag position until some unrelated window happens
to move. Re-arming a short one-shot timer on every throttled event makes it
always fire once the burst actually goes quiet, applying the final
position. Runs on this same dedicated WinEventHook thread.

`DragFollowTrailingTimerProc`'s `g_unloading` check is a cheap early-exit,
not a safety requirement — `InvalidateTaskbarLayout` itself already no-ops
safely once the subclass comes off during teardown — so there's just no
point updating `g_lastDragFollowInvalidate` or dispatching a call that
can't do anything once unload has started.

**LOCATIONCHANGE filter cost**: this is the cheapest and most selective
filter available for the non-SHOW case — only a window this mod has
actually resolved to a taskbar button can change the layout, and checking a
hashset first skips the three cross-process USER32 calls below for every
other moving window on the system (there are a lot of them — thousands of
raw events within seconds has been observed).

## Module/symbol hooking plumbing

**Why most `HookTaskbarViewDllSymbols` entries are optional**: marking them
optional means `HookSymbols` doesn't abort the whole batch on the first
miss — the goal is a per-symbol report while bringing this mod up on a new
Windows build, not just a single opaque FAILED.

**Why `ArrangeOverride` itself is NOT optional, unlike every other entry**:
this hook is the mod's entire function — if it can't be resolved,
positioning silently does nothing. Every other symbol in that table is
optional purely so `HookSymbols` doesn't abort the whole batch on the first
miss (a per-symbol report is more useful than one opaque FAILED while
bringing this mod up on a new Windows build), but a build where
`ArrangeOverride` itself is missing should surface as a real failure in
Windhawk, not a mod that "loads successfully" and does nothing.

**Why `HandleLoadedModuleIfTaskbarView`/`Wh_ModAfterInit`/`Wh_ModInit` apply
hooks unconditionally, not gated on `HookTaskbarViewDllSymbols`' own return
value**: that value reflects whether EVERY symbol in the table resolved,
optional ones included, so a single missing optional HWND-resolution
symbol — exactly the case that table's `optional = true` entries exist to
tolerate — would otherwise skip applying hooks for every symbol that DID
resolve, `ArrangeOverride` included. The per-symbol resolved/MISSING logging
already reports what to investigate; there's nothing to gain from also
discarding the hooks that succeeded. `ArrangeOverride` is the one truly
required symbol in that table, so `Wh_ModInit`/`HandleLoadedModuleIfTaskbarView`
check it directly via
`TaskbarCollapsibleLayoutXamlTraits_ArrangeOverride_Original` instead of
trusting the aggregate return value, matching `HookTaskbarDllSymbols`' own
check (safe to trust as-is there, since every symbol gating that check is
genuinely required, not optional).

## Mod lifecycle

**Why the WinEvent hooks run on a dedicated mod-owned thread**:
`EVENT_OBJECT_LOCATIONCHANGE` fires for every window move on the entire
system — "thousands of raw events within seconds" has been observed — and
`EVENT_OBJECT_SHOW` (added for the resolve-timer fast path) is nearly as
frequent. `WINEVENT_OUTOFCONTEXT` delivers callbacks on whichever thread
called `SetWinEventHook`, and only if that thread pumps messages;
registering on `g_hTaskbarWnd`'s own thread would put all of that — plus
`WinEventProc`'s own filtering calls on each one — in direct contention
with the shell's own layout work. This dedicated mod-owned thread (the
pattern `taskbar-background-helper.wh.cpp` and
`taskbar-auto-hide-when-maximized.wh.cpp` both use) keeps it off the
shell's thread entirely.

**`kArmResolveNowMsg`**: like `WM_APP` below, a private signal on this
thread's own message queue. The resolve timer lives here too, but callers
of `ArmButtonHwndResolveTimer` run on the taskbar thread, and a per-thread
`SetTimer`/`KillTimer` can only be touched from the thread that owns it —
`PostThreadMessage` is how a different thread asks this one to do that on
its behalf.

**`PeekMessage` at the top of `WinEventHookThreadProc`**: forces this
thread's message queue into existence before `SetWinEventHook` runs, so
`WINEVENT_OUTOFCONTEXT` has somewhere to deliver callbacks to as soon as
registration succeeds, rather than racing the `GetMessage` loop below into
existence.

**Not `WINEVENT_SKIPOWNPROCESS`**: File Explorer windows often run inside
explorer.exe's own process, and that flag would silently drop their
location-change events too. The taskbar's own windows are already excluded
explicitly in `WinEventProc`.

**Why the SHOW hook is a second, separate `SetWinEventHook` call** rather
than widening the LOCATIONCHANGE range: the two event IDs aren't adjacent,
and a single range spanning both would also pick up several unrelated event
types in between.

**Why the resolve timer is kicked off immediately at thread start**: gets
buttons already present at mod startup picked up right away — after that,
`NextResolveDelayMs`/`ArmButtonHwndResolveTimer` keep it armed only for as
long as there's actually something to do.

**`WM_APP` as shutdown signal**: posted by `StopWinEventHook`; it has no
window to route to, so it's read directly out of the queue rather than via
a window procedure.

**Why `g_winEventThreadId` is atomic**: written under
`g_winEventThreadMutex` (`Start`/`StopWinEventHook`), but read without it in
`ArmButtonHwndResolveTimer` from the taskbar thread — atomic makes that
well-defined, matching `g_taskbarWndSubclassed`'s own treatment elsewhere in
this file.

**The race `g_winEventThreadMutex` prevents**: `EnsureTaskbarWnd` can call
`StartWinEventHook` from either `Wh_ModAfterInit`'s own thread or Explorer's
UI thread (via the `ArrangeOverride` hook) once `g_hTaskbarWnd` first
resolves, and that call can still be inside `CreateThread` — before
`g_winEventThread` is written — when `Wh_ModUninit` calls `StopWinEventHook`
concurrently. A plain check-then-act (even an atomic one guarding just the
`CreateThread` call) can't stop `StopWinEventHook` from checking
`g_winEventThread` before `StartWinEventHook` has finished writing it,
missing the very thread it was meant to tear down and leaving it to crash
the process when Windhawk unmaps the module later. The mutex makes "is
there a thread, and should one ever be created again" one atomic question
both functions agree on.

**`CreateThread`'s out-parameter**: needs a plain `DWORD*`, not
`atomic<DWORD>*` — written through a local and then published to the atomic
once `CreateThread` returns.

**Why `StopWinEventHook` sets `g_winEventThreadStopped` before releasing
the lock**, not after the thread is confirmed gone: this is what stops a
`StartWinEventHook` call that arrives after this point (e.g. a late
`ArrangeOverride` pass) from recreating a thread nobody would be left to
tear down.

**Why `StopWinEventHook` retries `PostThreadMessage` instead of giving up**:
it fails with `ERROR_INVALID_THREAD_ID` if the thread hasn't created its
message queue yet (`WinEventHookThreadProc` does that as its first action).
The retry ties to the thread's own liveness rather than a fixed attempt
count: giving up after N tries and falling through to the unconditional
wait below with the signal never delivered would hang `Wh_ModUninit`
forever, since nothing else can wake that wait.

**Why the final `WaitForSingleObject` is unconditional (`INFINITE`), not a
bounded timeout**: giving up early would let `Wh_ModUninit` return and
Windhawk unmap this module's code while the thread is still inside it — a
deferred crash, not a mitigated one. This thread is entirely mod-owned:
`PostThreadMessage` can't silently fail the way a marshaled call onto
someone else's thread can, and waiting for the thread to actually exit
guarantees `UnhookWinEvent` (and the resolve timer's own `KillTimer`, since
it lives on this thread too) has already run before Windhawk unmaps this
module's code.

**`kResolveBackoffCeilingMs`**: the capped exponential backoff is a
fallback safety net for a pinned-but-not-running app's button, in case its
launch is ever reached through a path that doesn't produce
`EVENT_OBJECT_SHOW` (`WinEventProc`'s fast path normally re-resolves it
immediately). The 30-minute ceiling keeps this fallback from becoming a
source of frequent synthetic clicks itself.

**`Wh_ModInit`'s taskbar-view-already-loaded branch**: not gated on
`HookTaskbarViewDllSymbols`' own return value here for the same reason as
`HandleLoadedModuleIfTaskbarView` (see "Module/symbol hooking plumbing"
above) — it reflects whether EVERY symbol in its table resolved, optional
ones included, so a single missing optional HWND-resolution symbol would
otherwise fail this ENTIRE mod's load, exactly the outcome `optional = true`
exists to prevent. `ArrangeOverride` is the one truly required symbol, so
that's checked directly instead, matching `HookTaskbarDllSymbols`' own
check.

**Why `Wh_ModBeforeUninit` deliberately does NOT tear down the
WinEventHook thread**: this mod's hooks stay installed until this function
returns, and `ArrangeOverride` isn't gated on `g_unloading` — a pass landing
in that window could still call `StartWinEventHook` via `EnsureTaskbarWnd`,
creating a thread nobody would tear down (`Wh_ModUninit`, which does the
actual teardown, runs after this). `g_unloading` being set first is what
keeps `ResolvePendingButtonHwnds`' click-sentinel probe from running once
its hooks are gone, and makes `IUIElement_Arrange_Hook` fall through to
native positioning immediately regardless of whether anything below forces
a relayout.

**Why the subclass comes off Shell_TrayWnd as early as it's safe to**,
rather than staying wired in for the rest of teardown: with no fallback
marshal once this is gone, there is no reliable way left to force an
immediate visual snap-back to native positions on disable — `g_unloading`
already guarantees the mod stops overriding anything, so this is a
cosmetic timing difference only: native positions apply as soon as
anything next triggers XAML to re-run Arrange on its own (opening/closing
an app, a resize, etc.), rather than instantly. An earlier version of this
mod called `InvalidateTaskbarLayout()` at the very end of
`Wh_ModBeforeUninit` specifically to force that snap-back immediately; once
`RunFromWindowThread` was removed (see below), that call became a dead
no-op (the subclass is already gone by the point it would run, so
`InvalidateTaskbarLayout` always no-ops), and it was deleted outright as
dead weight rather than kept as a documented no-op — the tradeoff being the
minor cosmetic delay described above.

**Why `Wh_ModUninit` (not `Wh_ModBeforeUninit`) tears down the WinEventHook
thread**: the mod's own hooks are gone by the time `Wh_ModUninit` runs, so
nothing can reawaken the thread being torn down here. This one join also
covers the HWND-resolve timer, which lives on the same thread.

**`ApplyLoadedSettings`, in detail**: applies a freshly-loaded settings
struct on the taskbar's own thread — needed since it reassigns
`leftApps`/`rightApps` (`std::vector<std::wstring>`), which a concurrent
`ContainsAnyFragment` call during a layout pass on that thread could
otherwise read mid-reassignment. No fallback if the subclass isn't
installed yet or `PostMessage` itself fails. Unlike
`InvalidateTaskbarLayout`/the HWND-resolve tick, a dropped settings change
here is a real loss rather than a delayed retry (there's no periodic
mechanism that would pick this specific settings change up later), so it's
logged loudly rather than silently swallowed — in practice this should only
ever be hit in the brief window before `EnsureTaskbarWnd`'s own retry gets
the subclass installed. `PostMessage` is async, so the settings can't live
on this function's own stack — ownership transfers to a heap allocation,
released via `.release()` only once `PostMessage` has actually queued it,
and reclaimed by `TaskbarWndSubclassProc`'s `SettingsChangedMsg` case on the
dispatch side.

**`Wh_ModSettingsChanged`**: reads every setting on the calling thread —
`Wh_Get`/`FreeStringSetting` don't touch XAML/COM, so they don't need to run
on the taskbar thread at all, only the final assignment into `g_settings`
does (see `ApplyLoadedSettings`).

## Removal of `RunFromWindowThread`

An earlier version of this mod maintained a second, blocking cross-thread
marshal path (`RunFromWindowThread`, built on a temporary
`SetWindowsHookEx(WH_CALLWNDPROC)` hook) as a fallback for
`InvalidateTaskbarLayout`/`ButtonHwndResolveTimerProc`/`ApplyLoadedSettings`
in case the `TaskbarWndSubclassProc` subclass install ever failed. It was
removed entirely: `SetWindowSubclassFromAnyThread` failing is rare, the
fallback added real complexity (~65 lines including its own hook
installation/teardown), and the round-23 fix to `EnsureTaskbarWnd` (making
the subclass install retry on every call instead of once) already closes
most of the gap the fallback existed for. The accepted tradeoff: if the
subclass install fails, live drag-follow/HWND-resolution/settings-updates
are unavailable until a later `EnsureTaskbarWnd` call succeeds, with no
blocking fallback in between. Core centering/splitting is unaffected either
way, since that's computed synchronously in `RecomputeLayoutPlan`
regardless of the subclass. See "Why the subclass comes off Shell_TrayWnd
as early as it's safe to" above for the one further consequence of this
removal (the disable-time snap-back becoming cosmetically delayed instead
of immediate).
