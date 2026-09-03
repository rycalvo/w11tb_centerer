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

**`g_lastUpdateVisualStatesArm`**: leading-edge throttle for
`TaskListButton::UpdateVisualStates`' hook (round 29; see "Removing
`EVENT_OBJECT_SHOW`" below) - same idea as `g_lastDragFollowInvalidate`'s
LOCATIONCHANGE throttle, but touched from the taskbar/XAML UI thread this
hook runs on instead of the dedicated WinEventHook thread. A single
`arm(0)` call is enough to catch every button currently pending regardless
of how many `UpdateVisualStates` calls land in the same burst. There's no
"final position" that specifically needs a trailing event the way
drag-follow does, so no trailing timer is needed here either.

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
internally, or the item was mid-teardown during a taskbar rebuild. An
earlier version of this mod tolerated `kClickSentinelMissesBeforeBroken = 3`
misses before latching, reasoning that a false latch (degrading every
button on the taskbar to `unresolvedAppsDefaultSide` for the rest of the
session over one unlucky miss) was worth avoiding at the cost of a few more
probes.

Round 29's AI review argued the asymmetry actually favors the opposite
choice, specifically for the *pre-confirmation* case: at that point there
is zero evidence the interception works at all, so a false latch only
costs position tracking (recoverable by reloading the mod), while a false
negative here means dispatching more probes with no safety net at all -
each one a real, unintercepted click if the mechanism is genuinely broken
(a window activating or minimizing that the user didn't touch). Changed
`kClickSentinelMissesBeforeBroken` to `1`: the first pre-confirmation miss
on either path now latches it immediately. `g_realTaskbarClickObserved`
(the first-probe gate — see its own section below) already does the heavy
lifting of keeping the *pre-confirmation* window itself short and evidence-
based, so this mostly bounds the case where interception is broken in a
way that gate can't detect (e.g. `ReportClicked` reaches `HandleClick` fine
in general, but this specific button's call happens to fail for an
unrelated reason on its very first try).

This latch only ever applies pre-confirmation:
`NoteUnconfirmedClickSentinelMiss` returns immediately once a path is
confirmed, by design — a post-confirmation miss is routinely innocent (see
above), so latching a working path dead over an unrelated timing hiccup
would be a worse tradeoff there than pre-confirmation. Post-confirmation
protection instead comes from `kMaxResolveFailures` (a per-cache-entry cap
on genuinely dispatched-and-missed clicks, not a whole-path latch — see its
own section). Also note `ResolveHwndFromTaskListButton` tries the item path
first and falls through to the group path on a miss, so one unresolvable
button can still burn a probe on both paths in a single attempt before
either has confirmed — the worst case before both latches trip is 2
dispatched clicks total (one per path), not `kClickSentinelMissesBeforeBroken`
per path.

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

**`g_lastArrangedTaskListWidth` and the label-populate overlap bug (found
during round 29 live-testing)**: a freshly-launched app's `TaskListButton`
can render icon-only (its label not yet measured/populated) for the very
`ArrangeOverride` pass that first plans its position, then grow wider a
moment later once the label populates — with neither the button count nor
`g_planDirty` changing in between, since XAML's own internal re-measure of
just that one button's content is invisible to both. The hash-lookup-only
staleness check above had no way to notice: the button was already a key in
`g_lastArrangedX`, so it read as "covered" regardless of whether the width
used to compute that entry's X still matched reality. The practical effect
was specific to the *left* side: a left-side button's X is computed as
`innermostX - widthAtComputationTime`, so growing wider afterward extends
its rendered right edge past `innermostX` — directly into Start's own
space, rendering its label on top of Start. (A right-side button's X is
its own left edge directly, so growing wider extends its right edge
further away from Start instead - no overlap, just a harmless gap that the
next real recompute closes.) The 500ms `kMaxPlanStalenessMs` backstop
doesn't help here either, despite the name: it only forces a real recompute
on whatever pass actually runs after that much time has elapsed, and
nothing about a label populating is guaranteed to trigger another
`ArrangeOverride` pass at all if XAML doesn't consider that specific change
significant enough to invalidate the parent repeater's own arrange - the
bug could in principle persist indefinitely, only closed in practice by
some unrelated trigger (most visibly, any drag/move, which calls
`InvalidateTaskbarLayout` and forces a fresh pass). Fixed by having
`PlanTaskListButtons` also record each button's `FullFootprintWidth()` at
plan-computation time into `g_lastArrangedTaskListWidth` (rebuilt from
scratch alongside `g_lastArrangedX` every full recompute, so a removed
button's stale entry doesn't linger), and having the staleness check
compare each already-covered task list button's *current* width against
that recorded value - a mismatch (beyond a 0.5px epsilon, for floating-
point noise) now forces a real recompute the same way a coverage or count
mismatch already did. The lookup is scoped for free: the map only ever
holds task list button keys, so the comparison naturally no-ops for Start
and Search/Task View/Widgets (which don't dynamically resize the same way,
and weren't observed to hit this). Not a round-29 regression - the
underlying gap in the staleness check predates it - but round 29's faster
resolve timing plausibly made it far more likely to actually get caught by
a positioning pass while still in its narrow, pre-label state, rather than
being incidentally masked by a slower path settling first.

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
defeating the fast path's whole purpose of triggering an immediate resolve
(round 29: `TaskListButton::UpdateVisualStates`; originally
`EVENT_OBJECT_SHOW` - see "Removing `EVENT_OBJECT_SHOW`" below). It's
bounded to `consecutiveFailures < kMaxForcedRetryFailures` —
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
bug**: set by `TaskListButton::UpdateVisualStates`' hook (round 29; see
"Removing `EVENT_OBJECT_SHOW`" below — originally `WinEventProc`'s
`EVENT_OBJECT_SHOW` branch) and the `ArrangeOverride` hook's count-change
check — both call `ArmButtonHwndResolveTimer(0)` to make the next resolve
pass run immediately, but arming the timer sooner doesn't by itself bypass
a negatively-cached entry's own backoff gate (that backoff can be
long-lived — see `ButtonHwndCacheEntry` below). Consumed once per pass in
`ResolvePendingButtonHwnds` to force a negatively-cached entry to retry
regardless of backoff, but only up to `kMaxForcedRetryFailures` consecutive
failures. A group with no running windows can never yield an HWND — since
round 29, that's now caught even earlier, by `TaskListButtonIsRunning`'s
cheap pre-check in `ResolveHwndFromTaskListButton`, before
`ResolveHwndFromTaskGroup`'s own `IsRunning` check ever runs — the
original, unconditional version of this force assumed THAT is the only
reason a negatively-cached entry keeps failing (a pinned app that just
launched, still catching up to its own running-state signal). But a
RUNNING app's button can also fail to resolve for reasons that don't bail
out early — `ReportClicked` failing internally, a task item mid-teardown,
`GetTaskItemsArray` coming back empty — and for those, forcing every retry
unconditionally meant dispatching a real `ReportClicked` on both paths on
every forced pass, indefinitely, at whatever rate the fast-path hook can
arm this at — exactly the runaway-real-clicks scenario the backoff schedule
exists to bound. Capping the force-bypass to entries that have only failed
a few times keeps the "pinned app just launched" fast path intact (that
case is still failing 0 times when the fast path first fires) while letting
the normal backoff schedule take back over for an entry that's failed
repeatedly, the same way it already would with no force at all.

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

**`notRunning` (round 29)**: set when the last resolve attempt bailed via
`TaskListButtonIsRunning`'s cheap pre-check specifically, as opposed to any
other bail-out reason (`!g_realTaskbarClickObserved`, a null view model,
`ITaskGroup::IsRunning == false` reached the old way). Before this field
existed, a confirmed-not-running entry was indistinguishable from any other
zero-`consecutiveFailures` bail-out, so `NextResolveDelayMs` had no way to
tell it apart from a genuinely-still-pending one - it would poll it at the
fast `ResolveBackoffMs(0)` (2s) cadence forever, since a bail-out never
accumulates failures and so never reaches the terminal `kMaxResolveFailures`
idle treatment either. `notRunning` closes that gap directly: it's checked
everywhere `entry.hwnd` and terminal `consecutiveFailures` already are (see
`NextResolveDelayMs`), so a confirmed-not-running button gets the same idle
`kIdleResolveTickMs` cadence as a resolved one, and only reactivates on
identity mismatch (a drag-reorder) or `TaskListButton::UpdateVisualStates`
forcing a re-check (see "Removing `EVENT_OBJECT_SHOW`" below). It does not
need a separate reset path back to `false`: `ResolveAndCacheButtonHwnd`
recomputes the whole cache entry fresh on every resolve attempt, so a
button that starts running again simply gets a fresh entry with
`notRunning = false` the next time it's actually resolved.

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

