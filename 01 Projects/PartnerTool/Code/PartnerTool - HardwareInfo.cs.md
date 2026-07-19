---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\HardwareInfo.cs
---

# PartnerTool\HardwareInfo.cs

```csharp
using Microsoft.Win32;
using System.Management;

namespace PartnerTool;

public record DiskInfo(string Model, string Type, string Health, double SizeGb)
{
    /// <summary>Drive temperature in °C, from SMART reliability counters (null if unavailable).</summary>
    public int?  TempC        { get; init; }
    /// <summary>SSD wear indicator 0–100% (% of rated write life consumed); null for HDDs / unknown.</summary>
    public int?  WearPct      { get; init; }
    /// <summary>Total powered-on hours from SMART (null if unavailable).</summary>
    public long? PowerOnHours { get; init; }
    /// <summary>Uncorrected read+write errors from SMART — a non-zero value is a warning sign.</summary>
    public long? Errors       { get; init; }

    /// <summary>One-line SMART summary for the UI (empty when no SMART data is available).</summary>
    public string SmartText
    {
        get
        {
            var parts = new List<string>();
            if (TempC is { } t)                       parts.Add($"{t}°C");
            if (WearPct is { } w && w > 0)            parts.Add($"{w}% life used");
            if (PowerOnHours is { } h && h > 0)       parts.Add($"{h:N0} h powered on");
            if (Errors is { } e && e > 0)             parts.Add($"{e} SMART errors");
            return string.Join("   ·   ", parts);
        }
    }
    public bool HasSmart => SmartText.Length > 0;
}
public record VolumeInfo(string Letter, string Label, string FileSystem,
                         double UsedGb, double TotalGb, double UsedPct)
{
    public double FreeGb => TotalGb - UsedGb;
}
public record MemoryModule(string Slot, double SizeGb, int SpeedMhz, string Maker)
{
    // Column values for the System Info memory table (Slot / Size / Speed / Maker).
    public string SlotText  => string.IsNullOrWhiteSpace(Slot) ? "—" : Slot;
    public string SizeText  => $"{SizeGb:F0} GB";
    public string SpeedText => SpeedMhz > 0 ? $"{SpeedMhz} MHz" : "—";
    public string MakerText => string.IsNullOrWhiteSpace(Maker) ? "—" : Maker;
}
public record GpuInfo(string Name, double VramGb, string Driver, string DriverDate, string Resolution);

public class HardwareInfo
{
    public List<DiskInfo>     Disks         { get; set; } = new();
    public List<VolumeInfo>   Volumes       { get; set; } = new();
    public List<MemoryModule> Memory        { get; set; } = new();
    public int                SlotsUsed     { get; set; }
    public int                SlotsTotal    { get; set; }
    public double             TotalMemoryGb { get; set; }
    public double             MaxMemoryGb   { get; set; }
    public List<GpuInfo>      Gpus          { get; set; } = new();
    public string             BiosVersion   { get; set; } = "Unknown";
    public string             BiosDate      { get; set; } = "";
    public string             BootMode      { get; set; } = "Unknown";
    public string             Tpm           { get; set; } = "Not present";
    public bool?              TpmHealthy    { get; set; }
    public int?               BatteryWearPct    { get; set; }
    public int?               BatteryDesignMwh  { get; set; }
    public int?               BatteryFullMwh    { get; set; }

    public static HardwareInfo Collect()
    {
        var hw = new HardwareInfo();

        // ── Physical disks (modern Storage namespace gives SSD/HDD + health) ──
        try
        {
            using var q = new ManagementObjectSearcher(@"root\Microsoft\Windows\Storage",
                "SELECT FriendlyName, MediaType, BusType, HealthStatus, Size FROM MSFT_PhysicalDisk");
            foreach (ManagementObject o in q.Get())
            {
                string media = Convert.ToInt32(o["MediaType"] ?? 0) switch
                    { 3 => "HDD", 4 => "SSD", 5 => "SCM", _ => "Disk" };
                string bus = Convert.ToInt32(o["BusType"] ?? 0) switch
                    { 17 => "NVMe ", 11 => "SATA ", 7 => "USB ", 8 => "RAID ", _ => "" };
                string health = Convert.ToInt32(o["HealthStatus"] ?? 99) switch
                    { 0 => "Healthy", 1 => "Warning", 2 => "Unhealthy", _ => "Unknown" };
                double sizeGb = o["Size"] != null ? Convert.ToDouble(o["Size"]) / 1073741824.0 : 0;

                // SMART reliability data (temperature, wear, power-on hours, errors).
                // It lives on the MSFT_StorageReliabilityCounter related to this disk,
                // and needs no third-party tool or kernel driver — just admin (we are).
                int? tempC = null, wear = null; long? poh = null, errs = null;
                try
                {
                    foreach (ManagementObject rc in o.GetRelated("MSFT_StorageReliabilityCounter"))
                    using (rc)
                    {
                        if (rc["Temperature"] is { } t && Convert.ToInt32(t) > 0) tempC = Convert.ToInt32(t);
                        if (rc["Wear"]        is { } w) wear = Convert.ToInt32(w);
                        if (rc["PowerOnHours"] is { } p) poh = Convert.ToInt64(p);
                        long re = rc["ReadErrorsUncorrected"]  is { } r ? Convert.ToInt64(r) : 0;
                        long we = rc["WriteErrorsUncorrected"] is { } e ? Convert.ToInt64(e) : 0;
                        errs = re + we;
                        break;
                    }
                }
                catch { }

                hw.Disks.Add(new DiskInfo(
                    o["FriendlyName"]?.ToString()?.Trim() ?? "Disk", $"{bus}{media}", health, sizeGb)
                {
                    TempC = tempC, WearPct = wear, PowerOnHours = poh, Errors = errs,
                });
            }
        }
        catch { }

        // Fallback if the Storage namespace was unavailable
        if (hw.Disks.Count == 0)
        {
            try
            {
                using var q = new ManagementObjectSearcher("SELECT Model, Size FROM Win32_DiskDrive");
                foreach (ManagementObject o in q.Get())
                {
                    double sizeGb = o["Size"] != null ? Convert.ToDouble(o["Size"]) / 1073741824.0 : 0;
                    hw.Disks.Add(new DiskInfo(o["Model"]?.ToString()?.Trim() ?? "Disk", "Disk", "Unknown", sizeGb));
                }
            }
            catch { }
        }

        // ── Volumes ──────────────────────────────────────────
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT DeviceID, VolumeName, FileSystem, Size, FreeSpace FROM Win32_LogicalDisk WHERE DriveType=3");
            foreach (ManagementObject o in q.Get())
            {
                double total = o["Size"] != null ? Convert.ToDouble(o["Size"]) / 1073741824.0 : 0;
                double free  = o["FreeSpace"] != null ? Convert.ToDouble(o["FreeSpace"]) / 1073741824.0 : 0;
                double used  = total - free;
                hw.Volumes.Add(new VolumeInfo(
                    o["DeviceID"]?.ToString() ?? "?",
                    o["VolumeName"]?.ToString()?.Trim() is { Length: > 0 } v ? v : "Local Disk",
                    o["FileSystem"]?.ToString() ?? "",
                    used, total, total > 0 ? used / total * 100 : 0));
            }
        }
        catch { }

        // ── Memory modules ───────────────────────────────────
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT DeviceLocator, Capacity, Speed, ConfiguredClockSpeed, Manufacturer FROM Win32_PhysicalMemory");
            foreach (ManagementObject o in q.Get())
            {
                double gb  = o["Capacity"] != null ? Convert.ToDouble(o["Capacity"]) / 1073741824.0 : 0;
                int speed  = Convert.ToInt32(o["ConfiguredClockSpeed"] ?? o["Speed"] ?? 0);
                hw.Memory.Add(new MemoryModule(
                    o["DeviceLocator"]?.ToString()?.Trim() ?? "DIMM",
                    gb, speed,
                    o["Manufacturer"]?.ToString()?.Trim() is { Length: > 0 } m ? m : "—"));
                hw.TotalMemoryGb += gb;
            }
            hw.SlotsUsed = hw.Memory.Count;
        }
        catch { }

        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT MemoryDevices, MaxCapacityEx FROM Win32_PhysicalMemoryArray");
            foreach (ManagementObject o in q.Get())
            {
                hw.SlotsTotal  = Convert.ToInt32(o["MemoryDevices"] ?? 0);
                if (o["MaxCapacityEx"] != null)
                    hw.MaxMemoryGb = Convert.ToDouble(o["MaxCapacityEx"]) / 1073741824.0;
                break;
            }
        }
        catch { }

        // ── Graphics ─────────────────────────────────────────
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT Name, AdapterRAM, DriverVersion, DriverDate, CurrentHorizontalResolution, " +
                "CurrentVerticalResolution, CurrentRefreshRate FROM Win32_VideoController");
            foreach (ManagementObject o in q.Get())
            {
                var name = o["Name"]?.ToString()?.Trim();
                if (string.IsNullOrEmpty(name)) continue;
                double vram = o["AdapterRAM"] != null ? Convert.ToDouble(o["AdapterRAM"]) / 1073741824.0 : 0;
                string drvDate = "";
                try { if (o["DriverDate"] != null)
                        drvDate = ManagementDateTimeConverter.ToDateTime(o["DriverDate"].ToString()).ToString(Dates.Date); }
                catch { }
                string res = "";
                if (o["CurrentHorizontalResolution"] != null && o["CurrentVerticalResolution"] != null)
                {
                    res = $"{o["CurrentHorizontalResolution"]}×{o["CurrentVerticalResolution"]}";
                    if (o["CurrentRefreshRate"] != null) res += $" @ {o["CurrentRefreshRate"]} Hz";
                }
                hw.Gpus.Add(new GpuInfo(name, vram,
                    o["DriverVersion"]?.ToString() ?? "—", drvDate, res));
            }
        }
        catch { }

        // ── BIOS ─────────────────────────────────────────────
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT SMBIOSBIOSVersion, ReleaseDate FROM Win32_BIOS");
            foreach (ManagementObject o in q.Get())
            {
                hw.BiosVersion = o["SMBIOSBIOSVersion"]?.ToString()?.Trim() ?? "Unknown";
                try { if (o["ReleaseDate"] != null)
                        hw.BiosDate = ManagementDateTimeConverter.ToDateTime(o["ReleaseDate"].ToString()).ToString(Dates.Date); }
                catch { }
                break;
            }
        }
        catch { }

        // ── Boot mode (UEFI vs Legacy) ───────────────────────
        try
        {
            using var k = Registry.LocalMachine.OpenSubKey(
                @"SYSTEM\CurrentControlSet\Control\SecureBoot\State");
            hw.BootMode = k != null ? "UEFI" : "Legacy BIOS";
        }
        catch { hw.BootMode = "Unknown"; }

        // ── TPM ──────────────────────────────────────────────
        try
        {
            using var q = new ManagementObjectSearcher(@"root\cimv2\Security\MicrosoftTpm",
                "SELECT SpecVersion, IsEnabled_InitialValue, IsActivated_InitialValue FROM Win32_Tpm");
            foreach (ManagementObject o in q.Get())
            {
                var spec = o["SpecVersion"]?.ToString()?.Split(',')[0].Trim() ?? "?";
                bool en  = o["IsEnabled_InitialValue"] is bool b1 && b1;
                bool act = o["IsActivated_InitialValue"] is bool b2 && b2;
                hw.Tpm = $"{spec} ({(en ? "enabled" : "disabled")})";
                hw.TpmHealthy = en && act;
                break;
            }
        }
        catch { }

        // ── Battery wear (mWh design vs full charge) ─────────
        try
        {
            int? design = null, full = null;
            using (var q = new ManagementObjectSearcher(@"root\WMI",
                "SELECT DesignedCapacity FROM BatteryStaticData"))
                foreach (ManagementObject o in q.Get())
                    { design = Convert.ToInt32(o["DesignedCapacity"]); break; }

            using (var q = new ManagementObjectSearcher(@"root\WMI",
                "SELECT FullChargedCapacity FROM BatteryFullChargedCapacity"))
                foreach (ManagementObject o in q.Get())
                    { full = Convert.ToInt32(o["FullChargedCapacity"]); break; }

            if (design is > 0 && full is > 0)
            {
                hw.BatteryDesignMwh = design;
                hw.BatteryFullMwh   = full;
                hw.BatteryWearPct   = (int)Math.Round((1.0 - (double)full.Value / design.Value) * 100);
            }
        }
        catch { }

        return hw;
    }
}
```
