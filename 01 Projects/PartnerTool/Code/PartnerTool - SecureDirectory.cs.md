---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\SecureDirectory.cs
---

# PartnerTool\SecureDirectory.cs

```csharp
using System.IO;
using System.Security.AccessControl;
using System.Security.Principal;

namespace PartnerTool;

/// <summary>
/// Creates and locks down directories the elevated app trusts (its logs, and the folder it
/// downloads/executes vendor tools from). The default ACL inherited from <c>C:\</c> lets
/// authenticated users create files in freshly-made subfolders — which would let a standard user
/// plant a binary (or a junction) that this admin-level process then reads, writes or executes.
/// Hardening strips that: only Administrators + SYSTEM may write; everyone else is read-only.
/// </summary>
public static class SecureDirectory
{
    public static void EnsureHardened(string path)
    {
        try
        {
            var di = Directory.CreateDirectory(path);

            var admins = new SecurityIdentifier(WellKnownSidType.BuiltinAdministratorsSid, null);
            var system = new SecurityIdentifier(WellKnownSidType.LocalSystemSid, null);
            var users  = new SecurityIdentifier(WellKnownSidType.BuiltinUsersSid, null);

            const InheritanceFlags inherit = InheritanceFlags.ContainerInherit | InheritanceFlags.ObjectInherit;

            var sec = new DirectorySecurity();
            sec.SetAccessRuleProtection(true, false);   // disable inheritance + drop the C:\ "create" grants
            sec.AddAccessRule(new FileSystemAccessRule(admins, FileSystemRights.FullControl,    inherit, PropagationFlags.None, AccessControlType.Allow));
            sec.AddAccessRule(new FileSystemAccessRule(system, FileSystemRights.FullControl,    inherit, PropagationFlags.None, AccessControlType.Allow));
            sec.AddAccessRule(new FileSystemAccessRule(users,  FileSystemRights.ReadAndExecute, inherit, PropagationFlags.None, AccessControlType.Allow));
            try { sec.SetOwner(admins); } catch { /* needs SeTakeOwnership; harmless if it fails */ }

            di.SetAccessControl(sec);
        }
        catch { /* best effort — never block the app on an ACL failure */ }
    }
}
```
