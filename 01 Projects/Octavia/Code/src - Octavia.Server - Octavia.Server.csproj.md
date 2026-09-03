---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Server\Octavia.Server.csproj
---

# src\Octavia.Server\Octavia.Server.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <!--
    Her, and the tray icon that manages her. One executable with three modes: no arguments
    gives the tray (install, start, stop, restart, and her settings); the service switch runs
    the service itself, in session 0, with no UI at all; and the console switch runs her here
    and prints, which is what you want while watching her start.

    It was two executables for one release, and the reason given was that a Windows service
    has no desktop. That is true, and it never required a second *binary* only a second
    *mode*: the tray is always launched by a person in their own session, and the service
    never draws anything. Merged in v0.47.0.

    It stays a console application, and that was tried the other way first. WinExe removes the
    console window a double-click leaves behind, and silently breaks every switch a person
    types: a shell does not wait for a windowed process, so output races the prompt, the exit
    code is lost, and the secret prompt has no console to read a key from. Console flash on a
    double-click is the cheaper cost, and the tray frees the console the moment it starts.
  -->
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <UseWPF>true</UseWPF>
    <UseWindowsForms>true</UseWindowsForms>
    <AssemblyName>Octavia.Server</AssemblyName>
    <!-- Her mark, borrowed rather than copied. The icon belongs to *her*, not to any one
         executable, and a desktop shortcut showing a generic console window beside one
         showing Octavia reads as two unrelated programs. -->
    <ApplicationIcon>..\Octavia.App\Assets\octavia.ico</ApplicationIcon>
    <!-- AttachConsole is a LibraryImport, and the generator writes that with unsafe. -->
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
  </PropertyGroup>

  <ItemGroup>
    <!-- Her tray mark, shared with the client rather than copied: two files that must stay
         identical is one more thing to forget. -->
    <Content Include="..\Octavia.App\Assets\octavia-tray.ico" Link="Assets\octavia-tray.ico"
             CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\Octavia.Core\Octavia.Core.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="System.ServiceProcess.ServiceController" Version="10.0.11" />
  </ItemGroup>

</Project>
```
