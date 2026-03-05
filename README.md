Torch Fork Fixes — yankeevader/Torch

1. NLog Double Console Logging — Rule Ordering Bug
File: Torch.Server/NLog.config
The Keen Info rule was missing final="true", causing Keen Info messages to match both the Keen rule AND the wildcard * rule, printing to console twice.
xml<!-- Before -->
<logger name="Keen" minlevel="Info" writeTo="console, wpf"/>

<!-- After -->
<logger name="Keen" minlevel="Info" writeTo="console, wpf" final="true"/>

2. Runtime Console Rule Injection
File: Torch.Server/Initializer.cs ~line 67
A #if !DEBUG block dynamically added a second console rule at runtime after NLog.config was already loaded, doubling all console output regardless of NLog.config settings.
csharp// Removed:
LogManager.Configuration.AddRule(LogLevel.Info, LogLevel.Fatal, "console");
LogManager.ReconfigExistingLoggers();

3. NLog Config Overwrite on Torch Update
File: Torch/TorchConfig.cs
OverwriteGlobalNLogConfigOnUpdate defaults to true, meaning every Torch self-update pulled from the upstream Jenkins release overwrites the user's NLog.config with the unfixed upstream version, silently re-introducing the double logging bug. Defaulting to false prevents this until upstream fixes their NLog.config.
csharp// Before
public bool OverwriteGlobalNLogConfigOnUpdate { get; set; } = true;

// After
public bool OverwriteGlobalNLogConfigOnUpdate { get; set; } = false;

4. SteamCMD Runscript Path Bug (../../)
File: Torch.Server/Initializer.cs ~line 37
The RUNSCRIPT property used a relative path (../../) for force_install_dir which resolved incorrectly depending on working directory, causing SteamCMD to install SE to the wrong location or fail entirely. Fixed by using Directory.GetCurrentDirectory() for an absolute path.
csharp// Before
private static string RUNSCRIPT => $@"force_install_dir ../../
login anonymous
app_update 298740
quit";

// After
private static string RUNSCRIPT => $@"force_install_dir {Directory.GetCurrentDirectory()}
login anonymous
app_update 298740
quit";

5. SteamCMD Premature Exit During Long Downloads
File: Torch.Server/Initializer.cs ~lines 250-290
The while (!cmd.HasExited) loop exited prematurely when stdout closed mid-download (common during large downloads), causing Torch to continue executing before SE finished downloading. Applied to both the init run and the update run loops.
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

6. steam_api64.dll Copy Crash on Fresh Install
File: Torch.Server/Initializer.cs ~line 90
The DLL copy had no existence check, so on a fresh install where DedicatedServer64 didn't exist yet, it threw a FileNotFoundException and crashed silently before the unhandled exception handler was fully wired.
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
NOTE:  Delete steamapps,  steamcmd, and DedicatedServer64. this forces a redownload of SteamCMD, and Space Engineers Dedicated Server.