**`TaskListButton::UpdateVisualStates` handling, in detail** (round 29;
see "Removing `EVENT_OBJECT_SHOW`" below for what this replaced): a
visual-state transition on a taskbar button — which includes a
pinned-but-not-running app transitioning to running — nudges the resolve
timer to run right away, and forces it to ignore each negatively-cached
entry's own backoff (see `g_forceResolveUnresolved` above), instead of
leaving it to the backoff schedule. That schedule is kept as a fallback —
this is a fast path on top of it, not a replacement for it, in case a
launch is ever reached through a code path that doesn't produce this call,
or on a Windows build where the symbol didn't resolve at all (it's
optional — see `HookTaskbarViewDllSymbols`). Leading-edge throttled the
same way the location-change branch is: this hook fires on every
visual-state transition of every taskbar button (hover, press, and
running/not-running alike), not just the one this mod cares about, so a
burst of unrelated UI activity would otherwise schedule a full resolve
pass — a blocking marshal onto Explorer's UI thread plus a repeater walk
and an `AutomationProperties::GetName` per button — for every single call.
No trailing timer is needed the way drag-follow has one: unlike a window's
final drop position, `arm(0)` just needs to run once to pick up every
currently-pending button, so missing the last event in a burst costs
nothing.

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

**Why the WinEvent hook runs on a dedicated mod-owned thread**:
`EVENT_OBJECT_LOCATIONCHANGE` fires for every window move on the entire
system — "thousands of raw events within seconds" has been observed (this
used to also register `EVENT_OBJECT_SHOW`, nearly as frequent; round 29
removed it - see "Removing `EVENT_OBJECT_SHOW`" below - but
`EVENT_OBJECT_LOCATIONCHANGE` alone already justifies keeping this off the
shell's own thread). `WINEVENT_OUTOFCONTEXT` delivers callbacks on whichever
thread
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
launch is ever reached through a path that doesn't produce a
`TaskListButton::UpdateVisualStates` call, or on a build where that symbol
didn't resolve (the fast path normally re-resolves it immediately - see
"Removing `EVENT_OBJECT_SHOW`" below). The 30-minute ceiling keeps this
fallback from becoming a source of frequent synthetic clicks itself. Since
round 29, this ceiling mostly matters for a genuinely still-failing
*running* app's button - a confirmed not-running one now goes idle instead
of climbing this schedule at all (see `ButtonHwndCacheEntry::notRunning`
below).

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

## `ComputeSystemButtonX`'s taskbar-order rewrite

Previously used a hardcoded `SystemButtonRank` table (Search=0, TaskView=1,
Widgets=2) to decide each system button's ordering relative to the others.
This silently assumed at most one instance of each `SystemButton` enum
value could ever exist - if a future Windows build ever surfaced two
`Taskbar.TaskbarExtensionElement`s (both classified as `SystemButton::Search`
by `IdentifySystemButton`), both would get rank 0, both would compute the
same X, and they'd render on top of each other. Rewritten to walk
`childInfos` (already in taskbar order) directly, summing the footprint of
every other system button that appears earlier than the target element,
stopping at the target's own identity (`info.element == targetElement`) -
this tracks whatever order the taskbar actually presents these buttons in,
rather than a fixed table, and degrades gracefully (each button just gets
its own real position, no forced collision) if the one-of-each assumption
ever breaks. Behaviorally identical to the old table-based version for the
normal case (exactly one Search, one TaskView, one Widgets, in that native
order).

## `trackWindowPositions` setting

Added so a user can keep the headline feature (Start pinned to true center,
icons split into two groups by `leftApps`/`rightApps`/
`unresolvedAppsDefaultSide`) without the click-sentinel probing or the two
system-wide `SetWinEventHook` registrations at all - see the reviewer
discussion that prompted this in the PR history for the full tradeoff
analysis.

Implemented as narrowly as possible: `EnsureTaskbarWnd` simply doesn't call
`StartWinEventHook()` when the setting is off. Nothing else needed to
change - `g_buttonHwndCache` then never gets populated (nothing ever calls
`ResolveAndCacheButtonHwnd`, since that only ever runs from the resolve
timer this thread would have owned), so `GetButtonHwnd` always returns
`nullptr`, and `ClassifyTaskListButton`'s existing "unresolved" fallback
path (leftApps/rightApps, then `ResolveUnresolvedAppsDefaultSide()`) already
does exactly what's wanted for every running app, with zero changes to that
function. The click-sentinel hooks (`CTaskListWnd_HandleClick_Hook`,
`ITaskGroup_IsRunning_Hook`) stay installed either way - they only ever
intercept a sentinel-marked call, and with nothing ever dispatching one,
they're inert.

One real interaction worth calling out in the settings description:
`taskListOrder: distance-from-center` mode's ranking degenerates when this
is off, since every button now gets the same +/-infinity `orderKey` from
`PinnedAppOrderKey()` (there being no live window position left to measure
a real distance from) - the only remaining ordering signal within a side is
`pinnedAppsAnchor`, plus the taskbar-index tiebreak. Not a bug, just a
mode combination worth knowing about.

**Why toggling it live needed `StopWinEventHookForToggle`, a separate
function from `StopWinEventHook`, rather than just calling the existing
one**: `StartWinEventHook`/`StopWinEventHook`'s serialization already has a
documented, deliberately one-way latch - `g_winEventThreadStopped`, set
permanently `true` by `StopWinEventHook` specifically so a
`StartWinEventHook` call arriving after mod teardown begins can't recreate
a thread nobody would tear down (see the "race `g_winEventThreadMutex`
prevents" entry above). The first implementation of this setting reused
`StopWinEventHook` directly from a live settings change, which would have
tripped that same latch permanently - flipping the setting back on later
would then silently do nothing until the mod was reloaded, since
`StartWinEventHook` checks that exact latch and refuses to start. Caught
during the user's own testing (toggling the setting live and seeing no
effect either direction) before it shipped.

Fixed by splitting the shared thread-teardown body into
`StopWinEventHookInternal(bool permanent)`, with `StopWinEventHook()`
(mod-teardown path, called from `Wh_ModUninit`) always passing `true` and a
new `StopWinEventHookForToggle()` (live-settings path) passing `false`. The
critical invariant - `g_winEventThreadStopped` only ever transitions
permanently, and always does so on every real teardown regardless of
whether a concurrent toggle-stop already tore the thread down first - holds
because `StopWinEventHookInternal` sets it unconditionally when
`permanent` is true, *inside* the mutex-guarded section, before checking
whether a thread was actually found. So a `StopWinEventHookForToggle` call
racing an actual `Wh_ModUninit` teardown can't leave the latch unset: either
it runs first and clears `g_winEventThread` (the teardown's own
`StopWinEventHookInternal(true)` then still sets the latch, just finds
`thread == nullptr` and returns immediately), or the teardown runs first
and sets the latch itself before the toggle's call gets the lock. Either
interleaving ends with the latch set and the thread properly joined exactly
once.

The actual start/stop-on-change trigger lives in `TaskbarWndSubclassProc`'s
`SettingsChangedMsg` case, not in `ApplyLoadedSettings` - that's the one
point where `g_settings` actually changes value for every settings-update
path, including the deferred one through `g_pendingSettings`/
`ApplyPendingSettingsIfAny` (which itself re-enters `ApplyLoadedSettings`,
which - once the taskbar and subclass are both ready - always ends up back
at this same `PostMessage(SettingsChangedMsg())`/dispatch path). Comparing
old vs. new `trackWindowPositions` right there, on the taskbar thread,
before deciding to call `StartWinEventHook()`/`StopWinEventHookForToggle()`,
means every settings-change path is covered by one check with no risk of
missing one. `EnsureTaskbarWnd`'s own one-shot `StartWinEventHook()` call
remains for the genuine initial-startup case (and the rare taskbar-recreate
case) - by the time any settings change could reach the
`SettingsChangedMsg` dispatch path, the mod is already fully up and
running, so the two mechanisms never need to agree on anything beyond "is
the thread currently running," which the shared `g_winEventThread`/
`g_winEventThreadId`/mutex state already answers consistently either way.

**Confirmed working by the user's own live testing**: toggling the setting
off froze `winEvents: raw=` for the rest of that session (thread genuinely
stopped) and toggling it back on produced a brand-new
`WinEventHookThreadProc` log line with a different thread id (fresh
`CreateThread`, not a leftover). One gap the same test surfaced:
`g_buttonHwndCache` isn't cleared on toggle-off, so an already-resolved
button kept reporting a nonzero `hwndResolved` and used its last-known
(now frozen) window position instead of immediately falling back to
`leftApps`/`rightApps`/`unresolvedAppsDefaultSide`, contradicting the
setting's own description. Fixed by clearing `g_buttonHwndCache` in the
same `SettingsChangedMsg` branch that calls `StopWinEventHookForToggle()` -
safe to do directly there since every other access to that cache already
runs on this same taskbar thread (via `ResolvePendingButtonHwnds`,
dispatched through the identical `TaskbarWndSubclassProc` mechanism).

## `g_realTaskbarClickObserved` (first-probe safety gate)

