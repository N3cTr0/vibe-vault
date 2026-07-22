---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PartnerTool.csproj
---

# PartnerTool\PartnerTool.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net10.0-windows10.0.17763.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWPF>true</UseWPF>
    <ApplicationIcon>Resources\logo.ico</ApplicationIcon>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <Version>0.23.0</Version>
    <!-- The exe techs get is the single-file publish (publish-singlefile.bat /
         build-installer.bat: PublishSingleFile + IncludeNativeLibrariesForSelfExtract),
         which the MSI installs to C:\PCI\PartnerTool as one file. -->
    <SatelliteResourceLanguages>en</SatelliteResourceLanguages>
  </PropertyGroup>

  <ItemGroup>
    <!-- No RID-specific / native packages: everything left is managed, so plain
         `dotnet build` works with no RuntimeIdentifier and the single-file publish
         sets -r win-x64 itself. (LibreHardwareMonitorLib was removed in 0.23.0 —
         its ring-0 sensor driver never returned values on our fleet and tripped
         memory-integrity/ASR on locked-down machines.) -->
    <PackageReference Include="System.Management" Version="10.0.2" />
  </ItemGroup>

  <ItemGroup>
    <Resource Include="Resources\logo.png" />
    <Resource Include="Resources\logo.ico" />
  </ItemGroup>

</Project>
```
