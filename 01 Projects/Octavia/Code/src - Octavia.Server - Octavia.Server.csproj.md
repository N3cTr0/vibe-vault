---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\Octavia.Server.csproj
---

# src\Octavia.Server\Octavia.Server.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <!--
    Her, with nothing to look at. A console application on purpose rather than a service:
    the first thing anybody does with a new server is watch it start, and a service that
    fails before it writes its first log line is diagnosed by guesswork. Registering it as a
    service is Stage 15 item 4, and wants this to be boring first.
  -->
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <AssemblyName>Octavia.Server</AssemblyName>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\Octavia.Core\Octavia.Core.csproj" />
  </ItemGroup>

</Project>
```
