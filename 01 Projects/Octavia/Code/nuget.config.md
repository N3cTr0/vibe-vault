---
project: Octavia
tags: [octavia, code]
source-path: nuget.config
---

# nuget.config

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- Pin nuget.org so `dotnet restore/build/publish` works on any fresh machine.
     A bare user NuGet config can have an empty <packageSources>, in which case
     restore fails with NU1100 for every package including the Windows SDK ref pack.
     <clear/> makes this the authoritative source list (ignores machine-global feeds).
     Same fix, and same reason, as PartnerTool's. -->
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
```
