---
project: PartnerTool
tags: [partnertool, code]
source-path: reference\HUS_MultiVendorUpdate_v1.14.1.ps1
---

# reference\HUS_MultiVendorUpdate_v1.14.1.ps1

```powershell
#===========================================================================
# Purpose:     Detects device manufacturer and automatically installs all
#              available updates for Dell, Lenovo, and HP endpoints using
#              their respective vendor update tools, with auto-installation
#              of any missing prerequisites. HP limited to commercial models
#              only (EliteBook, ProBook, ZBook, HP Pro/Elite desktops).
# Author:      Graeme Lowe
# Version:     v1.14.1
# Date:        05-13-2026
# Requires:    Windows 11+, Run as SYSTEM via NinjaOne
# Log output:  C:\PCI\Logs\HUS_<Device>_<Manufacturer>_MMddyyyy_HHmm.log
#
# NOTE (PartnerTool reference copy): this is the production NinjaOne hardware
# update script. PartnerTool's in-app "Scan Vendor Updates" feature
# (VendorUpdatesInfo.cs) is a SCAN-ONLY adaptation of the logic below — same
# vendor detection, tool auto-install, dcu-cli / LSUClient / HPIA invocation
# and HP commercial-model gating, but it lists available updates instead of
# installing them. Keep the two in sync when either changes.
#
# Changelog:
#   v1.0.0  (05-06-2026) - Initial release. Batch file for Dell Command Update.
#   v1.1.0  (05-06-2026) - Converted to PowerShell.
#   v1.2.0  (05-06-2026) - Added NinjaOne compatibility. Removed interactive prompts, added exit codes.
#   v1.3.0  (05-07-2026) - Added Lenovo support via Lenovo System Update. Added auto-install via Winget if vendor tool is missing.
#   v1.4.0  (05-07-2026) - Added Winget detection and auto-installation if not present on the endpoint.
#   v1.5.0  (05-07-2026) - Added Windows 11 or newer requirement. Added Windows Server block.
#   v1.6.0  (05-07-2026) - Fixed DCU error 107. Removed colour output. Moved all paths to C:\PCI. Added HP support via HP Image Assistant.
#   v1.7.0  (05-08-2026) - Full script review. Consolidated OS checks. Cleaned up comments. Streamlined vendor branches.
#   v1.7.1  (05-08-2026) - Changed log date format from yyyyMMdd to MMddyyyy.
#   v1.8.0  (05-08-2026) - Added Start-Transcript for full output capture. Fixed HPIA download URL to resolve dynamically.
#   v1.9.0  (05-08-2026) - Added HP commercial model detection. Consumer models exit gracefully with code 0.
#   v1.10.0 (05-08-2026) - Consolidated to single log file. Log name includes device and manufacturer. DCU SSM noise filtered.
#   v1.11.0 (05-09-2026) - Ruleset alignment. Added Assert-WindowsWorkstation11. Replaced Write-Host with Write-Output via Write-Log.
#                           Removed Start-Transcript, superseded by Write-Log with Add-Content. Added try/catch/finally with
#                           Clear-TempFolder. Winget check now only runs when vendor tool is missing. Removed backtick-n newlines.
#                           HPIA moved to C:\PCI\Tools for persistent storage. Vendor tool paths updated to use environment variables.
#                           Log filename prefixed with HUS (Hardware Update Script) to avoid overlap with other scripts.
#   v1.12.0 (05-12-2026) - Fixed Lenovo branch argument passing — flag-value pairs were combined into single strings causing tvsu
#                           to fall back to its own scheduled task defaults instead of using our arguments. Replaced stdout redirect
#                           with reading from Lenovo's own log directory (C:\ProgramData\Lenovo\SystemUpdate\Logs\). Updated exit
#                           code messaging to be neutral since exit code 1 is ambiguous across tvsu versions.
#   v1.13.0 (05-13-2026) - Fixed Lenovo argument passing at the root cause. tvsu.exe acts as a launcher that spawns TvsuKernel.exe
#                           as a Windows scheduled task with its own hardcoded arguments, completely ignoring what we pass.
#                           Now calls TvsuKernel.exe directly, bypassing tvsu.exe, so our arguments are actually applied.
#   v1.14.0 (05-13-2026) - Replaced Lenovo System Update / TvsuKernel approach entirely. Both tvsu.exe and TvsuKernel.exe proved
#                           unreliable as SYSTEM. Switched to LSUClient, a PowerShell module designed specifically for silent
#                           unattended Lenovo updates in RMM/SYSTEM contexts. LSUClient requires no external Lenovo software and
#                           installs from PowerShell Gallery. Only unattended packages are installed to prevent GUI dialogs or
#                           unexpected reboots when running headless.
#   v1.14.1 (05-13-2026) - Fixed LSUClient result handling. REBOOT_REQUIRED was not handled in the switch causing staged updates
#                           (BIOS, firmware) to log as skipped despite installing correctly. Added ToString() to enum comparison
#                           and explicit REBOOT_REQUIRED case.
#===========================================================================

function Assert-WindowsWorkstation11 {
    if ([System.Environment]::OSVersion.Platform -ne [System.PlatformID]::Win32NT) {
        Write-Output "ERROR: This script only runs on Windows. Detected platform: $([System.Environment]::OSVersion.Platform). Exiting."
        Exit 1
    }
    $ProductType = (Get-CimInstance -ClassName Win32_OperatingSystem).ProductType
    if ($ProductType -ne 1) {
        Write-Output "ERROR: This script is for Windows workstations only. Detected ProductType: $ProductType. Exiting."
        Exit 1
    }
    $Build = [System.Environment]::OSVersion.Version.Build
    if ($Build -lt 22000) {
        Write-Output "ERROR: This script requires Windows 11 or later. Detected build: $Build. Exiting."
        Exit 1
    }
}

Assert-WindowsWorkstation11

# Manufacturer is detected before the logging bootstrap so the log filename can include it
try {
    $cs           = Get-CimInstance -ClassName Win32_ComputerSystem
    $manufacturer = $cs.Manufacturer
    $model        = $cs.Model
} catch {
    $manufacturer = "Unknown"
    $model        = "Unknown"
}

$isDell   = $manufacturer -match "Dell"
$isLenovo = $manufacturer -match "Lenovo"
$isHP     = $manufacturer -match "HP|Hewlett"
$mfrShort = if ($isDell) { "Dell" } elseif ($isLenovo) { "Lenovo" } elseif ($isHP) { "HP" } else { "Unknown" }

# Logging bootstrap
$LogFolder   = "C:\PCI\Logs"
$TempFolder  = "C:\PCI\Temp"
$ToolsFolder = "C:\PCI\Tools"
$Timestamp   = Get-Date -Format "MMddyyyy_HHmm"
$LogFile     = Join-Path $LogFolder "HUS_$($env:COMPUTERNAME)_${mfrShort}_$Timestamp.log"

foreach ($Folder in @("C:\PCI", $LogFolder, $TempFolder, $ToolsFolder)) {
    if (-not (Test-Path $Folder)) {
        New-Item -ItemType Directory -Path $Folder -Force | Out-Null
    }
}

function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    $Entry = "[$(Get-Date -Format 'MM-dd-yyyy HH:mm:ss')] [$Level] $Message"
    Add-Content -Path $LogFile -Value $Entry
    Write-Output $Entry
}

function Clear-TempFolder {
    Get-ChildItem -Path $TempFolder -Recurse -ErrorAction SilentlyContinue |
        Remove-Item -Recurse -Force -ErrorAction SilentlyContinue
    Write-Log "Temp folder cleared: $TempFolder"
}

Write-Log "Script started on $env:COMPUTERNAME | Manufacturer: $manufacturer | Model: $model"
Write-Log "Log file: $LogFile"

# CONFIGURATION
$updateTypesDell      = "bios,driver,firmware,application,utility"
$hpiaPageUrl          = "https://ftp.hp.com/pub/caps-softpaq/cmit/HPIA.html"
$hpiaExe              = "$ToolsFolder\HPImageAssistant.exe"
$hpCommercialPrefixes = @(
    "EliteBook", "ProBook", "ZBook", "EliteDesk", "ProDesk",
    "ProOne", "EliteOne", "Z[0-9]", "HP Pro", "HP Elite"
)

$exitCode = 0

try {

    if (-not $isDell -and -not $isLenovo -and -not $isHP) {
        throw "Unsupported manufacturer: $manufacturer. This script supports Dell, Lenovo and HP only."
    }

    Write-Log "Manufacturer confirmed: $mfrShort"

    # Winget helper functions — only invoked when a vendor tool needs to be installed
    function Get-WingetPath {
        $candidates = @(
            "winget",
            "$env:LOCALAPPDATA\Microsoft\WindowsApps\winget.exe",
            "$env:ProgramFiles\WindowsApps\Microsoft.DesktopAppInstaller_*_x64__8wekyb3d8bbwe\winget.exe"
        )
        foreach ($path in $candidates) {
            if ($path -match "\*") {
                $resolved = Get-Item $path -ErrorAction SilentlyContinue | Select-Object -Last 1
                if ($resolved) { return $resolved.FullName }
            } else {
                if (Get-Command $path -ErrorAction SilentlyContinue) { return $path }
                if (Test-Path $path) { return $path }
            }
        }
        return $null
    }

    function Install-Winget {
        Write-Log "Winget not found. Attempting to install..." "WARN"

        $wingetTemp = "$TempFolder\WingetInstall"
        New-Item -ItemType Directory -Path $wingetTemp -Force | Out-Null

        try {
            [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

            Write-Log "Fetching latest Winget release from GitHub..."
            $release   = Invoke-RestMethod -Uri "https://api.github.com/repos/microsoft/winget-cli/releases/latest" -Headers @{ "User-Agent" = "PowerShell" }
            $msixAsset = $release.assets | Where-Object { $_.name -match "\.msixbundle$" } | Select-Object -First 1

            if (-not $msixAsset) {
                Write-Log "Could not find Winget .msixbundle in the latest release." "ERROR"
                return $false
            }

            $msixPath   = "$wingetTemp\Microsoft.DesktopAppInstaller.msixbundle"
            $vcLibsPath = "$wingetTemp\VCLibs.appx"
            $xamlPath   = "$wingetTemp\Microsoft.UI.Xaml.appx"

            Write-Log "Downloading Winget..."
            Invoke-WebRequest -Uri $msixAsset.browser_download_url -OutFile $msixPath -UseBasicParsing

            Write-Log "Downloading VCLibs dependency..."
            Invoke-WebRequest -Uri "https://aka.ms/Microsoft.VCLibs.x64.14.00.Desktop.appx" -OutFile $vcLibsPath -UseBasicParsing

            Write-Log "Downloading UI.Xaml dependency..."
            Invoke-WebRequest -Uri "https://github.com/microsoft/microsoft-ui-xaml/releases/download/v2.8.6/Microsoft.UI.Xaml.2.8.x64.appx" -OutFile $xamlPath -UseBasicParsing

            Write-Log "Installing Winget and dependencies..."
            Add-AppxProvisionedPackage -Online `
                -PackagePath $msixPath `
                -DependencyPackagePath @($vcLibsPath, $xamlPath) `
                -SkipLicense | Out-Null

            $env:PATH = [System.Environment]::GetEnvironmentVariable("PATH", "Machine") + ";" +
                        [System.Environment]::GetEnvironmentVariable("PATH", "User")

            Write-Log "Winget installed successfully." "SUCCESS"
            return $true

        } catch {
            Write-Log "Failed to install Winget: $_" "ERROR"
            return $false
        }
    }

    function Install-ViaWinget {
        param([string]$PackageId, [string]$DisplayName)

        $wingetPath = Get-WingetPath

        if (-not $wingetPath) {
            if (-not (Install-Winget)) {
                throw "Winget could not be installed. Cannot auto-install $DisplayName."
            }
            $wingetPath = Get-WingetPath
            if (-not $wingetPath) {
                throw "Winget still not found after installation. A reboot may be required."
            }
        }

        Write-Log "Winget found at: $wingetPath"
        Write-Log "Installing $DisplayName via Winget..."

        try {
            $result = Start-Process -FilePath $wingetPath `
                -ArgumentList "install --id $PackageId --silent --accept-source-agreements --accept-package-agreements" `
                -Wait -PassThru -NoNewWindow

            if ($result.ExitCode -eq 0) {
                Write-Log "$DisplayName installed successfully." "SUCCESS"
                return $true
            }

            Write-Log "Winget exited with code $($result.ExitCode) while installing $DisplayName." "ERROR"
            return $false
        } catch {
            Write-Log "Error installing $DisplayName : $_" "ERROR"
            return $false
        }
    }

    # DELL BRANCH
    if ($isDell) {
        Write-Log "Dell device confirmed. Using Dell Command Update." "SUCCESS"

        $dcuCli = @(
            "$env:ProgramFiles\Dell\CommandUpdate\dcu-cli.exe",
            "${env:ProgramFiles(x86)}\Dell\CommandUpdate\dcu-cli.exe"
        ) | Where-Object { Test-Path $_ } | Select-Object -First 1

        if (-not $dcuCli) {
            Write-Log "Dell Command Update not found. Attempting to install..." "WARN"
            if (-not (Install-ViaWinget -PackageId "Dell.CommandUpdate.Universal" -DisplayName "Dell Command Update")) {
                throw "Dell Command Update could not be installed."
            }

            $dcuCli = @(
                "$env:ProgramFiles\Dell\CommandUpdate\dcu-cli.exe",
                "${env:ProgramFiles(x86)}\Dell\CommandUpdate\dcu-cli.exe"
            ) | Where-Object { Test-Path $_ } | Select-Object -First 1

            if (-not $dcuCli) {
                throw "DCU CLI not found after installation. Please verify the install and retry."
            }
        }

        Write-Log "Found DCU CLI at: $dcuCli" "SUCCESS"

        $dcuTempLog = "$TempFolder\DCU_TempLog.log"

        function Invoke-DCU {
            param([string[]]$Arguments)
            return (Start-Process -FilePath $dcuCli -ArgumentList $Arguments -Wait -PassThru -NoNewWindow).ExitCode
        }

        # Reads the DCU temp log, strips SSM memory noise, outputs each line via Write-Log, then removes the temp file
        function Write-DCULog {
            if (Test-Path $dcuTempLog) {
                Write-Log "--- Dell Command Update Log ---"
                Get-Content $dcuTempLog |
                    Where-Object { $_ -notmatch "Temporary closing synchronized view|Re-Initializing synchronized view" } |
                    ForEach-Object { Write-Log $_ }
                Write-Log "--- End Dell Command Update Log ---"
                Remove-Item $dcuTempLog -Force -ErrorAction SilentlyContinue
            }
        }

        $dcuExitCodes = @{
            0   = "Success"; 1   = "Reboot required"; 2   = "Fatal error"
            3   = "Invalid arguments"; 4   = "Reboot initiated by DCU"
            107 = "Invalid log path"; 500 = "No updates available"; 501 = "Updates not applicable"
        }

        function Get-DCUExitMessage { param([int]$code)
            if ($dcuExitCodes.ContainsKey($code)) { return $dcuExitCodes[$code] }
            return "Unknown exit code"
        }

        Write-Log "Scanning for available Dell updates..."
        $scanCode = Invoke-DCU @("/scan", "-outputLog=$dcuTempLog")
        Write-DCULog
        Write-Log "Scan exit code: $scanCode ($(Get-DCUExitMessage $scanCode))"

        if ($scanCode -eq 107) {
            throw "DCU rejected the log path. Verify C:\PCI\Temp exists and is accessible."
        }

        if ($scanCode -eq 500) {
            Write-Log "No updates available. System is up to date." "SUCCESS"
        } else {
            if ($scanCode -notin @(0, 1)) {
                Write-Log "Unexpected scan exit code ($scanCode). Proceeding anyway..." "WARN"
            }

            Write-Log "Installing Dell updates: $updateTypesDell"

            $applyCode = Invoke-DCU @("/applyUpdates", "-updateType=$updateTypesDell", "-reboot=disable", "-outputLog=$dcuTempLog")
            Write-DCULog
            Write-Log "Install exit code: $applyCode ($(Get-DCUExitMessage $applyCode))"

            switch ($applyCode) {
                0       { Write-Log "All Dell updates installed successfully." "SUCCESS" }
                1       { Write-Log "Dell updates installed. Reboot will occur at scheduled time." "SUCCESS" }
                107     { throw "DCU rejected the log path during install." }
                500     { Write-Log "No applicable updates found." }
                default { throw "Dell updates may not have installed correctly. Exit code: $applyCode" }
            }
        }
    }

    # LENOVO BRANCH
    if ($isLenovo) {
        Write-Log "Lenovo device confirmed. Using LSUClient PowerShell module." "SUCCESS"

        # Ensure NuGet provider is available — required for Install-Module
        [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
        if (-not (Get-PackageProvider -Name NuGet -ErrorAction SilentlyContinue | Where-Object { $_.Version -ge '2.8.5.201' })) {
            Write-Log "Installing NuGet package provider..."
            Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Scope AllUsers | Out-Null
            Write-Log "NuGet provider installed." "SUCCESS"
        }

        # Install LSUClient from PowerShell Gallery if not already present
        if (-not (Get-Module -Name LSUClient -ListAvailable)) {
            Write-Log "LSUClient module not found. Installing from PowerShell Gallery..." "WARN"
            try {
                Install-Module -Name LSUClient -Force -Scope AllUsers -AllowClobber
                Write-Log "LSUClient installed successfully." "SUCCESS"
            } catch {
                throw "Failed to install LSUClient module: $_"
            }
        } else {
            Write-Log "LSUClient module found." "SUCCESS"
        }

        Import-Module LSUClient -Force

        # LSUClient downloads packages to a temp folder — redirect to our standard path
        $lenovoPackagePath = "$TempFolder\Lenovo"
        if (-not (Test-Path $lenovoPackagePath)) {
            New-Item -ItemType Directory -Path $lenovoPackagePath -Force | Out-Null
        }

        Write-Log "Scanning for available Lenovo updates..."

        try {
            # Only install unattended packages — non-unattended packages may launch GUI dialogs
            # or force reboots when running headless as SYSTEM
            $updates = Get-LSUpdate | Where-Object { $_.Installer.Unattended }

            if (-not $updates -or $updates.Count -eq 0) {
                Write-Log "No applicable unattended updates found. System is up to date." "SUCCESS"
            } else {
                Write-Log "$($updates.Count) unattended update(s) found:"
                foreach ($update in $updates) {
                    Write-Log "  [$($update.Type)] $($update.Title)"
                }

                Write-Log "Downloading updates to: $lenovoPackagePath"
                $updates | Save-LSUpdate -Path $lenovoPackagePath

                Write-Log "Installing updates..."
                $results = $updates | Install-LSUpdate

                $rebootRequired = $false
                $failCount      = 0

                foreach ($result in $results) {
                    switch ($result.Result.ToString()) {
                        'SUCCESS' {
                            Write-Log "Installed: $($result.Title)" "SUCCESS"
                            if ($result.RebootRequired) { $rebootRequired = $true }
                        }
                        'REBOOT_REQUIRED' {
                            Write-Log "Staged for reboot: $($result.Title) — will apply at scheduled reboot time." "SUCCESS"
                            $rebootRequired = $true
                        }
                        'FAILED' {
                            Write-Log "Failed: $($result.Title) — $($result.FailureReason)" "ERROR"
                            $failCount++
                        }
                        default {
                            Write-Log "Unexpected result for $($result.Title): $($result.Result)" "WARN"
                        }
                    }
                }

                if ($failCount -gt 0) {
                    throw "$failCount update(s) failed to install. Review log above for details."
                }

                if ($rebootRequired) {
                    Write-Log "Lenovo updates installed. Reboot will occur at scheduled time." "SUCCESS"
                } else {
                    Write-Log "All Lenovo updates installed successfully. No reboot required." "SUCCESS"
                }
            }
        } catch {
            throw "Lenovo update process failed: $_"
        }
    }

    # HP BRANCH
    if ($isHP) {
        Write-Log "HP device detected. Checking model compatibility..."

        $isCommercialHP = ($hpCommercialPrefixes | Where-Object { $model -match $_ }).Count -gt 0

        if (-not $isCommercialHP) {
            Write-Log "Model '$model' is not a supported commercial HP model." "WARN"
            Write-Log "HP Image Assistant only supports EliteBook, ProBook, ZBook, and HP Pro/Elite desktops." "WARN"
            Write-Log "Consumer models (OmniBook, Pavilion, Envy, Spectre, etc.) are not supported by HPIA." "WARN"
            Write-Log "Updates for this device should be managed via Windows Update." "WARN"
        } else {
            Write-Log "Commercial HP model confirmed. Using HP Image Assistant." "SUCCESS"

            if (Test-Path $hpiaExe) {
                Write-Log "HP Image Assistant found at: $hpiaExe"
            } else {
                Write-Log "HP Image Assistant not found. Resolving latest version from HP..." "WARN"

                [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

                # Scrape HP's HPIA page to get the current download URL — avoids hardcoding a version that goes stale
                $hpiaPage        = Invoke-WebRequest -Uri $hpiaPageUrl -UseBasicParsing
                $hpiaDownloadUrl = ($hpiaPage.Links | Where-Object { $_.href -match "hp-hpia-" } | Select-Object -First 1).href

                if (-not $hpiaDownloadUrl) {
                    throw "Could not resolve HPIA download URL from HP's page. Check connectivity to ftp.hp.com."
                }

                $hpiaFileName    = $hpiaDownloadUrl.Split('/')[-1]
                $hpiaVersion     = ($hpiaFileName.Split("-") | Select-Object -Last 1).Replace(".exe", "")
                $hpiaSoftPaqPath = "$TempFolder\$hpiaFileName"

                Write-Log "Latest HPIA version: $hpiaVersion"
                Write-Log "Downloading from: $hpiaDownloadUrl"
                Invoke-WebRequest -Uri $hpiaDownloadUrl -OutFile $hpiaSoftPaqPath -UseBasicParsing
                Write-Log "Download complete." "SUCCESS"

                # The downloaded file is a SoftPaq self-extractor — extract it to get HPImageAssistant.exe
                Write-Log "Extracting HP Image Assistant to: $ToolsFolder"
                Start-Process -FilePath $hpiaSoftPaqPath -ArgumentList "/s /f `"$ToolsFolder`" /e" -Wait -NoNewWindow

                if (-not (Test-Path $hpiaExe)) {
                    throw "Extraction completed but HPImageAssistant.exe was not found in: $ToolsFolder"
                }

                Remove-Item -Path $hpiaSoftPaqPath -Force -ErrorAction SilentlyContinue
                Write-Log "HP Image Assistant ready at: $hpiaExe" "SUCCESS"
            }

            Write-Log "Installing HP updates..."

            # HPIA writes its debug log to the folder specified by /LogFolder — read and pipe into our log then remove it
            $hpiaTempLog = "$TempFolder\HpImageAssistant.log"

            $hpiaCode = (Start-Process -FilePath $hpiaExe `
                -ArgumentList @(
                    "/Operation:Analyze",
                    "/Action:Install",
                    "/Silent",
                    "/Category:All",
                    "/ReportFolder:`"$LogFolder`"",
                    "/SoftPaqFolder:`"$TempFolder`"",
                    "/NoReboot",
                    "/Debug",
                    "/LogFolder:`"$TempFolder`""
                ) `
                -Wait -PassThru -NoNewWindow).ExitCode

            if (Test-Path $hpiaTempLog) {
                Write-Log "--- HP Image Assistant Log ---"
                Get-Content $hpiaTempLog | ForEach-Object { Write-Log $_ }
                Write-Log "--- End HP Image Assistant Log ---"
                Remove-Item $hpiaTempLog -Force -ErrorAction SilentlyContinue
            }

            Write-Log "HPIA exit code: $hpiaCode"

            switch ($hpiaCode) {
                0     { Write-Log "All HP updates installed successfully." "SUCCESS" }
                1     { Write-Log "No updates needed. System is up to date." "SUCCESS" }
                2     { Write-Log "HP updates installed. Reboot will occur at scheduled time." "SUCCESS" }
                3010  { Write-Log "HP updates installed. Reboot will occur at scheduled time." "SUCCESS" }
                16386 {
                    Write-Log "HPIA could not retrieve the reference file for this device (exit code 16386)." "ERROR"
                    Write-Log "Possible causes: model not yet in HP's reference database, or connectivity issue with hpia.hpcloud.hp.com." "ERROR"
                    throw "HPIA failed with exit code 16386."
                }
                256     { throw "HPIA reported a download error (exit code 256). Check connectivity." }
                default { throw "HPIA exited with unexpected code $hpiaCode." }
            }
        }
    }

} catch {
    Write-Log "Script failed: $_" "ERROR"
    $exitCode = 1
} finally {
    Clear-TempFolder
    Write-Log "Script completed. Exit code: $exitCode"
    exit $exitCode
}
```
