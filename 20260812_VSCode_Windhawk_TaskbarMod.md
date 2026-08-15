# Windhawk Taskbar Mod — Session Summary (2026-08-12)

**File:** `taskbar-centered-start-split-icons.wh.cpp`
**Mod ID:** `taskbar-centered-start-split-icons`
**Previous session:** `20260808_VSCode_Windhawk_TaskbarMod.md` (cold-boot inertness fix, `unresolvedAppsDefaultSide` third option, `pinnedAppsAnchor` setting)

One piece of work today: a crash fix reported from real use — enabling a second
monitor while the mod was active crashed explorer.exe.

## Bug fix: enabling a second monitor crashed explorer.exe

### Symptom

With the mod active, enabling another monitor (via Windows Display Settings)
crashed explorer.exe, repeating in a tight loop (Windows auto-restarting it, then
crashing again within seconds) for as long as the second monitor stayed enabled.

### Investigation

Rather than guess from the symptom, checked the Windows Event Log directly first:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1000} -MaxEvents 15 |
  Where-Object { $_.Message -match 'explorer\.exe' } |
  Select-Object TimeCreated, Message
```

Every crash record showed the same faulting module and the same fault offset:

```
Faulting application name: Explorer.EXE
Faulting module name: Windows.UI.Xaml.dll
Exception code: 0xc000027b
Fault offset: 0x00000000008f9cc3
```

`0xC000027B` is `STATUS_STOWED_EXCEPTION` — the OS's signal for an unhandled C++
exception crossing a COM/ABI boundary (converted to a fail-fast crash rather than
propagating normally). An identical, repeating fault offset across multiple crash
events pointed at a deterministic code bug, not memory corruption, so the next
step was finding what in the mod could throw across that specific boundary.

`IUIElement_Arrange_Hook` replaces the process-wide `IUIElement::Arrange` vtable
slot, so it's invoked directly by XAML's own native call sites for every
`UIElement` being arranged anywhere in `explorer.exe` — a raw ABI boundary with no
C++/WinRT exception translation on the far side. Grepping the file for `.as<`
turned up exactly one hit, inside that hook:

```cpp
auto repeater =
    Media::VisualTreeHelper::GetParent(element).as<FrameworkElement>();
```

Every other cast in the file consistently uses `.try_as<T>()` (returns null on
failure); this one line used `.as<T>()`, which throws instead. `GetParent()`
returns a null `DependencyObject` for an element that has no parent yet — which
happens transiently while a *brand new* secondary-monitor taskbar's XAML tree is
being constructed from scratch, exactly when a second monitor gets enabled. The
hook fires for that tree too (it's gated only by `g_inTaskbarArrangeOverride`,
true for any taskbar's `ArrangeOverride`, primary or secondary — the mod's own
readme claims secondary monitors are "not specially handled," but nothing in the
code actually skips them). `.as<FrameworkElement>()` on that null parent throws,
and with no exception boundary between the throw and XAML's native caller, the
process fail-fasts.

### Solution

1. Changed `.as<FrameworkElement>()` to `.try_as<FrameworkElement>()` — matches
   the existing `if (!repeater) { ...; return original(); }` handling right below
   it, which was already written to handle a null result gracefully; it just
   never got the chance to run.
2. Wrapped the whole `IUIElement_Arrange_Hook` body in `try { ... } catch (...) {
   return original(); }` as a safety net. The file had no exception handling
   anywhere, and this function is the highest-risk boundary in it (a raw
   vtable-slot replacement called for every XAML element in the process, not just
   taskbar ones) — worth hardening against any *future* WinRT call added here
   that might throw in some other edge case, not just this specific one.
3. Added an `exceptions` counter to `ArrangePassStats`, surfaced in the existing
   throttled `Wh_Log` diagnostic line, so if this safety net ever gets exercised
   again it's visible in the logs instead of silently swallowing a crash-shaped
   problem.

**Status:** fix is in the file; not yet retested by the user with a second
monitor enabled (planned for tomorrow).

## Process notes

- Checking the Windows Event Log (`Get-WinEvent`, Application log, event ID 1000)
  for the exact faulting module/exception code/offset before touching any code
  was decisive here — it turned "crashes when I enable a monitor" into a specific,
  reproducible, one-line root cause in a single step, rather than needing several
  rounds of the "add diagnostic counters and re-test" loop used for past bugs
  (which doesn't work as well for a hard crash anyway, since there's no
  opportunity to log *after* the fault). Worth doing this first for any future
  report that's an actual crash rather than just wrong behavior.
- No local Windhawk build toolchain in this directory — as in past sessions, the
  change is syntax/brace-balance-checked locally but the real compile happens
  through the Windhawk app itself, on the user's next test pass.
- No version bump (`@version 0.1.0` unchanged), consistent with prior sessions.
