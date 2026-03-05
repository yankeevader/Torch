# Torch Fork Fixes — yankeevader/Torch

## Summary

This fork addresses several bugs affecting fresh installs and console logging behavior, all present in the upstream TorchAPI/Torch codebase.
NOTE: I mostly made this for myself to fix my 10 Nexus servers stuck on 1.207, I assume I havent followed proper methods for passing this on to Torch proper so here it sits.

---

## Fix 1 — NLog Double Console Logging (Rule Ordering)

**File:** `Torch.Server/NLog.config`

The Keen Info rule was missing `final="true"`, causing Keen Info messages to match both the Keen-specific rule **and** the wildcard `*` rule, resulting in every line printing to the console twice.

```xml
<!-- Before -->
<logger name="Keen" minlevel="Info" writeTo="console, wpf"/>

<!-- After -->
<logger name="Keen" minlevel="Info" writeTo="console, wpf" final="true"/>
```

---

## Fix 2 — Runtime Console Rule Injection

**File:** `Torch.Server/Initializer.cs` (~line 67)

A `#if !DEBUG` block dynamically added a second console logging rule at runtime, **after** `NLog.config` was already loaded. This doubled all console output regardless of what was configured in `NLog.config`.

```csharp
// Removed from the #if !DEBUG block:
LogManager.Configuration.AddRule(LogLevel.Info, LogLevel.Fatal, "console");
LogManager.ReconfigExistingLoggers();

// Kept:
#if !DEBUG
    AppDomain.CurrentDomain.UnhandledException += HandleException;
#endif
```

---

## Fix 3 — NLog Config Overwritten on Torch Self-Update

**File:** `Torch/TorchConfig.cs`

`OverwriteGlobalNLogConfigOnUpdate` defaulted to `true`, meaning every Torch self-update pulled from the upstream Jenkins release would silently overwrite the local `NLog.config` with the upstream unfixed version, re-introducing the double logging bug on every update.

Changed the default to `false` so the corrected `NLog.config` survives updates.

```csharp
// Before
public bool OverwriteGlobalNLogConfigOnUpdate { get; set; } = true;

// After
public bool OverwriteGlobalNLogConfigOnUpdate { get; set; } = false;
```

> **Note:** The existing description in code warns against changing this, which is sound advice for end users modifying an unknown config. However, since this fork ships with a corrected `NLog.config`, overwriting it with the upstream broken version is the wrong default behavior here.

---

## Fix 4 — SteamCMD Runscript Incorrect Relative Path

**File:** `Torch.Server/Initializer.cs` (~line 37)

The `RUNSCRIPT` property used a relative `../../` path for `force_install_dir`, which resolved incorrectly depending on the working directory at runtime. This caused SteamCMD to install Space Engineers to the wrong location or fail entirely.

Fixed by using `Directory.GetCurrentDirectory()` to produce an absolute path.

```csharp
// Before
private static string RUNSCRIPT => $@"force_install_dir ../../
login anonymous
app_update 298740
quit";

// After
private static string RUNSCRIPT => $@"force_install_dir {Directory.GetCurrentDirectory()}
login anonymous
app_update 298740
quit";
```

---

## Fix 5 — SteamCMD Premature Exit During Large Downloads

**File:** `Torch.Server/Initializer.cs` (~lines 250–290)

The stdout read loop used `while (!cmd.HasExited)` with `ReadLine()` and `Thread.Sleep(100)`. During large downloads (SE is ~7.9 GB), the stdout stream closes or disconnects before the process actually exits. When `ReadLine()` returns `null` in this state, the loop condition still passes and Torch continues executing before SteamCMD has finished.

Fixed by replacing both loops (init run and update run) with a null-check `ReadLine()` pattern followed by `WaitForExit()`.

```csharp
// Before
while (!cmd.HasExited)
{
    log.Info(cmd.StandardOutput.ReadLine());
    Thread.Sleep(100);
}

// After
string line;
while ((line = cmd.StandardOutput.ReadLine()) != null)
{
    log.Info(line);
}
cmd.WaitForExit();
```

Applied to both the **SteamCMD init run** and the **DS update run** loops.

---

## Fix 6 — `steam_api64.dll` Copy Crash on Fresh Install

**File:** `Torch.Server/Initializer.cs` (~line 90)

The `steam_api64.dll` copy had no existence guard. On a fresh install where `DedicatedServer64` does not yet exist (because SteamCMD hasn't finished downloading SE), `File.Copy()` threw a `FileNotFoundException`. This exception occurred before the unhandled exception handler was fully wired, causing Torch to crash silently with no useful log output.

```csharp
// Before
File.Copy(apiSource, apiTarget);

// After
if (File.Exists(apiSource))
{
    if (!File.Exists(apiTarget))
    {
        File.Copy(apiSource, apiTarget);
    }
    else if (File.GetLastWriteTime(apiTarget) < File.GetLastWriteTime(apiSource))
    {
        File.Delete(apiTarget);
        File.Copy(apiSource, apiTarget);
    }
}
```

---

## Net Result

- Fresh installs complete the full ~7.9 GB SE download without crashing
- SE 1.208.15 loads correctly
- Console logging is clean (no duplicate lines)
- Fixes survive Torch self-updates
