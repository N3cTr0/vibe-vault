---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PrintersInfo.cs
---

# PartnerTool\PrintersInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record PrinterInfo(string Name, bool Default, bool Offline, string Status, string Port, string Driver)
{
    public string Display => (Default ? "★ " : "") + Name;
    public string Detail  => $"{(Offline ? "Offline" : Status)}   ·   {Port}   ·   {Driver}";

    public bool   Protected => PrintersInfo.IsProtected(Name);
    public bool   CanRemove => !Protected;
    public string RemoveTip => Protected
        ? "Built into Windows - cannot be removed"
        : $"Remove {Name}";

    /// <summary>Ticked in the Manage list. Only ever set on rows where CanRemove is true.</summary>
    public bool Selected { get; set; }
}

/// <summary>Installed printers, the default, and whether each is offline (Win32_Printer).</summary>
public static class PrintersInfo
{
    public static List<PrinterInfo> Collect()
    {
        var list = new List<PrinterInfo>();
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT Name, Default, WorkOffline, PrinterStatus, PortName, DriverName FROM Win32_Printer");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                string status = Convert.ToInt32(o["PrinterStatus"] ?? 0) switch
                {
                    1 => "Other", 2 => "Unknown", 3 => "Idle", 4 => "Printing",
                    5 => "Warming up", 6 => "Stopped", 7 => "Offline", _ => "-",
                };
                list.Add(new PrinterInfo(
                    o["Name"]?.ToString() ?? "Printer",
                    o["Default"] is bool d && d,
                    o["WorkOffline"] is bool w && w,
                    status,
                    o["PortName"]?.ToString() ?? "",
                    o["DriverName"]?.ToString() ?? ""));
            }
        }
        catch { }
        return list.OrderByDescending(p => p.Default).ThenBy(p => p.Name, StringComparer.OrdinalIgnoreCase).ToList();
    }

    /// <summary>
    /// Windows' own virtual printers. Deleting these breaks Print to PDF / XPS / Fax for every user
    /// on the machine and they can only be restored through Optional Features, so removal is blocked.
    /// Matched by prefix so RDP-redirected copies ("... (redirected 2)") are covered too.
    /// </summary>
    private static readonly string[] Builtin =
    {
        "Microsoft Print to PDF",
        "Microsoft XPS Document Writer",
        "Fax",
    };

    public static bool IsProtected(string name) =>
        !string.IsNullOrWhiteSpace(name) &&
        Builtin.Any(b => name.StartsWith(b, StringComparison.OrdinalIgnoreCase));

    /// <summary>Delete one printer queue. Protected built-ins are refused.</summary>
    public static (bool Ok, string Message) Remove(string name)
    {
        if (string.IsNullOrWhiteSpace(name)) return (false, "No printer selected.");
        if (IsProtected(name))               return (false, $"“{name}” is built into Windows and cannot be removed.");

        try
        {
            // SELECT * on purpose: a projection returns partial instances with no object path, and
            // Delete() then throws "operation is not valid due to the current state of the object".
            // Matching in C# rather than a WQL filter keeps user-supplied names out of the query.
            using var q = new ManagementObjectSearcher("SELECT * FROM Win32_Printer");
            foreach (ManagementObject o in q.Get())
                using (o)
                {
                    if (!string.Equals(o["Name"]?.ToString(), name, StringComparison.OrdinalIgnoreCase)) continue;
                    o.Delete();
                    return (true, $"Removed “{name}”.");
                }
            return (false, $"“{name}” was not found - it may already be gone.");
        }
        catch (Exception ex) { return (false, ex.Message); }
    }

    /// <summary>Remove the named printers. Protected built-ins are refused by <see cref="Remove"/>.</summary>
    public static (int Removed, List<string> Errors) RemoveMany(IEnumerable<string> names)
    {
        int removed = 0;
        var errors = new List<string>();
        foreach (var name in names)
        {
            var (ok, msg) = Remove(name);
            if (ok) removed++;
            else errors.Add($"{name}: {msg}");
        }
        return (removed, errors);
    }
}
```
