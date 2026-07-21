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
    <Version>0.22.0</Version>
    <!-- Default build/publish is a folder (exe + dependency DLLs), which the MSI
         installs to C:\PCI\PartnerTool. The single-file exe handed to techs is a
         separate publish (publish-singlefile.bat: PublishSingleFile +
         IncludeNativeLibrariesForSelfExtract) that self-extracts the native
         LibreHardwareMonitorLib bits at runtime. -->
    <SatelliteResourceLanguages>en</SatelliteResourceLanguages>
    <!-- LibreHardwareMonitorLib ships its implementation assembly as a RID-specific
         asset (runtimes\win-x64\...). A build with no RID compiles against the ref
         assembly but never copies the real DLL, so default every build to win-x64.
         The publish profile sets its own RID + SelfContained, so these conditional
         defaults don't fight it. -->
    <RuntimeIdentifier Condition="'$(RuntimeIdentifier)' == ''">win-x64</RuntimeIdentifier>
    <SelfContained Condition="'$(SelfContained)' == ''">false</SelfContained>
    <!-- Keep the dev build at bin\Release\<tfm>\ instead of a win-x64 subfolder, even
         though we now set a RID. (Publish has its own PublishDir, so this doesn't affect it.) -->
    <AppendRuntimeIdentifierToOutputPath>false</AppendRuntimeIdentifierToOutputPath>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="LibreHardwareMonitorLib" Version="0.9.6" />
    <PackageReference Include="System.Management" Version="10.0.2" />
  </ItemGroup>

  <ItemGroup>
    <Resource Include="Resources\logo.png" />
    <Resource Include="Resources\logo.ico" />
  </ItemGroup>

</Project>
```
