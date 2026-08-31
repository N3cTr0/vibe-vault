---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Octavia.App.csproj
---

# src\Octavia.App\Octavia.App.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net10.0-windows10.0.19041.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWPF>true</UseWPF>
    <UseWindowsForms>true</UseWindowsForms>
    <RootNamespace>Octavia</RootNamespace>
    <AssemblyName>Octavia</AssemblyName>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <Version>0.15.2</Version>
  </PropertyGroup>

  <ItemGroup>
    <!-- The WindowsDesktop implicit-using set omits these two. -->
    <Using Include="System.IO" />
    <Using Include="System.Net.Http" />
    <InternalsVisibleTo Include="EarsTest" />
  </ItemGroup>

  <ItemGroup>
    <None Remove="wwwroot\**" />
    <Content Include="wwwroot\**" CopyToOutputDirectory="PreserveNewest" />
    <None Remove="Assets\**" />
    <Content Include="Assets\**" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Anthropic" Version="12.44.0" />
    <PackageReference Include="Microsoft.ML.OnnxRuntime" Version="1.29.0" />
    <PackageReference Include="Microsoft.Web.WebView2" Version="1.0.4191.47" />
    <PackageReference Include="NAudio" Version="3.0.1" />
    <PackageReference Include="System.Speech" Version="10.0.11" />
    <PackageReference Include="Whisper.net" Version="1.9.1" />
    <PackageReference Include="Whisper.net.Runtime" Version="1.9.1" />
    <PackageReference Include="Whisper.net.Runtime.Cuda" Version="1.9.1" />
  </ItemGroup>

</Project>
```
