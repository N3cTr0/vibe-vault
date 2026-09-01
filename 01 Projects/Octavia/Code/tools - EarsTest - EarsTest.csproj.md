---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\EarsTest.csproj
---

# tools\EarsTest\EarsTest.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <!--
    The checks run against `Octavia.Core`, which is her. They never referenced the window
    even when the window was the whole application, so the Stage 15 split changed this file
    by one line - which is the strongest evidence available that the seam was already in the
    right place.

    WPF and WinForms stay on: `EmbedderChecks` drives the real page in a WebView2, which is
    a window whether or not anybody looks at it.
  -->
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <UseWPF>true</UseWPF>
    <UseWindowsForms>true</UseWindowsForms>
    <RootNamespace>EarsTest</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\Octavia.Core\Octavia.Core.csproj" />
    <PackageReference Include="Microsoft.Web.WebView2" Version="1.0.4191.47" />
  </ItemGroup>

</Project>
```