The unattended click-sentinel probe (see the HWND-resolution section
above) is only as safe as `CTaskListWnd_HandleClick_Hook`'s interception
actually being reachable. Everything else in that system (the miss-counter
latch, the per-entry backoff ceiling) detects a broken interception only
*after* it's already dispatched at least one probe that may have become a
real click. This doesn't close that gap - a `ReportClicked` implementation
that touches its `LauncherOptions const&` argument before ever reaching
`CTaskListWnd::HandleClick` would still crash on the very first probe, with
no miss ever recorded - but it does close a narrower, cheaper-to-close one:
proving the hook chain is *installed and actually being exercised* at all,
before the very first unattended probe of the session ever runs.

`CTaskListWnd_HandleClick_Hook` already sees every genuine (non-sentinel)
click that reaches it, via its existing pass-through branch to
`CTaskListWnd_HandleClick_Original`. Setting `g_realTaskbarClickObserved`
there the first time that branch runs, and having both
`ResolveHwndFromIndividualTaskItem`/`ResolveHwndFromTaskGroup` bail out
before ever dispatching their own sentinel click until that flag is true,
means the interception point has to have already handled one real click
successfully before this mod's own background timer is allowed to risk
one of its own. The practical cost is small and one-time: until the user
clicks any taskbar button once per session, running apps classify the same
way pinned-but-not-running ones always do (via
`leftApps`/`rightApps`/`unresolvedAppsDefaultSide`) - in practice a
near-imperceptible delay, since most sessions see a real taskbar click
within seconds. A single shared flag (not per-path) is enough: the item
and group paths share the same `CTaskListWnd::HandleClick` entry point, so
proving it's reachable at all covers both.

## Diagnostics logging trim (per-symbol resolved/MISSING lines)

`HookTaskbarDllSymbols`/`HookTaskbarViewDllSymbols` used to log every
optional symbol's resolved/MISSING status unconditionally, every single
mod load - 3 and 6 lines respectively, on top of the aggregate OK/FAILED
line. That's pure noise once a build is known-good, but the per-symbol
detail has real, repeatedly-used value the one time it isn't: pinpointing
exactly which symbol a future Windows build renamed, without needing a
second round-trip with the user to ask "which one is missing." Collapsed
to a single conditional line per function, built from the same booleans,
that only prints (and only lists the missing ones) when the aggregate `ok`
is false - the common case (everything resolves) now logs nothing extra at
all, and the failure case still names every missing symbol, just on one
line instead of several.

Deliberately **not** applied to the periodic `Arrange pass: ...` stats
line (`ArrangePassStats`, `LayoutPlanStats`, `g_resolveStats`, the four
standalone WinEvent counters) the way a reviewer once suggested trimming
to just the `taskList=N (hwndResolved=M left=.. right=..)` portion. That
richer line is the primary tool that actually diagnosed this project's own
crash history (`skippedReentrant`, `qiFail`/`exceptions`,
`invalidateExceptions`, and `winEvents: raw=` were each individually
load-bearing in separate past incidents - see project history/commit log
for specifics) - while the mod is still under active iteration with the
same debugging workflow, the ongoing diagnostic value outweighs the
code-surface argument for cutting it. Worth revisiting once the mod is
genuinely stable and that workflow winds down.

## Two round-25 suggestions deliberately left as-is

**Folding `RATIONALE.md`'s content back into inline comments.** A
reviewer suggested this would make the file self-contained for a future
maintainer who never sees the external repo. The specific spots where a
"see RATIONALE.md" pointer had actually replaced a load-bearing constraint
(rather than just deferring extra history/color) were identified and fixed
directly in round 24 - the reviewer named three, all restored with a
one-sentence conclusion plus the pointer. Folding the remaining ~30
pointers back in would undo the comment-trimming pass itself, which was a
separate, explicit, deliberate request from the mod's author two rounds
earlier. Not attempted at the time, since it would have reversed a
decision made independently of that specific review round without the
author's own say-so.

Round 29 raised the identical suggestion again, and this time the mod's
author explicitly asked for it to be addressed - see "Round 29's optional
items" below for what that turned into: a partial, selective fold (the
pointers that were genuinely bare, not the majority that already carried
real inline reasoning), keeping `RATIONALE.md` as the long-form companion
throughout rather than reversing the round-23 trim wholesale.

**Dropping overflow task-list buttons to native position instead of
letting a crowded side's icons overlap.** Raised by a reviewer as an
"if it bothers you in practice" idea, not a firm recommendation, and the
reviewer's own phrasing already flags the tradeoff: this mod overrides X
position only, never sizing, so there's no clean handoff to the taskbar's
real overflow handling. What "native positioning" actually renders as for
a button permanently excluded from `outPlan` (as opposed to the existing
one-frame case for a just-realized zero-width button, which is
imperceptible) is genuinely unverified - the native `ItemsRepeater` panel
has no notion of this mod's split-layout model, so an excluded button's
native position could as easily land somewhere more confusing than the
compressed-overlap it would be replacing. `PlanTaskListButtons` is also
this file's single most crash-history-laden function (see the crash
narrative in project history for the multiple incidents that were only
resolved by rewriting how this file relates to XAML layout entirely).
Given both the unverified visual outcome and that history, this wasn't
implemented blind - it needs a live crowded-taskbar test loop with the
user before landing, not a speculative change to the file's most fragile
code path.

## Two `kMaxResolveFailures` bugs found by round 26, both from the same
root cause: the cap didn't distinguish *why* a resolve attempt failed

`kMaxResolveFailures` (added the round it was introduced) was meant to
bound one specific risk: a button whose click-sentinel interception is
broken dispatching real `ReportClicked` calls forever. But
`ResolveAndCacheButtonHwnd` treated every `nullptr` return from
`ResolveHwndFromTaskListButton` as equally close to that risk, when most
`nullptr` returns never got near dispatching a click at all -
`!g_realTaskbarClickObserved`, a null view model, `ITaskGroup::IsRunning
== false` (the ordinary pinned-but-not-running case, i.e. almost every
real taskbar) are all bail-outs *before* `ReportClicked`. Counting them
toward the same cap as a genuine dispatched-and-missed click created two
distinct bugs:

**Bug 1: the terminal cap silently killed resolution for the rest of the
session, well before any real click-safety concern.** A pinned-but-not-
running app's group bails at the `IsRunning` check on *every* attempt, so
its `consecutiveFailures` climbed the normal 2s/4s/8s/.../256s backoff
schedule and hit 8 (`kMaxResolveFailures`) after summing to ~510s (~8.5
min) - at which point `ResolvePendingButtonHwnds`' `backoffElapsed` check
permanently refused to retry it, with no dispatched click ever having
happened. Worse, `g_realTaskbarClickObserved`'s own gate (added the same
round as the cap) meant *every* entry bailed out this way during the
window before the user's first taskbar click, so a session with no click
in the first ~8.5 minutes reached this terminal state for every button at
once - position-based splitting and drag-follow silently died for the
rest of the session, with every icon stuck on `unresolvedAppsDefaultSide`,
directly contradicting the "until you click any taskbar button once"
tradeoff the gate's own README bullet promises.

