---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Octavia.App.csproj
---

# src\Octavia.App\Octavia.App.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <!--
    The Windows client: a window, a WebView2, a tray icon and a hotkey.

    It holds no session, no brain, no ears and no voice. It is a browser that knows where
    she lives and how to introduce itself - which is precisely what the Android client has
    always been, and the reason this stage could be a move rather than a rewrite.

    It keeps the name, the icon and the manifest because it is still the thing a person
    double-clicks. It simply stopped containing her.
  -->
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <UseWPF>true</UseWPF>
    <UseWindowsForms>true</UseWindowsForms>
    <AssemblyName>Octavia</AssemblyName>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <ApplicationIcon>Assets\octavia.ico</ApplicationIcon>
    <!-- `Native.cs` registers the global hotkey through LibraryImport, which generates it. -->
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>

  <ItemGroup>
    <None Remove="Assets\**" />
    <Content Include="Assets\**" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

  <ItemGroup>
    <!--
      For `Log`, `Paths` and `RemoteKey` - not for her. Nothing here constructs an
      `OctaviaSession`, and a check asserts that it never starts doing so again.
    -->
    <ProjectReference Include="..\Octavia.Core\Octavia.Core.csproj" />
    <PackageReference Include="Microsoft.Web.WebView2" Version="1.0.4191.47" />
  </ItemGroup>

</Project>
```
