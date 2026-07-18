---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SystemManagement.cs
---

# PartnerTool\SystemManagement.cs

```csharp
using System.IO;
using System.Management;
using System.Text.RegularExpressions;

namespace PartnerTool;

public record EnvVar(string Scope, string Name, string Value);
public record OptionalFeature(string Name, bool Enabled);
public record UserProfileItem(string Path, DateTime? LastUsed, bool Loaded, string Sid)
{
    public string Name => System.IO.Path.GetFileName(Path.TrimEnd('\\'));
    public bool CanDelete { get; init; }   // false for the current user / loaded (in-use) profiles
}

/// <summary>
/// Smaller system-management readers grouped together: hosts file, environment variables,
/// Windows optional features and user profiles.
/// </summary>
public static class SystemManagement
{
    /// <summary>A Windows SID: "S-1-" then hyphen-separated decimal sub-authorities. Nothing else.</summary>
    private static readonly Regex SidPattern = new(@"^S-1-\d+(-\d+)*$", RegexOptions.Compiled);

    // ── Hosts file ────────────────────────────────────────────
    public static string HostsPath =>
        Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.System), @"drivers\etc\hosts");

    public static string ReadHosts()
    {
        try { return File.Exists(HostsPath) ? File.ReadAllText(HostsPath) : ""; }
        catch (Exception ex) { return $"# Could not read hosts file: {ex.Message}"; }
    }

    public static (bool ok, string message) SaveHosts(string content)
    {
        try { File.WriteAllText(HostsPath, content); return (true, "Saved."); }
        catch (Exception ex) { return (false, ex.Message); }
    }

    // ── Environment variables ─────────────────────────────────
    public static List<EnvVar> EnvVars()
    {
        var list = new List<EnvVar>();
        void Add(string scope, System.Collections.IDictionary d)
        {
            foreach (System.Collections.DictionaryEntry kv in d)
                list.Add(new EnvVar(scope, kv.Key?.ToString() ?? "", kv.Value?.ToString() ?? ""));
        }
        try { Add("Machine", Environment.GetEnvironmentVariables(EnvironmentVariableTarget.Machine)); } catch { }
        try { Add("User",    Environment.GetEnvironmentVariables(EnvironmentVariableTarget.User)); } catch { }
        return list.OrderBy(v => v.Scope).ThenBy(v => v.Name, StringComparer.OrdinalIgnoreCase).ToList();
    }

    /// <summary>
    /// Create or update an environment variable. <paramref name="machine"/> = true writes the
    /// System (Machine) scope, which needs elevation (we run elevated) and persists in the registry;
    /// .NET broadcasts WM_SETTINGCHANGE so new processes pick it up (already-running apps won't).
    /// The name is validated — a real variable name has no '=', whitespace or control characters.
    /// </summary>
    public static (bool ok, string message) SetEnvVar(string name, string value, bool machine)
    {
        name = (name ?? "").Trim();
        value ??= "";
        if (name.Length == 0) return (false, "Enter a variable name.");
        if (name.Length > 255 || name.Any(c => c == '=' || char.IsWhiteSpace(c) || char.IsControl(c)))
            return (false, "Invalid name — no spaces, '=' or control characters.");
        if (value.Length == 0) return (false, "Enter a value.");

        var target = machine ? EnvironmentVariableTarget.Machine : EnvironmentVariableTarget.User;
        try
        {
            ActivityLog.Action("Environment", $"Set {(machine ? "System" : "User")} variable {name}={value}");
            Environment.SetEnvironmentVariable(name, value, target);
            return (true, $"Set {(machine ? "System" : "User")} variable “{name}”. New processes will see it.");
        }
        catch (Exception ex) { return (false, ex.Message); }
    }

    // ── Windows optional features ─────────────────────────────
    public static List<OptionalFeature> OptionalFeatures()
    {
        var list = new List<OptionalFeature>();
        try
        {
            using var q = new ManagementObjectSearcher("SELECT Name, InstallState FROM Win32_OptionalFeature");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                var name = o["Name"]?.ToString();
                if (string.IsNullOrEmpty(name)) continue;
                list.Add(new OptionalFeature(name, Convert.ToInt32(o["InstallState"] ?? 0) == 1));
            }
        }
        catch { }
        return list.OrderByDescending(f => f.Enabled).ThenBy(f => f.Name, StringComparer.OrdinalIgnoreCase).ToList();
    }

    // ── User profiles ─────────────────────────────────────────
    public static List<UserProfileItem> UserProfiles()
    {
        var list = new List<UserProfileItem>();

        // The account running this tool — never offer to delete its own profile.
        string currentSid = "";
        try { currentSid = System.Security.Principal.WindowsIdentity.GetCurrent().User?.Value ?? ""; } catch { }

        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT LocalPath, LastUseTime, Loaded, Special, SID FROM Win32_UserProfile WHERE Special=False");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                var path = o["LocalPath"]?.ToString();
                if (string.IsNullOrEmpty(path)) continue;
                DateTime? last = null;
                try { if (o["LastUseTime"] != null) last = ManagementDateTimeConverter.ToDateTime(o["LastUseTime"].ToString()); }
                catch { }
                var loaded = o["Loaded"] is bool b && b;
                var sid    = o["SID"]?.ToString() ?? "";
                list.Add(new UserProfileItem(path, last, loaded, sid)
                {
                    CanDelete = !loaded && sid.Length > 0 && sid != currentSid,
                });
            }
        }
        catch { }
        return list.OrderByDescending(p => p.LastUsed).ToList();
    }

    /// <summary>
    /// Remove a user profile the *correct* way — via the Win32_UserProfile WMI delete, which deletes
    /// the profile folder, removes the <c>ProfileList\&lt;SID&gt;</c> registry entry and unloads the
    /// NTUSER.DAT hive (exactly what Advanced System Properties ▸ User Profiles ▸ Delete does).
    /// Refuses system/special profiles and any profile that's currently loaded (signed in).
    /// </summary>
    public static (bool ok, string message) DeleteUserProfile(string sid)
    {
        if (string.IsNullOrWhiteSpace(sid)) return (false, "No SID for this profile.");
        // The SID is interpolated into a WQL query that drives an ELEVATED, irreversible delete, so
        // validate its shape rather than trying to escape it: a SID is always S-1-<digits/hyphens>.
        // Anything else can't be a real profile and is refused outright.
        if (!SidPattern.IsMatch(sid)) return (false, "That doesn't look like a valid SID — refused.");
        try
        {
            using var q = new ManagementObjectSearcher(
                $"SELECT * FROM Win32_UserProfile WHERE SID = '{sid}'");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                if (o["Special"] is bool sp && sp) return (false, "That's a system profile — refused.");
                if (o["Loaded"]  is bool ld && ld) return (false, "Profile is in use — sign that user out first.");
                o.Delete();   // folder + ProfileList\<SID> registry key + NTUSER.DAT hive, all together
                return (true, "Profile removed (folder, registry entry and data).");
            }
            return (false, "Profile not found — it may already be removed.");
        }
        catch (Exception ex) { return (false, ex.Message); }
    }
}
```
