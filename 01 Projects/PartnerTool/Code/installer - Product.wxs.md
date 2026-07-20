---
project: PartnerTool
tags: [partnertool, code]
source-path: installer\Product.wxs
---

# installer\Product.wxs

```xml
<Wix xmlns="http://wixtoolset.org/schemas/v4/wxs">
  <Package Name="Partner Tool"
           Manufacturer="Progressive Computing"
           Version="0.20.2.0"
           UpgradeCode="7C9E6A2B-4F1D-4E8A-9B3C-2A1D5E6F7B80"
           Scope="perMachine"
           Compressed="yes">

    <!-- Replace older installs cleanly -->
    <MajorUpgrade DowngradeErrorMessage="A newer version of Partner Tool is already installed." />
    <MediaTemplate EmbedCab="yes" />

    <!-- Force the install root to C: so the path resolves to C:\PCI\PartnerTool -->
    <Property Id="ROOTDRIVE" Value="C:\" />

    <!-- Add/Remove Programs metadata -->
    <Icon Id="AppIcon.ico" SourceFile="$(sys.SOURCEFILEDIR)..\PartnerTool\Resources\logo.ico" />
    <Property Id="ARPPRODUCTICON" Value="AppIcon.ico" />
    <Property Id="ARPHELPLINK" Value="https://www.progressivecomputing.com" />
    <Property Id="ARPURLINFOABOUT" Value="https://www.progressivecomputing.com" />

    <!-- Start Menu folder -->
    <StandardDirectory Id="ProgramMenuFolder">
      <Directory Id="ShortcutDir" Name="Partner Tool" />
    </StandardDirectory>

    <!-- C:\PCI\PartnerTool  (PCIDIR is implicitly rooted at TARGETDIR = C:\).
         This MSI installs the folder publish (exe + dependency DLLs + native sensor
         bits) — the optional folder-install path. The single-file exe
         (publish-singlefile.bat) is the primary hand-out to techs. -->
    <Directory Id="PCIDIR" Name="PCI">
      <Directory Id="INSTALLFOLDER" Name="PartnerTool" />
    </Directory>

    <Feature Id="Main" Title="Partner Tool" Level="1">
      <!-- Harvest every file produced by the FolderInstall publish profile.
           Paths are relative to this .wxs (sys.SOURCEFILEDIR = installer\), so the
           MSI builds regardless of where the repo is cloned. -->
      <Files Directory="INSTALLFOLDER"
             Include="$(sys.SOURCEFILEDIR)..\release\**" />
      <ComponentRef Id="AppShortcut" />
    </Feature>

    <!-- Start Menu shortcut to the installed exe + uninstall cleanup -->
    <Component Id="AppShortcut" Directory="ShortcutDir" Guid="*">
      <Shortcut Id="StartMenuShortcut" Name="Partner Tool"
                Target="[INSTALLFOLDER]PartnerTool.exe"
                WorkingDirectory="INSTALLFOLDER"
                Icon="AppIcon.ico" />
      <RemoveFolder Id="RemoveShortcutDir" On="uninstall" />
      <RegistryValue Root="HKLM" Key="Software\ProgressiveComputing\PartnerTool"
                     Name="installed" Type="integer" Value="1" KeyPath="yes" />
    </Component>
  </Package>
</Wix>
```
