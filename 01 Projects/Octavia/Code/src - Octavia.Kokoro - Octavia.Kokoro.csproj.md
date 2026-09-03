---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Kokoro\Octavia.Kokoro.csproj
---

# src\Octavia.Kokoro\Octavia.Kokoro.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <!--
    Her voice, in a process of its own.

    This exists as a separate executable for one hard reason: sherpa-onnx ships its own native
    `onnxruntime.dll`, and `Octavia.Core` carries Microsoft's for Silero and the wake word. Two
    of those in one output folder is a native collision, and the same rule already put Piper and
    the local brain outside the process. So Octavia.Core never references this project - it
    starts it, writes sentences to its input and reads PCM from its output.

    It deliberately has no reference to Octavia.Core either. Nothing about her is in here.
  -->
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>Octavia.Kokoro</RootNamespace>
    <AssemblyName>octavia-kokoro</AssemblyName>
    <RuntimeIdentifier>win-x64</RuntimeIdentifier>
    <SelfContained>false</SelfContained>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="org.k2fsa.sherpa.onnx" Version="1.13.5" />
  </ItemGroup>

</Project>
```