**Bug 2: `NextResolveDelayMs` didn't know about the cap at all, so a
terminal entry pinned the resolve timer at ~100 Hz forever.**
`ResolvePendingButtonHwnds` stops updating `lastAttempt` once an entry is
terminal (it's never retried), but `NextResolveDelayMs` kept computing
`dueAt = lastAttempt + ResolveBackoffMs(consecutiveFailures)` for every
`hwnd == nullptr` entry regardless, with no equivalent terminal check.
Once that fixed, no-longer-advancing `dueAt` fell into the past, `now -
dueAt` stayed positive forever, so `pendingDelay` stayed at `0` on every
subsequent call - `ScheduleNextResolveTick` then re-armed
`SetTimer(nullptr, 0, 0, ...)` every tick, which USER32 clamps to its
10ms floor (`USER_TIMER_MINIMUM`), producing a permanent ~100Hz loop: a
full `GetRepeaterChildElements` walk plus a `winrt::get_class_name`/
`GetButtonAccessibleName` UIA fetch per button, on the shell's own UI
thread, accomplishing nothing, for the rest of the session. This bug
existed independently of bug 1 - fixing only bug 1 (so the cap rarely
triggers) wouldn't have made `NextResolveDelayMs` correct if a terminal
entry ever *did* occur through the intended path (a genuinely broken
interception).

**The fix closes both from the same root cause.** `ResolveHwndFromIndividualTaskItem`/
`ResolveHwndFromTaskGroup` now take an `outClickDispatched` out-parameter,
set `true` only immediately before the `TaskItem_ReportClicked_Original`/
`TaskGroup_ReportClicked_Original` call each makes - every earlier
`return nullptr` in either function leaves it `false`. `ResolveHwndFromTaskListButton`
threads this through (also fixing a related latent double-dispatch: if the
item path dispatches and misses, `itemDispatched` is `true` even though
`hwnd` is `nullptr`, so the group path is no longer tried afterward for
the same button - previously a missed item-path click would silently fall
through and dispatch a second, redundant group-path click too).
`ResolveAndCacheButtonHwnd` now only increments `consecutiveFailures` when
`clickDispatched` is `true`; a bail-out leaves it unchanged (`lastAttempt`
still advances either way, so a persistently-bailing-out entry - like a
pinned-not-running app for its entire session - keeps retrying forever at
`ResolveBackoffMs(0)`'s flat 2s cadence, cheap and indefinite, rather than
escalating toward a cap that was never meant to apply to it - round 29,
below, later replaced "cheap and indefinite" with genuinely idle for the
specific not-running case, once a cheap enough running-state signal existed
to justify it). Separately,
`NextResolveDelayMs` now treats `consecutiveFailures >= kMaxResolveFailures`
exactly like an already-resolved entry (`anyIdleWorthy`, not `anyPending`)
- both conditions mean "`ResolvePendingButtonHwnds` will never move this
entry's `dueAt` again," so both need the same `kIdleResolveTickMs`
treatment rather than a `dueAt` that can fall further into the past
forever. With the click-dispatch fix in place the terminal state should
now be rare in practice (it requires several genuine capture-misses on
one specific button, and the pre-existing session-wide
`kClickSentinelMissesBeforeBroken` latch already trips the whole path
broken after just 3), but `NextResolveDelayMs` has to be correct
regardless of how rarely the state it's guarding against actually occurs.

## Removing `EVENT_OBJECT_SHOW`: `TaskListButton::UpdateVisualStates` and
the not-running pre-check (round 29)

Round 29's review raised two related but distinct costs in the HWND-
resolution chain, both hitting hardest for the ordinary case of a pinned
app that isn't currently running:

1. Every 2 seconds, forever, for as long as that button exists,
   `ResolvePendingButtonHwnds` ran the *entire* group-path chain -
   `TryGetItemFromContainer<TaskListGroupViewModel>`, `IsMultiWindow`
   (itself calling `ITaskGroup::IsRunning` internally, hooked to capture
   its `this`), then `ITaskGroup::IsRunning` again for the real answer -
   only to learn, every time, what was already true the previous 2
   seconds: the app still isn't running. `ResolveHwndFromIndividualTaskItem`
   was already cheap for this case (see the "Two `kMaxResolveFailures` bugs
   ..." section above - `TryGetItemFromContainer<TaskListWindowViewModel>`
   fails fast for a button with no window, before ever reaching
   `ReportClicked`), so this cost was specific to the group path.
2. `EVENT_OBJECT_SHOW` - a `SetWinEventHook` registration for every
   top-level window becoming visible *anywhere on the system* - existed
   solely to nudge the resolve timer the instant a pinned app actually
   launched. That's a lot of desktop-wide event traffic to buy one signal
   about this taskbar's own buttons specifically.

Both were addressed by moving to two real primitives from
`Taskbar.View.dll`, cross-checked against `taskbar-labels.wh.cpp` (a
published Windhawk mod using both against the exact same
`winrt::Taskbar::implementation::TaskListButton` class) rather than
guessed:

**`TaskListButton::get_IsRunning`** (symbol:
`winrt::impl::produce<TaskListButton, ITaskListButton>::get_IsRunning`) -
a plain accessor (registered with no hook function, like
`CWindowTaskItem::GetWindow`), called directly off the XAML element itself:
`TaskListButton_get_IsRunning_Original(get_abi(element.as<IUnknown>()),
&running)`. `TaskListButtonIsRunning` wraps this and is called first thing
inside `ResolveHwndFromTaskListButton`, before either path - so a
not-running button now costs one direct property read instead of the whole
group-path chain, and the existing `ITaskGroup::IsRunning` check inside
`ResolveHwndFromTaskGroup` is left in place as a second, independent
confirmation for whatever's still reached it (defense in depth, not
redundancy - the two accessors are different objects at different layers,
and there's no proof they can never transiently disagree). The `.as<IUnknown>()`
QueryInterface (not a raw reinterpret-cast of the element's own default-
interface pointer, unlike the `TryGetItemFromContainer` call sites
elsewhere in this file) matches `taskbar-labels.wh.cpp`'s exact calling
convention for this specific symbol - copied deliberately rather than
assumed equivalent, since a "produce" trampoline's expected `this` pointer
is an ABI detail of that specific vtable slot, not something safe to infer
from how a *different* symbol in this file happens to be called.
Optional, like the rest of the resolution chain: if a future Windows build
renames it, `TaskListButtonIsRunning` returns `true` (assume running, i.e.
skip nothing), which is exactly this mod's pre-round-29 behavior.

**`TaskListButton::UpdateVisualStates`** (symbol:
`winrt::Taskbar::implementation::TaskListButton::UpdateVisualStates`,
`private`) - hooked (unlike `get_IsRunning`) because there's no other way
to observe *when* a state transition happens, only read the current state
on demand. The hook calls through to the real implementation first, then
does exactly what `WinEventProc`'s old `EVENT_OBJECT_SHOW` branch did:
leading-edge-throttled (150ms, via the new `g_lastUpdateVisualStatesArm`),
sets `g_forceResolveUnresolved` and calls `ArmButtonHwndResolveTimer(0)`.
Unlike `EVENT_OBJECT_SHOW`, this fires only for this taskbar's own XAML
buttons (in-process hook on a specific vtable slot, not a desktop-wide
`WinEventHook`), and covers strictly more than "a pinned app just
launched" - any visual-state change, including running/not-running in
either direction - though it also fires for states this mod doesn't care
about at all (hover, press), which is exactly why the throttle still
matters here. This hook runs on the taskbar/XAML UI thread (the same
thread `ArrangeOverride` already calls `ArmButtonHwndResolveTimer` from),
not the dedicated WinEventHook thread `EVENT_OBJECT_SHOW`'s throttle
variable used to live on - hence `g_lastUpdateVisualStatesArm` being a
plain, not `std::atomic`, `ULONGLONG`: exactly one thread ever touches it,
same as `g_lastDragFollowInvalidate` on its own (different) thread.
Optional; if it's ever missing, the fast path is simply gone and every
button - not just not-running ones - falls back to the pre-round-29
`ResolveBackoffMs` capped-backoff schedule, the same degraded mode this
mod already tolerated for a launch that reached `EVENT_OBJECT_SHOW`
through an unexpected path. No FrameworkElement is ever extracted from the
hook's `pThis` (`taskbar-labels.wh.cpp` does, via a fixed member-pointer
offset, to look up its own per-button customization) - this mod's use only
needs a global "something changed, go recheck everything" nudge, since
`ResolvePendingButtonHwnds` already walks every button each pass, so
skipping that extraction avoids relying on an unverified struct-layout
offset for no benefit.

**Why the 2-second poll wasn't fully eliminated, only slowed to the idle
cadence**: `ButtonHwndCacheEntry::notRunning` (see above) makes
`NextResolveDelayMs` treat a confirmed-not-running entry as idle-worthy,
so the *shared* resolve timer (one timer for every button, not one per
button) only fires every `kIdleResolveTickMs` (30s) once nothing else is
genuinely pending - at which point `ResolvePendingButtonHwnds`' own
`backoffElapsed` check (unchanged; still `ResolveBackoffMs(0)` = 2s) is
trivially satisfied by the time 30s have passed, so the button still gets
rechecked, just at 1/15th the frequency, and via the now-cheap pre-check
rather than the old chain. A purely event-driven design (zero polling,
relying only on `UpdateVisualStates`) was considered and rejected: that
hook is a `private`, unversioned symbol - a materially higher risk of
breaking on some future Windows build than the file's one genuinely
required symbol, `ArrangeOverride` - and this mod's established pattern
(see `kMaxPlanStalenessMs`'s backstop for `g_planDirty` elsewhere in this
file) is to never let a single mechanism be the only thing standing
between "up to date" and "silently stuck," however unlikely that
mechanism is to fail. Keeping the periodic idle recheck, rather than
relying solely on the event, means a hook that fails to resolve on some
future build degrades to "not-running buttons recheck every 30s instead of
being nudged instantly" - worse, but not broken, and no different in kind
from the drag-reorder rebind detection that idle tick already exists for.

## Round 29's optional items

The round 29 review's main-body findings (unattended-probing risk,
resolve-loop idling, the `taskbar-start-button-position` overlap) are
covered above and in the source's own comments. Its collapsed, explicitly
optional sections raised five "your call" polish items plus two
non-critical functionality notes; the mod's author asked for all of them
to be addressed in the same session, live-tested separately. What each
turned into:

**`EnsureTaskbarWnd` now snapshots `g_hTaskbarWnd` into a local at the
top**, used for every read (and the final return) instead of six direct
reads of the atomic. Purely a comment/contract-consistency fix - the
function is the variable's sole writer, so there was never a real bug -
but the variable's own comment asks every caller with more than one read
to do this, and this function didn't.

**`Wh_ModBeforeUninit`'s comment no longer calls the pre-removal
`SendMessage` an "immediate snap-back."** `InvalidateArrange`/
`InvalidateMeasure` only mark the tree dirty; the actual re-Arrange runs
on XAML's own next layout pass, by which point `g_unloading` already makes
`IUIElement_Arrange_Hook` fall through to native positioning anyway - same
end result, more accurate description of how it gets there.

