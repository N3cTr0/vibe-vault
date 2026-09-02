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
    <!-- Her mark, borrowed rather than copied. The icon belongs to *her*, not to either
         executable, and a desktop shortcut showing a generic console window beside one
         showing Octavia reads as two unrelated programs. -->
    <ApplicationIcon>..\Octavia.App\Assets\octavia.ico</ApplicationIcon>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\Octavia.Core\Octavia.Core.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="System.ServiceProcess.ServiceController" Version="10.0.11" />
  </ItemGroup>

</Project>
```
