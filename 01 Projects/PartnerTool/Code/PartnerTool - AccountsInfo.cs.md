---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\AccountsInfo.cs
---

# PartnerTool\AccountsInfo.cs

```csharp
using System.Management;

namespace PartnerTool;

public record LocalAccount(
    string Name, string FullName, bool Disabled, bool Locked, bool PasswordNeverExpires, bool IsAdmin)
{
    public string RoleText => IsAdmin ? "Administrator" : "Standard user";
    public string Detail
    {
        get
        {
            var p = new List<string>();
            if (!string.IsNullOrWhiteSpace(FullName)) p.Add(FullName);
            p.Add(RoleText);
            if (Disabled)             p.Add("disabled");
            if (Locked)               p.Add("locked out");
            if (PasswordNeverExpires) p.Add("password never expires");
            return string.Join("   ·   ", p);
        }
    }
}

/// <summary>
/// Local user accounts on the machine, flagging which are members of the local
/// Administrators group (SID S-1-5-32-544) and whose passwords never expire — a quick
/// security/hygiene check techs reach for.
/// </summary>
public static class AccountsInfo
{
    public static List<LocalAccount> Collect()
    {
        var admins = new HashSet<string>(StringComparer.OrdinalIgnoreCase);
        try
        {
            using var grp = new ManagementObjectSearcher(
                "SELECT * FROM Win32_Group WHERE LocalAccount=True AND SID='S-1-5-32-544'");
            foreach (ManagementObject g in grp.Get())
            using (g)
                foreach (ManagementBaseObject m in g.GetRelated("Win32_UserAccount"))
                using (m)
                    if (m["Name"]?.ToString() is { Length: > 0 } n) admins.Add(n);
        }
        catch { }

        var list = new List<LocalAccount>();
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT Name, FullName, Disabled, Lockout, PasswordExpires FROM Win32_UserAccount WHERE LocalAccount=True");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                var name = o["Name"]?.ToString() ?? "";
                if (string.IsNullOrEmpty(name)) continue;
                list.Add(new LocalAccount(
                    name,
                    o["FullName"]?.ToString()?.Trim() ?? "",
                    o["Disabled"] is bool d && d,
                    o["Lockout"]  is bool l && l,
                    o["PasswordExpires"] is bool pe && !pe,   // PasswordExpires=false => never expires
                    admins.Contains(name)));
            }
        }
        catch { }

        return list.OrderByDescending(a => a.IsAdmin).ThenBy(a => a.Name, StringComparer.OrdinalIgnoreCase).ToList();
    }
}
```
