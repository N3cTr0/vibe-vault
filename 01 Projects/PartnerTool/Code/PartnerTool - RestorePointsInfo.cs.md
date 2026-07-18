---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\RestorePointsInfo.cs
---

# PartnerTool\RestorePointsInfo.cs

```csharp
using System.Diagnostics;
using System.Management;

namespace PartnerTool;

public record RestorePoint(int Sequence, string Description, DateTime Created, string Type);

/// <summary>
/// Existing System Restore points (root\default SystemRestore). We list them and hand off
/// the actual restore to the built-in rstrui.exe wizard — rolling a machine back is a
/// reboot-and-pray operation best left to the OS UI with its own confirmation.
/// </summary>
public static class RestorePointsInfo
{
    public static List<RestorePoint> Collect()
    {
        var list = new List<RestorePoint>();
        try
        {
            using var q = new ManagementObjectSearcher(@"\\.\root\default", "SELECT * FROM SystemRestore");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                DateTime created = DateTime.MinValue;
                try { created = ManagementDateTimeConverter.ToDateTime(o["CreationTime"]?.ToString() ?? ""); }
                catch { }
                string type = Convert.ToInt32(o["RestorePointType"] ?? -1) switch
                {
                    0  => "Application install",
                    1  => "Application uninstall",
                    10 => "Device driver install",
                    12 => "Modify settings",
                    13 => "Canceled operation",
                    _  => "Manual / checkpoint",
                };
                list.Add(new RestorePoint(
                    Convert.ToInt32(o["SequenceNumber"] ?? 0),
                    o["Description"]?.ToString() ?? "Restore point",
                    created, type));
            }
        }
        catch { /* System Restore may be disabled, or access denied */ }
        return list.OrderByDescending(r => r.Sequence).ToList();
    }

    /// <summary>Open the Windows System Restore wizard so a tech can roll back interactively.</summary>
    public static void OpenWizard()
        => Process.Start(new ProcessStartInfo(ProcessRunner.ResolveSystemTool("rstrui.exe")) { UseShellExecute = true });
}
```
