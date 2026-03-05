Torch Fork Fixes — yankeevader/Torch

1. NLog Double Console Logging — Rule Ordering Bug
File: Torch.Server/NLog.config
The Keen Info rule was missing final="true", causing Keen Info messages to match both the Keen rule AND the wildcard * rule, printing to console twice. Adding final="true" to the Keen Info rule stops processing after it's handled.
xml<!-- Before -->
<logger name="Keen" minlevel="Info" writeTo="console, wpf"/>

<!-- After -->
<logger name="Keen" minlevel="Info" writeTo="console, wpf" final="true"/>

2. NLog Config Overwrite on Update
File: Torch/TorchConfig.cs
OverwriteGlobalNLogConfigOnUpdate defaults to true, meaning every Torch update pulled from the Jenkins/upstream release would overwrite the user's NLog.config with the unfixed version, silently re-introducing the double logging bug. Temporarily defaulting to false prevents this until the upstream NLog.config is corrected.
csharp// Before
public bool OverwriteGlobalNLogConfigOnUpdate { get; set; } = true;

// After
public bool OverwriteGlobalNLogConfigOnUpdate { get; set; } = false;

3. Runtime Console Rule Injection
File: Torch.Server/Initializer.cs ~line 67
Removed the #if !DEBUG block that dynamically added a second console rule at runtime, doubling all output regardless of NLog.config.
csharp// Removed:
LogManager.Configuration.AddRule(LogLevel.Info, LogLevel.Fatal, "console");
LogManager.ReconfigExistingLoggers();

4. SteamCMD Premature Exit
File: Torch.Server/Initializer.cs ~lines 250-290
Replaced the while (!cmd.HasExited) loop with a proper null-check ReadLine pattern plus WaitForExit(). The old loop exited when stdout closed during long downloads, causing Torch to crash mid-install.
csharp// Before
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
Applied to both the init run and the update run loops.

5. steam_api64.dll Copy Crash on Fresh Install
File: Torch.Server/Initializer.cs ~line 90
Wrapped the DLL copy in a File.Exists() guard so a fresh install doesn't crash when DedicatedServer64 doesn't exist yet.
csharp// Before
File.Copy(apiSource, apiTarget);

// After
if (File.Exists(apiSource))
{
    if (!File.Exists(apiTarget))
        File.Copy(apiSource, apiTarget);
    else if (File.GetLastWriteTime(apiTarget) < File.GetLastWriteTime(apiSource))
    {
        File.Delete(apiTarget);
        File.Copy(apiSource, apiTarget);
    }
}

Net Result: Fresh installs complete the full 7.9 GB SE download without crashing, SE 1.208.15 loads correctly, console logging is clean, and the fix survives Torch updates.
Torch was stuck on 1.207.22
Attempting to update using the standard Torch release failed.
NOTE:  Delete steamapps,  steamcmd, and DedicatedServer64. this forces a redownload of SteamCMD, and Space Engineers Dedicated Server.
