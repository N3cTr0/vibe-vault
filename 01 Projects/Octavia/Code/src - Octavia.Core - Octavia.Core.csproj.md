---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Octavia.Core.csproj
---

# src\Octavia.Core\Octavia.Core.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <!--
    Her. Everything except the thing you look at.

    The session, the brain, the ears, the voice, the rooms and the socket all live here, and
    nothing in this project knows whether a window exists. That was already true before the
    split - see Stage 15 - which is why this project is a move rather than a rewrite.

    `UseWPF` is gone as of v0.33.0. It was on for exactly one method - `Sight.Inspect` greying
    a camera still with `BitmapFrame` - which now decodes with ImageSharp instead: managed,
    cross-platform, and measured producing the same numbers.

    **The comment it replaces called that "the single thing standing between this project and
    a plain net10.0 target", and that was wrong.** NAudio, System.Speech and WebView2 are all
    still referenced here and all Windows-only, so the target has not moved. What went is one
    of several blockers, and the one that could be removed without deciding anything else.

    The rest belongs to Stage 15 item 2, and wants item 3 done first: item 3 moves the audio
    devices out to the client, which is most of what an `Octavia.Windows` project would
    otherwise have been created to hold.
  -->
  <PropertyGroup>
    <UseWPF>true</UseWPF>
    <AssemblyName>Octavia.Core</AssemblyName>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>

  <ItemGroup>
    <!--
      Three assemblies, one program.

      A public surface here would advertise an API that nothing outside this repository will
      ever call, and would force `public` onto forty types the design deliberately keeps
      closed. The server and the client are not consumers of a library; they are the same
      program split for deployment. So they see internals, exactly as the checks do.
    -->
    <InternalsVisibleTo Include="Octavia.Server" />
    <!-- The client's *assembly* name, which is not its project name: it has been `Octavia`
         since v0.1.0 because it is the thing a person double-clicks. -->
    <InternalsVisibleTo Include="Octavia" />
    <InternalsVisibleTo Include="EarsTest" />
  </ItemGroup>

  <ItemGroup>
    <!--
      Her page travels with the core, not with the client, because it is the *server* that
      serves it. A client is a browser pointed at an origin and holds no copy - which is
      what keeps one renderer in one place however many faces attach.
    -->
    <None Remove="wwwroot\**" />
    <Content Include="wwwroot\**" CopyToOutputDirectory="PreserveNewest" />
    <None Remove="Assets\**" />
    <Content Include="Assets\**" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Anthropic" Version="12.44.0" />
    <PackageReference Include="Microsoft.ML.OnnxRuntime" Version="1.29.0" />
    <!-- Not for rendering: `SystemReport` reports the installed runtime's version, which is
         a fact a diagnostics bundle needs whether or not this machine draws anything. -->
    <PackageReference Include="Microsoft.Web.WebView2" Version="1.0.4191.47" />
    <PackageReference Include="NAudio" Version="3.0.1" />
    <PackageReference Include="SixLabors.ImageSharp" Version="2.1.11" />
    <PackageReference Include="System.Speech" Version="10.0.11" />
    <PackageReference Include="Whisper.net" Version="1.9.1" />
    <PackageReference Include="Whisper.net.Runtime" Version="1.9.1" />
    <PackageReference Include="Whisper.net.Runtime.Cuda" Version="1.9.1" />
  </ItemGroup>

</Project>
```
