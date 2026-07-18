---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ServicesInfo.cs
---

# PartnerTool\ServicesInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record ServiceItem(string Name, string DisplayName, string State, string StartMode, string LogOn)
{
    public bool Running => State.Equals("Running", StringComparison.OrdinalIgnoreCase);
}

/// <summary>
/// Windows services list with inline control (start / stop / restart / change start type),
/// driven entirely through WMI Win32_Service methods so we need no extra dependency.
/// </summary>
public static class ServicesInfo
{
    public static List<ServiceItem> Collect()
    {
        var list = new List<ServiceItem>();
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT Name, DisplayName, State, StartMode, StartName FROM Win32_Service");
            foreach (ManagementObject o in q.Get())
            using (o)
                list.Add(new ServiceItem(
                    o["Name"]?.ToString() ?? "",
                    o["DisplayName"]?.ToString() ?? "",
                    o["State"]?.ToString() ?? "",
                    o["StartMode"]?.ToString() ?? "",
                    o["StartName"]?.ToString() ?? ""));
        }
        catch { }
        return list.OrderBy(s => s.DisplayName, StringComparer.OrdinalIgnoreCase).ToList();
    }

    /// <summary>
    /// Services that must not be stopped/restarted from here — stopping them can leave the
    /// session or the whole machine unusable (RPC, DCOM, the service control manager, etc.),
    /// or strips the machine's security protection (firewall, Defender, EDR agents).
    /// </summary>
    private static readonly HashSet<string> Critical = new(StringComparer.OrdinalIgnoreCase)
    {
        // Core Windows — stopping these can hang or bugcheck the session
        "RpcSs", "DcomLaunch", "RpcEptMapper", "LSM", "Power", "Winmgmt", "Schedule",
        "BrokerInfrastructure", "SystemEventsBroker", "gpsvc", "ProfSvc", "CoreMessagingRegistrar",
        "PlugPlay", "EventLog", "SamSs",
        // Firewall / network filtering — stopping breaks the network stack and drops protection
        "mpssvc", "BFE",
        // Security / endpoint protection — never stop AV/EDR from a support tool
        "WinDefend", "Sense", "wscsvc", "SecurityHealthService", "WdNisSvc",
        "SentinelAgent", "SentinelHelperService", "SentinelStaticEngine",
        "HuntressAgent", "HuntressRio", "CSFalconService", "CylanceSvc",
    };

    public static bool IsCritical(string name) => Critical.Contains(name);

    /// <summary>
    /// WMI object paths are built by string interpolation and this app runs ELEVATED, so rather than
    /// trying to escape a hostile service name we refuse it. A real service key never contains a
    /// quote, backslash, backtick or control character — anything that does isn't a service we
    /// should be touching, and would otherwise be a way to break out of the path.
    /// </summary>
    private static bool ValidServiceName(string name) =>
        !string.IsNullOrWhiteSpace(name) && name.Length <= 256 &&
        !name.Any(c => c is '\'' or '"' or '\\' or '`' || char.IsControl(c));

    public static (bool ok, string message) Control(string name, string action)
    {
        if (!ValidServiceName(name)) return (false, "Invalid service name — refused.");
        if ((action is "stop" or "restart") && IsCritical(name))
            return (false, "Critical system service — blocked (stopping it can make Windows unusable).");
        try
        {
            using var svc = new ManagementObject($"Win32_Service.Name='{name}'");
            uint Invoke(string method) => Convert.ToUInt32(svc.InvokeMethod(method, null));

            uint r = action switch
            {
                "start" => Invoke("StartService"),
                "stop"  => Invoke("StopService"),
                "restart" => Restart(svc),
                _ => 0xFFFFFFFF,
            };
            return (r == 0, Result(r));
        }
        catch (Exception ex) { return (false, ex.Message); }
    }

    private static uint Restart(ManagementObject svc)
    {
        var stop = Convert.ToUInt32(svc.InvokeMethod("StopService", null));
        // Wait briefly for it to actually stop before starting again.
        for (int i = 0; i < 20; i++)
        {
            svc.Get();
            if (svc["State"]?.ToString() == "Stopped") break;
            System.Threading.Thread.Sleep(250);
        }
        return Convert.ToUInt32(svc.InvokeMethod("StartService", null));
    }

    public static (bool ok, string message) SetStartMode(string name, string mode)
    {
        if (!ValidServiceName(name)) return (false, "Invalid service name — refused.");
        if (mode is not ("Automatic" or "Manual" or "Disabled"))
            return (false, "Invalid start mode — refused.");
        try
        {
            using var svc = new ManagementObject($"Win32_Service.Name='{name}'");
            var args = svc.GetMethodParameters("ChangeStartMode");
            args["StartMode"] = mode;   // "Automatic" | "Manual" | "Disabled"
            uint r = Convert.ToUInt32(svc.InvokeMethod("ChangeStartMode", args, null)["ReturnValue"]);
            return (r == 0, Result(r));
        }
        catch (Exception ex) { return (false, ex.Message); }
    }

    private static string Result(uint code) => code switch
    {
        0 => "OK",
        2 => "Access denied",
        3 => "Dependent services running",
        5 => "Service already in that state",
        7 => "Timed out",
        10 => "Already running",
        _ => $"Result code {code}",
    };
}
```