**`GetButtonAccessibleName`'s comment now states its own blind spot**:
with "Combine taskbar buttons: Never," two windows of the same app produce
two `TaskListButton`s with the identical accessible name, so an
`ItemsRepeater` rebind between those two specifically wouldn't be caught
by the `identity != it->second.identity` check elsewhere in this file -
only a rebind onto a differently-named button is detected. Documentation
only; no attempt was made to fix the underlying gap (there's no cheaper
alternative identity signal available for a pinned-but-not-running button,
which is exactly the case this identity check exists for in the first
place).

**Known limitations gained a Start-menu bullet**: this mod repositions the
Start *button* only. Nothing decides where the Start *menu* itself opens,
so with the taskbar's own alignment set to "Left" (rather than "Center,"
where Windows' own default menu placement happens to already look right),
the button sits at true center while the menu still opens at the left
edge. Documentation only, matching how the `taskbar-start-button-position`
overlap itself is handled - there's no exposed API to move WinUI's own
Start flyout anchor from a Windhawk mod.

**`SystemButtonContentWidth`'s "0 while unmeasured" race, fixed - but not
exactly as the reviewer suggested.** The reviewer's suggestion was to give
the Search/Task View/Widgets cluster width the same "only update when
positive" treatment `g_lastStartWidth` already has. That doesn't actually
transfer: Start is *never* legitimately 0, so "only update when positive"
can't ever mask a real state for it, but the whole point of
`SystemButtonContentWidth` reading the content child's `DesiredSize`
instead of the outer element's `ActualWidth()` is that `DesiredSize`
*genuinely does* read ~0 when a button is deliberately hidden via its
negative-margin collapse trick - and Search/Task View/Widgets can be
toggled hidden/shown live, without a mod reload, via ordinary taskbar
settings. Applying "only update when positive" literally would make a
live hide-toggle get permanently stuck reserving space for a
now-nonexistent cluster, which is worse than the one-frame startup jitter
being fixed. Implemented instead: `g_lastSystemButtonContentWidth`, a
small per-element (not per-cluster) cache keyed the same way
`g_lastArrangedTaskListWidth` is, consulted only when *both* the content
child's `DesiredSize` *and* the outer element's own `ActualWidth()` read 0
- the second condition is what distinguishes "genuinely hidden" (outer
element has been through a real Arrange pass; its `ActualWidth()` is
positive; a `DesiredSize` of 0 is trustworthy, matching the file's
existing "just-realized element" pattern for Start/task-list buttons) from
"still mid-realization" (outer element hasn't been arranged even once
yet; a fallback to the last known width is the correct call, if one
exists). This is treating an AI-review finding the way the standing
instructions ask - verified, not applied blindly, since the literal
version had a real, foreseeable regression for a live-toggleable Windows
setting the naive fix's own author likely wasn't tracking.

**The diagnostics machinery (`ResolveStats`/`LayoutPlanStats`/
`ArrangePassStats`, four atomic WinEvent counters) was NOT trimmed**,
despite being explicitly called out as heavy for a shipped mod. Every
single field in it has demonstrated real, specific diagnostic value either
in this file's own crash history (`skippedReentrant` and
`invalidateExceptions` were added in direct response to real crashes,
specifically to make a recurrence visible - see the drag-follow crash
narrative above) or within round 29's own live-test cycle (`hwndResolved`,
`ok`/`fail`, and `invalidated` were all read directly off `Wh_Log` output
to confirm the click-gate behavior and drag-follow side-switching worked
correctly). This mod has no formal test suite and no maintenance channel
other than a user sharing exactly these `Wh_Log` lines - trimming the one
tool that's repeatedly proven itself for exactly that purpose would trade
a modest, largely inert (a few integer increments per pass) runtime cost
for materially worse future debuggability. Declined with this reasoning
recorded rather than silently skipped, consistent with the standing
instruction to say so when a finding doesn't hold up rather than changing
working code to satisfy it.

