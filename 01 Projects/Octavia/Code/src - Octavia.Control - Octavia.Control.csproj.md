---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Control\Octavia.Control.csproj
---

# src\Octavia.Control\Octavia.Control.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <!--
    Her server's tray icon and its settings window.

    **A separate process because of session 0.** The server runs as a Windows service, and a
    service has no desktop: a tray icon drawn from inside it would be drawn where nobody can
    see it. So the thing a person clicks has to live in their own session and control the
    service from outside, which is what this does.

    It configures her by writing `config.json` and sealing secrets - the same two mechanisms
    the server reads at startup - rather than by talking to a running session. That keeps it
    working when she is stopped, which is exactly when somebody needs to change a setting that
    is stopping her.
  -->
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <UseWPF>true</UseWPF>
    <UseWindowsForms>true</UseWindowsForms>
    <AssemblyName>Octavia.Control</AssemblyName>
    <RootNamespace>Octavia.Control</RootNamespace>
    <ApplicationIcon>..\Octavia.App\Assets\octavia.ico</ApplicationIcon>
  </PropertyGroup>

  <ItemGroup>
    <!-- Her tray mark, shared with the client rather than copied: two files that must stay
         identical is one more thing to forget. -->
    <Content Include="..\Octavia.App\Assets\octavia-tray.ico" Link="Assets\octavia-tray.ico"
             CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

  <ItemGroup>
    <!-- For `OctaviaConfig`, `Paths`, `SecretStore`, `ServerControl` and `Log`. Nothing here
         constructs an `OctaviaSession`; the split check asserts that stays true. -->
    <ProjectReference Include="..\Octavia.Core\Octavia.Core.csproj" />
  </ItemGroup>

</Project>
```
