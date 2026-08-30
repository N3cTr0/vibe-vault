---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\EarsTest.csproj
---

# tools\EarsTest\EarsTest.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0-windows10.0.19041.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWPF>true</UseWPF>
    <UseWindowsForms>true</UseWindowsForms>
  </PropertyGroup>

  <ItemGroup>
    <Using Include="System.IO" />
    <ProjectReference Include="..\..\src\Octavia.App\Octavia.App.csproj" />
  </ItemGroup>

</Project>
```