**~15 of the review's ~39-46 `RATIONALE.md` pointers were folded back
into inline comments** (`EnsureTaskbarWnd`'s retry rationale, the
`g_winEventThreadMutex`/dedicated-thread/`TaskbarWndSubclassProc`
threading rationale, the `GetTaskItemsArrayOffset`/`ComputeSystemButtonX`/
drag-follow-trailing-timer failure-mode specifics, and a few others) -
specifically the ones that were genuinely bare pointers with no inline
reasoning at all, not the majority that already explained their own
"why" and used the pointer only for genuinely-extra history or depth. This
was raised and explicitly declined back in round 25 (see "Two round-25
suggestions deliberately left as-is" above) on the grounds that fully
reversing the round-23 trim wasn't this file's call to make unilaterally -
round 29 asked again, and this time the mod's author explicitly requested
it, which is exactly the missing authorization. `RATIONALE.md` remains the
long-form companion throughout; nothing was deleted from it, and most
pointers - the ones already carrying real inline context - were left
untouched rather than mechanically touching all of them regardless of
whether folding would add anything.

## Round 30: the click-gate poll, and a lossless UpdateVisualStates debounce

Round 30 reviewed round 29's actual shipped code (not a diff) and found
two things genuinely missed at the time - both confirmed correct against
the code before being fixed, not applied blindly.

**Finding 1: `g_realTaskbarClickObserved`'s gate had exactly the same
"permanent 2s poll" shape item 2 had just fixed for pinned-not-running
buttons - just for a different bail-out reason.** Before the session's
first real (non-sentinel) taskbar click, `ResolveHwndFromIndividualTaskItem`/
`ResolveHwndFromTaskGroup` both bailed at their own `g_realTaskbarClickObserved`
check with `clickDispatched` staying `false` - meaning `consecutiveFailures`
never advanced past 0, so `NextResolveDelayMs` saw every single button
(running ones included, unlike the `notRunning` case) as genuinely pending
at the fast `ResolveBackoffMs(0)` = 2s cadence, for as long as the session
went without a click. For a user who launches everything from Start or
switches windows with Alt+Tab, that could be the entire session - a
poll that costs a full repeater walk plus a `get_IsRunning` call per
running button, on the shell UI thread, forever.

Fixed by hoisting the `g_realTaskbarClickObserved` check out of both
`ResolveHwndFromIndividualTaskItem`/`ResolveHwndFromTaskGroup` (removed
from each; they now rely entirely on their sole caller,
`ResolveHwndFromTaskListButton`, having already checked it - there is no
other call site) and into `ResolveHwndFromTaskListButton` itself, ahead of
even the `TaskListButtonIsRunning` pre-check, since neither path can do
anything at all until the click gate opens regardless of running-state. A
new `ButtonHwndCacheEntry::awaitingFirstClick` field records the reason,
treated by `NextResolveDelayMs` exactly like `notRunning` (idle cadence,
not the fast one). Separately, `CTaskListWnd_HandleClick_Hook`'s
non-sentinel branch now does `g_realTaskbarClickObserved.exchange(true)`
and, on the transition from `false`, immediately sets
`g_forceResolveUnresolved` and calls `ArmButtonHwndResolveTimer(0)` - so
every button that was parked waiting for this exact gate gets rechecked
the instant the click happens, rather than up to 2 seconds later. This
needed two forward declarations (`extern std::atomic<bool>
g_forceResolveUnresolved;` and `void ArmButtonHwndResolveTimer(DWORD);`)
ahead of the click hook, since both are otherwise defined later in the
file than this hook is.

Note the asymmetry with `notRunning`: an `awaitingFirstClick` entry still
gets picked up by `TaskListButton_UpdateVisualStates_Hook`'s own
force-resolve nudge too (via `g_anyButtonNeedsRecheck`, below) - that's
harmless (a forced recheck on a button still awaiting the click gate just
finds itself still gated and re-caches the same state), not a second,
redundant fix for the same problem; the click hook's own immediate nudge
is what actually matters for closing the gate promptly.

**Finding 2: `TaskListButton::UpdateVisualStates`'s leading-edge throttle
could silently drop the specific transition it existed to catch, while
also running full resolve passes for pure hover noise once nothing was
left to resolve.** The hook fires for every visual-state transition of
every taskbar button - hover, press, focus, badges, not just running/
not-running - so a 150ms leading-edge gate (`if (GetTickCount64() -
g_lastUpdateVisualStatesArm < 150) return;`) could discard the one call
that mattered (a pinned app's running transition) if it landed within
150ms of an irrelevant one, with nothing left to ever re-arm afterward.
Since that button's cache entry is `notRunning` - now idle-cadenced at
30s - the icon could sit on its default side for up to 30 seconds. In the
other direction, once every button is already resolved, ordinary mouse
movement across the taskbar was still producing up to ~6-7 full resolve
passes/second (a repeater walk plus per-button property reads, on the
shell UI thread) for a hook that could find nothing new every single time.

Fixed by removing the elapsed-time gate entirely and instead calling
`ArmButtonHwndResolveTimer(150)` unconditionally on every call (still
gated by `g_anyButtonNeedsRecheck`, see below). This isn't a regression to
"no throttle" - `ArmButtonHwndResolveTimer` posts to the WinEventHook
thread, whose message loop responds to each post with a `KillTimer`/
`SetTimer(..., delayMs, ...)` re-arm (see `WinEventHookThreadProc`'s
`kArmResolveNowMsg` case) - so each new call *resets* the pending timer's
150ms countdown rather than adding a new one. A burst of calls therefore
collapses into exactly one resolve pass, fired 150ms after the burst
actually goes quiet: a lossless trailing-edge debounce, for free, without
needing a second timer variable the way `DragFollowTrailingTimerProc`
does for the exact same shape of problem on the drag-follow side. The old
`g_lastUpdateVisualStatesArm` throttle variable is gone entirely - nothing
reads elapsed time for this anymore.

The "hover churn over an already-settled taskbar" half of the finding
needed a separate mechanism, since debouncing alone doesn't stop a burst
from producing at least one pass - `g_anyButtonNeedsRecheck`, a plain
`bool` (not atomic: written only by `ResolvePendingButtonHwnds`, read
only by the `UpdateVisualStates` hook, and both run exclusively on the
taskbar thread, the same "single-thread-only, no marshal needed" pattern
`g_buttonHwndCache` itself already relies on). Recomputed at the end of
every `ResolvePendingButtonHwnds` pass: true if any cache entry has
`!hwnd && consecutiveFailures < kMaxResolveFailures` - deliberately
broader than `NextResolveDelayMs`'s own "pending" notion, since a
`notRunning`/`awaitingFirstClick` entry counts as "still worth reacting to
an event about" here even though it's idle for *polling* purposes; those
are genuinely different questions (one is "should the periodic timer
chase this on its own," the other is "if something just told us to look,
is there any point looking"). Starts `true` so the very first
`UpdateVisualStates` call of a session, before any resolve pass has run
to compute a real answer, isn't skipped.

**Two `NextResolveDelayMs`-adjacent comment corrections, both round-30
optional findings confirmed accurate on inspection:**
- The old comment claimed `NextResolveDelayMs`'s idle-worthy classification
  "must exactly match" `ResolvePendingButtonHwnds`' `backoffElapsed` check
  in every case. That's only true for the *terminal* case (`backoffElapsed`
  structurally evaluates false forever once `consecutiveFailures` reaches
  `kMaxResolveFailures`, so `NextResolveDelayMs` genuinely has to know to
  stop chasing it independently, or the original round-26 busy-loop
  returns). For `notRunning`/`awaitingFirstClick` entries, `backoffElapsed`
  has no special case at all - it just naturally evaluates true for them
  too once enough time passes, since their `consecutiveFailures` never
  advances past 0 - so they're never actually excluded from a real
  re-resolve, just given a slower cadence. The comment now says this
  explicitly instead of overclaiming an exact match everywhere.
- `RecomputeLayoutPlan`'s staleness-check comment claimed the
  `FullFootprintWidth` width-comparison was "short-circuited" the same way
  the `IsTaskListButton` class-name lookup is. It isn't, for an
  already-covered task list button specifically: the `&&` short-circuit in
  `widthIt != end() && std::abs(FullFootprintWidth(child) - ...)  > 0.5`
  only protects against calling `FullFootprintWidth` for a *non*-task-list
  child (where `widthIt` is always `end()`), not for the task list buttons
  the whole check exists to catch a width change on - those pay for two
  property reads (`Margin()` + `ActualWidth()`) every cheap-path pass,
  unconditionally. Left as-is rather than adding complexity to avoid two
  cheap reads per button; the comment now says so accurately instead of
  claiming a short-circuit that isn't there for the case that matters.

**`TaskListButtonIsRunning` now checks the HRESULT it was previously
discarding.** If `TaskListButton_get_IsRunning_Original` ever failed
(distinct from the symbol never resolving at all, which was already
handled), `isRunning` stayed at its initialized `false` and the button
got cached as `notRunning` - directly contradicting the function's own
documented fallback contract ("assume running - i.e. don't skip
anything") for exactly this kind of can't-tell situation. Fixed by
checking `FAILED(...)` and returning `true` on failure, the same fallback
already used for the symbol-not-resolved case.

**`GetMonitorCenterXLocal`/`GetTaskbarWidthLocal` now snapshot
`g_hTaskbarWnd`** into a local, matching the variable's own documented
contract - the same fix `EnsureTaskbarWnd` got in round 29's optional
items, extended to the two other multi-read call sites the round 30
review specifically named.

**`trackWindowPositions`'s settings description and both readme copies'
`no system-wide event hook(s)` wording** updated from "two...hooks" to
"a...hook," now that `EVENT_OBJECT_SHOW` is gone (removed in round 29;
only `EVENT_OBJECT_LOCATIONCHANGE` remains system-wide - the
`UpdateVisualStates` hook is in-process and scoped, not counted as one of
these).

**Not acted on** (both explicitly framed by the reviewer as informational,
not requests): `SystemButtonContentWidth`'s live-toggle-safe fallback
(round 29) assumes a hidden button keeps a non-zero outer `ActualWidth()`
via the negative-margin collapse trick specifically - if some future
Windows build ever hides one by genuinely collapsing it (`ActualWidth()`
also going to 0), the cached last-known width would be returned instead
and the reserved gap next to Start would never collapse. Worth knowing as
the failure mode if that specific visual bug is ever reported, not
something to defend against speculatively today. Similarly, the
single-miss click-sentinel latch (round 29) costing position tracking for
the rest of the session with only a `Wh_Log` line to explain it is a
known, deliberate tradeoff already reasoned through at length above - the
reviewer flagged it only as "worth knowing," not asking for a change.

## Round 31: replacing the desktop-wide WinEventHook, and full self-containment

Round 31 was the largest single round yet - one architectural mechanism
replacement, a full pass making the file's ~50 "See RATIONALE.md" pointers
self-contained (this doc no longer being required reading to understand
any single comment), and about a dozen smaller correctness/clarity fixes.
Everything below was confirmed against the actual code (or, in one case,
the Windhawk SDK's own header) before being acted on, not applied blindly.

**Finding 1 (required): the desktop-wide `EVENT_OBJECT_LOCATIONCHANGE`
WinEventHook was a real, externalized cost with a cheaper alternative.**
`WinEventHookThreadProc` registered
`SetWinEventHook(EVENT_OBJECT_LOCATIONCHANGE, EVENT_OBJECT_LOCATIONCHANGE,
nullptr, WinEventProc, 0, 0, WINEVENT_OUTOFCONTEXT)` with `idProcess`/
`idThread` both 0 - every process on the desktop. `EVENT_OBJECT_LOCATIONCHANGE`
is one of the highest-volume WinEvents there is (cursor, caret, every
window move/size/scroll), so simply having it registered made every
unrelated process on the system generate and marshal a notification this
mod discarded almost all of anyway - thousands of raw events/sec observed,
per this file's own former diagnostics, against `WinEventProc`'s
`idObject`/`g_resolvedHwnds` filters.

The mod never actually needed a global event feed: it already knows the
small set of HWNDs it cares about (`g_resolvedHwnds`, typically a handful),
and drag-follow was already collapsed to at most one relayout per ~150ms
by the trailing timer. Replaced with a direct poll: a `SetTimer` on the
existing background thread (`DragFollowPollTimerProc`, `kDragFollowPollMs`
= 150) walks `g_resolvedHwnds` and calls `GetWindowRect` on each -
`GetWindowRect` is a direct win32k.sys read, not a cross-process message
send, so the entire cost moves onto this mod's own dedicated thread
instead of externalizing it onto every other process on the system. A new
`g_lastPolledRect` map (thread-exclusive, no lock needed) detects an
actual position change per window and only invalidates layout when one is
found, which the old `WinEventProc` couldn't do (it invalidated on every
raw event that passed its filters, not just ones representing a real
change). `g_locationChangeHook`, `DragFollowTrailingTimerProc`, and
`WinEventProc` are gone; `g_dragFollowTrailingTimerId`/
`g_lastDragFollowInvalidate` are gone too, no longer needed once the poll
itself is the single source of both "did anything change" and "when did
we last check."

The poll also takes the refinement the review pointed out as a bonus of
this approach ("it would additionally let you invalidate only when a
window's *side or distance actually changed* instead of on every
intermediate position"), in the strongest form the classification
actually permits. `ClassifyByWindowPositionCached` derives *both* outputs
that matter - the side (window center vs. the screen's center line) and
`distanceFromCenter` (which orders same-side icons) - from the window's
horizontal center and nothing else. So comparing horizontal center alone
is not an approximation of "did the layout-relevant state change," it is
exactly that predicate: a vertical-only move, or a resize that leaves the
center where it was, provably cannot change any icon's placement, and now
fires no relayout at all (comparing whole rects, as the first cut did,
fired one on every tick of such a drag). `g_lastPolledCenterX` stores
`left + right` - twice the true center - since only equality matters and
staying integral keeps the comparison exact. Minimized windows are
skipped outright for the same reason: `GetWindowRect` reports a nonsense
off-screen position for one, and `ClassifyByWindowPositionCached`
deliberately freezes at the last pre-minimize classification instead of
using it, so polling a minimized window could only ever produce
relayouts that change nothing - two of them per minimize/restore cycle,
which the old WinEventHook also produced and which now simply don't
happen.

The reviewer's alternative (scope the hook to `idProcess` of the windows
actually being tracked, via `SetWinEventHook`'s own `idProcess` parameter)
was considered and rejected: `g_resolvedHwnds` changes as buttons resolve
and windows come and go, which would mean re-registering the hook (or one
per tracked process) continuously - real complexity for a narrower win
than simply not using WinEventHook at all for this.

**Renaming the thread infrastructure.** Round 29 already removed this
mod's other WinEventHook (`EVENT_OBJECT_SHOW`, replaced by
`TaskListButton::UpdateVisualStates`); after this round's fix, zero
WinEventHooks remain anywhere in the mod. Leaving `WinEventHookThreadProc`/
`StartWinEventHook`/`StopWinEventHook`/`StopWinEventHookInternal`/
`StopWinEventHookForToggle`/`g_winEventThread*` as names would now be
actively misleading, so all of them were renamed to
`BackgroundWorkerThreadProc`/`StartBackgroundWorkerThread`/
`StopBackgroundWorkerThread`/`StopBackgroundWorkerThreadInternal`/
`StopBackgroundWorkerThreadForToggle`/`g_backgroundWorkerThread*` via a
prefix-substring-safe bulk rename (`StopWinEventHook` is itself a prefix
of `StopWinEventHookInternal`/`StopWinEventHookForToggle`, so replacing
the shorter identifier first correctly transforms the longer ones as a
side effect). The two diagnostic counters were similarly renamed
(`g_winEventRawCount` → `g_dragFollowPollCount`, `g_winEventInvalidateCount`
→ `g_dragFollowInvalidateCount`), and the periodic stats `Wh_Log` line's
`winEvents: raw=%d` label became `dragFollow: polls=%d` to match. This was
slightly beyond the review's literal ask, but leaving stale WinEventHook
naming in place after removing the last actual WinEventHook would have
been exactly the kind of staleness this same review round was elsewhere
asking to clean up.

**Finding 2 (required): the readme understated the click-sentinel's real
failure mode, and the PR description had drifted from the code.** The
readme's "worst case" line only covered the failure mode where
`CTaskListWnd::HandleClick` never resolves at all (probing bails out
before dispatching anything - a real but benign "icons freeze on default
side" outcome). The worse mode: if the hook installs and real clicks route
through it (so `g_realTaskbarClickObserved` latches true) but a probe
click somehow reaches the taskbar's actual handler unswallowed,
`&g_clickSentinel` (a `WCHAR[]`) gets delivered as that handler's real
`winrt::Windows::System::LauncherOptions const&` argument - the handler
reads the sentinel string's bytes as an interface pointer, so the
realistic worst case is a stray window activate/minimize or an access
violation in explorer.exe. Fixed in both readme copies (`README.md` and
the in-source `WindhawkModReadme` block, which must stay identical) and
inlined into the click-sentinel dispatch function's own comment in the
source, replacing what had been a bare "see RATIONALE.md for the
failure-mode analysis" pointer.

The PR description separately claimed the latch trips "after 3 consecutive
unconfirmed misses" (code: `kClickSentinelMissesBeforeBroken = 1`, a
single miss) and that probing was partly driven by `EVENT_OBJECT_SHOW`
(removed in round 29, replaced by `TaskListButton::UpdateVisualStates`).
Both corrected via `gh pr edit`, along with updating the worst-case
description there too for the same reason as the readme.

Also fixed while in the same area: the readme's Start-menu limitation
("There's no exposed way to move the menu's own anchor point") reworded
to note this is solvable, not a dead end -
`taskbar-start-button-position.wh.cpp` already does exactly this by
hooking `StartMenuExperienceHost.exe` and repositioning the menu window
directly; this mod just doesn't do it today.

**Finding 3 (required): full self-containment - every "See RATIONALE.md"
pointer folded or dropped.** Round 29 had done a selective pass (~15 of
the most bare pointers); this round's review reframed full
self-containment as a main-body finding rather than optional, on the
reasoning that a merged mod's future maintainer (who may not be the
original author) shouldn't depend on a third-party repo staying up to
understand any given comment. All ~48 remaining per-comment pointers were
resolved - most were already fully self-contained once read closely and
just needed the trailing "See RATIONALE.md" clause dropped; a handful
(the click-sentinel dispatch failure mode above, `g_forceResolveUnresolved`'s
"why unconditional forcing was a bug" story, `ResolveAndCacheButtonHwnd`'s
crash-dump trigger conditions, `TaskbarWndSubclassProc`'s "unlike an
earlier design" note about the removed `RunFromWindowThread` fallback)
needed the actual reasoning inlined from this doc. This doc itself is
unaffected in role or structure - it remains the long-form companion for
anyone who wants more background than a comment carries on its own, which
is now how the top-of-file pointer to it is worded, rather than promising
answers to any specific "see RATIONALE.md" comment (none remain).

**Optional items addressed:**

- **`HookSymbols`' return value doesn't mean what several comments
  claimed.** Verified directly against `windhawk_utils.h`'s own doc
  comment (`@param optional If set to true, the absence of this symbol
  isn't considered an error`): a missing *optional* symbol never affects
  `HookSymbols`' boolean return value, only a missing *required* one does.
  Several comments (in `HandleLoadedModuleIfTaskbarView`, and the
  `Wh_ModInit` branch for a taskbar view module already loaded at init
  time) claimed the opposite - that a single missing optional
  HWND-resolution symbol "would otherwise fail this ENTIRE mod's load."
  Not gating on the return value was always the right call (the one
  symbol that must load, `ArrangeOverride`, is checked directly instead),
  just not for that reason - both comments now state the real one. This
  also meant the "Missing (optional...)" diagnostic `Wh_Log` blocks in
  `HookTaskbarDllSymbols`/`HookTaskbarViewDllSymbols`, both gated on
  `if (!ok)`, could only ever fire when a *required* symbol failed -
  exactly when naming which optional symbol is ALSO missing is least
  useful, and never in the scenario the diagnostic exists for (an optional
  symbol quietly disappearing after a Windows update while every required
  symbol still resolves). Both now compute `anyOptionalMissing` directly
  from the resolved-pointer checks and log unconditionally on that instead
  of on `!ok`. While fixing `HookTaskbarDllSymbols`' version, noticed its
  listed-symbols string only ever named 3 of the file's 7 actually-optional
  symbols (missing `CTaskListWnd::HandleClick` - the click interception
  point itself - along with the two `GetWindow` overloads and the
  `CImmersiveTaskItem` vftable); expanded it to cover all 7, since a
  diagnostic that fires unconditionally but silently omits the most
  important symbol it could report on isn't much of a fix.

- **Teardown-serializing mutex added to `StopBackgroundWorkerThreadInternal`.**
  The function nulled `g_backgroundWorkerThread` under
  `g_backgroundWorkerThreadMutex` and then waited on the thread handle
  *outside* that lock. A second concurrent call (e.g. `Wh_ModUninit`'s
  permanent stop arriving while a live `trackWindowPositions` toggle-off
  was still waiting) would see the already-nulled pointer and return
  immediately, letting its caller proceed while the worker thread was
  still executing mod code - right before Windhawk unmaps it. Not
  currently believed reachable in practice (`Wh_ModBeforeUninit`'s two
  `SendMessage` calls plus `RemoveWindowSubclassFromAnyThread` already
  serialize behind the taskbar thread's own pump), but a one-line
  `static std::mutex teardownMutex` wrapping the whole function turns that
  into a structural guarantee instead of an argument.

- **`g_lastSystemButtonContentWidth` pruning.** The map is keyed by the
  XAML element's raw ABI pointer with no reference held, and previously
  persisted across calls with no pruning at all - a taskbar/XamlRoot
  recreate could destroy a system button and let its address be reused by
  an unrelated new one, which would then silently inherit the destroyed
  element's stale cached width (worse than the "costs nothing" framing the
  old comment used, since this is a correctness risk, not just unbounded
  memory). Fixed inside `RecomputeLayoutPlan`, right after `childInfos` is
  built: computes the current pass's live system-button keys and erases
  any cache entry not among them, mirroring the pruning pattern already
  used for `g_lastKnownWindowClassification` and `g_lastPolledRect`
  elsewhere in the file.

- **`g_anyButtonNeedsRecheck`'s comment overclaimed a steady state that
  essentially never occurs.** The comment said hover churn "costs nothing"
  once the flag reaches its common steady state of "every button resolved
  or given up." In practice a pinned-but-not-running button - present on
  essentially every real Windows 11 taskbar by default - never advances
  past 0 `consecutiveFailures` (a `notRunning` entry never dispatches a
  click), so it keeps the flag true indefinitely, meaning every hover
  sweep still triggers a full `ResolvePendingButtonHwnds` pass. The
  reviewer's suggested code fix (excluding `notRunning` entries from the
  flag) was considered and declined: that would silently regress round
  29's whole point for this case - losing the fast, `UpdateVisualStates`-
  driven detection of a pinned app starting and falling back to the 30s
  idle tick instead - to optimize an already-cheap pass. Fixed the comment
  only, to describe the actual (still-intentional) behavior instead of a
  steady state that doesn't hold with a pinned-not-running app present.

- **`ComputeSystemButtonX`'s adjacent-left comment had its own conclusion
  backwards.** It said the taskbar-order-earliest button "ends up closest
  to Start." Working through the math (`widthAfter = clusterWidth -
  widthBefore - ownWidth`, then `X = startCenterX - startWidth/2 - gap -
  widthAfter - ownWidth`) for both the earliest button (`widthBefore = 0`,
  so `widthAfter` is largest) and the latest one confirms the earliest
  button actually lands at the *outermost* (farthest-from-Start) slot -
  which is what the comment's own second clause (reading left-to-right
  matches far-left mode's order) already correctly depended on. Fixed the
  first clause to say "farthest from Start (the outermost slot)."

- **A benign startup race no longer logs as an error.** Live testing this
  round surfaced `ArmButtonHwndResolveTimer: PostThreadMessage failed,
  error=1444` (`ERROR_INVALID_THREAD_ID`) on every single mod startup - a
  pre-existing condition, not something this round introduced.
  `StartBackgroundWorkerThread` publishes the worker's thread id the
  moment `CreateThread` returns, but a thread has no message queue until
  it first calls a message function (`PeekMessage`, at the top of
  `BackgroundWorkerThreadProc`), and `EnsureTaskbarWnd` reaches its
  `ArmButtonHwndResolveTimer(0)` call in the very same pass that started
  the thread. The nudge was redundant regardless - the thread proc arms
  the first resolve itself - so this now suppresses that one error code
  and keeps logging every other. Nothing about the behavior changed; the
  log just no longer opens every session with a failure that isn't one.

- **Review-history narration stripped from several comments** (round
  29/30 attributions in `TaskListButtonIsRunning`'s comment,
  `ResolveHwndFromTaskListButton`'s `awaitingFirstClick` comment,
  `ButtonHwndCacheEntry::notRunning`'s comment, `TaskListButton_UpdateVisualStates_Hook`'s
  re-arm comment, and `DragFollowPollTimerProc`'s own banner) while
  keeping every conclusion - a file read by a future maintainer, not a
  past reviewer, shouldn't carry "round 30 review finding:" as if it were
  a changelog.

**Not acted on** (explicitly declined, matching round 25's precedent for
the same item): the crowded-side compression fallback in
`PlanTaskListButtons` still lets icons overlap once `compressibleAvailable`
goes negative, rather than clamping the scale at a floor and letting
outermost icons run past the bound or fall through to native positioning.
Already documented in the readme as a known limitation; a genuinely
crowded taskbar is an edge case, and the suggested change (a new
compression floor plus a decision about what happens to icons past it)
is new design surface, not a fix to something broken, the same
reasoning round 25 used to decline reshaping this same fallback.

## Round 32: making the poll idle-cheap, and unwinding a latch that outlived its reasoning

Round 32 confirmed CI green on all three targets and all four earlier
blocking items genuinely fixed, and left two required findings.

**Finding 1: the drag-follow poll ran at 150 ms forever.** Round 31
replaced a desktop-wide WinEventHook with a fixed 150 ms poll, which
traded a cost imposed on every process for ~6.7 wakeups/second in
explorer.exe that never stopped — including with nothing tracked, with
the click-sentinel path latched off, and with the machine idle. The
reviewer noted that sub-second timers in the shell get pushed back on
consistently in this repo, and this was worse than the 500-750 ms cases
that had drawn objections before. (This was flagged as a known tradeoff
when round 31 shipped, with the mitigation already identified; it came
back exactly as expected.)

Fixed with an adaptive cadence rather than the reviewer's first
suggestion of an `EVENT_SYSTEM_MOVESIZEEND` hook.
`DragFollowPollTimerProc` now runs at `kDragFollowPollActiveMs` (150 ms)
while anything is moving, and drops to `kDragFollowPollIdleMs` (1 s)
once `kDragFollowActiveWindowMs` (2 s) has passed with no detected
change; any change snaps it straight back. A drag produces a change on
nearly every tick, so the fast rate covers the whole drag and its
settle, while an ordinary desktop — which is where the machine sits
essentially all of the time — pays one wakeup per second. Nothing is
missed at the idle rate: it does the same comparison per tracked window,
so a new drag is picked up within one idle interval. The empty-`tracked`
case the reviewer called out falls out of this for free, since "nothing
tracked" is just a special case of "nothing changed."

`MOVESIZEEND` was considered and rejected on three counts. It would
reintroduce a desktop-wide WinEventHook one round after removing the
last one; it only fires when a drag *finishes*, so icons would stop
following live mid-drag (a visible regression against the behavior just
tested); and by the reviewer's own caveat it misses programmatic moves
(`Win`+arrow snap, third-party window managers), so it needs a backstop
poll anyway — leaving two mechanisms where the adaptive poll needs one.
The volume argument for it is sound, but it buys nothing the adaptive
poll doesn't already get more simply.

**Finding 2: `kClickSentinelMissesBeforeBroken = 1` permanently disabled
position tracking on a single miss.** The threshold of 1 was justified,
several rounds ago, by "with zero evidence yet that interception works, a
single miss is reason enough." Round 30 quietly invalidated that: hoisting
the `g_realTaskbarClickObserved` check to the top of
`ResolveHwndFromTaskListButton` (the sole caller of both resolve paths)
means no probe can run until a real click has already passed through
`CTaskListWnd_HandleClick_Hook` — so by the time any probe happens, the
hook is provably installed and provably reached. A miss after that isn't
evidence of a broken interception point; it's the same transient class
this file already calls "usually innocent" for post-confirmation misses,
and the first probes of a session are when those transients are most
likely, since buttons are still realizing. The consequence of getting it
wrong was severe and silent: every running app falls back to
`unresolvedAppsDefaultSide`, the whole task list piles onto one side of
Start, and nothing recovers short of reloading the mod. Round 31's own
live test showed the scenario — a taskbar rebuild produced `fail=9` in
one pass; those were post-confirmation so nothing latched, but the same
churn arriving before a path's first capture would have killed tracking
for the session.

Raised to 3, which keeps the fail-closed guarantee (three stray clicks
per path worst case, still bounded, still well short of a runaway) while
making an accidental permanent latch on startup churn much less likely.
A successful capture now also zeroes that path's miss counter. That
second half is belt-and-suspenders as the code stands, since
`NoteUnconfirmedClickSentinelMiss` already stops counting once
`Confirmed` is set and a capture is exactly what sets it — but it makes
"counts consecutive unconfirmed misses" true of the counter on its own
rather than only of the two flags read together, which is what the
constant's name claims.

**Optional items taken:** the per-call `MonitorFromWindow`/
`GetMonitorInfo` pair in `ClassifyByWindowPositionCached` is now resolved
once per plan recompute in `RecomputeLayoutPlan` and threaded down
through `PlanTaskListButtons`/`ClassifyTaskListButton` as a
`const MONITORINFO*` (null preserves the old per-call failure behavior
exactly — fall back to the default side); `EnsureTaskbarWnd`'s
`EnumWindows` retry is throttled to `kTaskbarWndEnumRetryMs` while the
taskbar window is unresolved, instead of enumerating every top-level
window on the desktop at layout frequency; `TaskListButtonIsRunning`
now credits `taskbar-labels.wh.cpp`, which it was adapted from, the way
this file's other borrowed techniques already are; the readme heading
now matches the mod's `@name` instead of being a second name for the
same thing in the same file; and the comment blocks the reviewer named
were trimmed to keep their conclusions and drop the deliberation, with
the duplicated nested-Arrange crash argument reduced to a
cross-reference at its second site. Note the tension with round 31, which
required *inlining* reasoning for self-containment: these are
reconcilable, since self-contained means "don't point at an external doc
for the decision," not "state the same decision in three places."

**Not acted on:** removing the diagnostics scaffolding
(`ResolveStats`/`LayoutPlanStats`/`ArrangePassStats` and the periodic
`Wh_Log`) — declined in an earlier round with reasoning that still
stands, and this round's own drag-follow verification was read directly
off that log line. And gating the `UpdateVisualStates` nudge per button
via `(void**)pThis + 3`: it would introduce a new hardcoded ABI-offset
assumption, of the kind this file otherwise only takes on where a
symbol can't provide the answer, to optimize a path that is already
cheap.

**Also worth recording from this round's functionality notes:** the
reviewer pushed back *in the mod's favor* on the unattended-probing
question the PR description flags. `taskbar-numberer` is merged in the
catalog and fires the same click-sentinel probe from its own
`TaskListButton::UpdateVisualStates` hook, for every button, with no
confirmation gate, no broken-path latch and no backoff at all. Against
that precedent this mod's version is the more conservative design, not
the riskier one — worth weighing it against that rather than against the
gesture-only mods the description currently cites.
