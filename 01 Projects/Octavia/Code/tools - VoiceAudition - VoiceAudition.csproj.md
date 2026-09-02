---
project: Octavia
tags: [octavia, code]
source-path: tools\VoiceAudition\VoiceAudition.csproj
---

# tools\VoiceAudition\VoiceAudition.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <!--
    Stage 16 research, deliberately free-standing.

    It does NOT reference `Octavia.Core`, and that is the whole point rather than an
    oversight. sherpa-onnx ships its own native `onnxruntime.dll`; Octavia.Core carries
    Microsoft.ML.OnnxRuntime for Silero and the wake word. Two copies of that DLL in one
    output folder is the kind of native collision that takes an evening to diagnose, so the
    candidate engine is auditioned in a process that has never met the incumbent one.

    Nothing here ships. If a voice is chosen, the winner moves into `Octavia.Core\Voice`
    and this project is deleted.
  -->
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>VoiceAudition</RootNamespace>
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="org.k2fsa.sherpa.onnx" Version="1.13.5" />
  </ItemGroup>

</Project>
```
