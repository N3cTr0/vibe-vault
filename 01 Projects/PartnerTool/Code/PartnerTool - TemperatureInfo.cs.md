---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\TemperatureInfo.cs
---

# PartnerTool\TemperatureInfo.cs

```csharp
using LibreHardwareMonitor.Hardware;

namespace PartnerTool;

/// <summary>Per-drive health from the SSD/NVMe SMART sensors LibreHardwareMonitor exposes
/// (these cover NVMe, which the Windows MSStorageDriver SMART path does not).</summary>
public record DriveHealth(string Name, double? LifePct, double? WrittenTb, double? TempC)
{
    public string HealthText
    {
        get
        {
            var parts = new List<string>();
            if (LifePct   is { } l) parts.Add($"Life {l:F0}%");
            if (WrittenTb is { } w) parts.Add($"{w:F1} TB written");
            if (TempC     is { } t) parts.Add($"{t:F0}°C");
            return string.Join("  ·  ", parts);
        }
    }
}

/// <summary>
/// Hardware telemetry via LibreHardwareMonitorLib: peak CPU / GPU temperature (used in the exported
/// report), GPU VRAM, NVMe remaining-life / total-bytes-written and CPU package power — all gathered
/// from a single driver open. This is the one collector that needs a third-party library and a
/// kernel sensor driver (loaded on demand — the app is already elevated). If the driver can't load
/// it returns empty; nothing else depends on it. (The temperatures &amp; fans list view was removed.)
/// </summary>
public class TemperatureInfo
{
    private const StringComparison OIC = StringComparison.OrdinalIgnoreCase;

    public double? CpuTemp { get; private set; }
    public double? GpuTemp { get; private set; }

    // Extras unlocked from the same single LHM read.
    public double? GpuVramUsedGb    { get; private set; }
    public double? GpuVramTotalGb   { get; private set; }
    public double? CpuPackagePowerW { get; private set; }
    public List<DriveHealth> Drives { get; } = new();

    public static TemperatureInfo Collect()
    {
        var info = new TemperatureInfo();
        // Opt-out safety valve: the sensor driver is the one component that can fault at the
        // native level (an AccessViolation that managed try/catch can't trap). If a machine
        // misbehaves, turning sensors off in Settings skips it entirely.
        if (!SettingsStore.Current.EnableSensors) return info;

        Computer? computer = null;
        try
        {
            // Only enable the subsystems Walk() actually reads (CPU temp/power, GPU temp/VRAM, storage
            // SMART). Motherboard + controller probing is unused here AND it makes the library query
            // Microsoft_IPMI / superIO on every collect — which fails with "invalid class" on client
            // hardware (no BMC) and floods the WMI-Activity/Operational log. Leaving them off also
            // trims sensor-init time and the native driver's surface (fewer chips poked). Confirmed via
            // the WMI-Activity log: those failed IPMI queries came from our own PID.
            computer = new Computer
            {
                IsCpuEnabled     = true,
                IsGpuEnabled     = true,
                IsStorageEnabled = true,
            };
            computer.Open();
            computer.Accept(new UpdateVisitor());

            foreach (var hw in computer.Hardware)
                Walk(hw, info);
        }
        catch { }
        finally { try { computer?.Close(); } catch { } }
        return info;
    }

    private static void Walk(IHardware hw, TemperatureInfo info)
    {
        hw.Update();
        bool isCpu = hw.HardwareType == HardwareType.Cpu;
        bool isGpu = hw.HardwareType is HardwareType.GpuNvidia or HardwareType.GpuAmd or HardwareType.GpuIntel;
        bool isStorage = hw.HardwareType == HardwareType.Storage;

        foreach (var s in hw.Sensors)
        {
            if (s.Value is not float raw || float.IsNaN(raw)) continue;
            double v = Math.Round(raw, 1);

            if (s.SensorType == SensorType.Temperature)
            {
                if (isCpu && (info.CpuTemp is null || v > info.CpuTemp)) info.CpuTemp = v;
                if (isGpu && (info.GpuTemp is null || v > info.GpuTemp)) info.GpuTemp = v;
            }
        }

        try
        {
            if (isGpu)
            {
                // VRAM is reported in MB (SmallData). Prefer dedicated "GPU Memory", fall back to D3D.
                double? used  = Find(hw, SensorType.SmallData, n => n.Contains("GPU Memory Used", OIC))
                             ?? Find(hw, SensorType.SmallData, n => n.Contains("D3D Dedicated Memory Used", OIC));
                double? total = Find(hw, SensorType.SmallData, n => n.Contains("GPU Memory Total", OIC));
                if (used  is { } u && info.GpuVramUsedGb  is null) info.GpuVramUsedGb  = Math.Round(u / 1024.0, 1);
                if (total is { } t && t > 0 && info.GpuVramTotalGb is null) info.GpuVramTotalGb = Math.Round(t / 1024.0, 1);
            }

            if (isCpu)
                info.CpuPackagePowerW ??= Find(hw, SensorType.Power, n => n.Contains("Package", OIC));

            if (isStorage)
            {
                double? life = Find(hw, SensorType.Level, n => n.Contains("Remaining Life", OIC));
                if (life is null && Find(hw, SensorType.Level, n => n.Contains("Percentage Used", OIC)) is { } used)
                    life = Math.Round(100 - used, 0);
                double? writtenGb = Find(hw, SensorType.Data, n => n.Contains("Written", OIC));
                double? temp      = Find(hw, SensorType.Temperature, _ => true);
                if (life != null || writtenGb != null || temp != null)
                    info.Drives.Add(new DriveHealth(hw.Name, life,
                        writtenGb is { } w ? Math.Round(w / 1024.0, 2) : null, temp));
            }
        }
        catch { /* hardware-specific sensor quirks — skip the extras for this node */ }

        foreach (var sub in hw.SubHardware)
            Walk(sub, info);
    }

    private static double? Find(IHardware hw, SensorType type, Func<string, bool> nameMatch)
    {
        foreach (var s in hw.Sensors)
            if (s.SensorType == type && s.Value is float v && !float.IsNaN(v) && nameMatch(s.Name))
                return Math.Round(v, 1);
        return null;
    }

    /// <summary>Tells LibreHardwareMonitor to refresh every hardware node before we read it.</summary>
    private sealed class UpdateVisitor : IVisitor
    {
        public void VisitComputer(IComputer computer) => computer.Traverse(this);
        public void VisitHardware(IHardware hardware)
        {
            hardware.Update();
            foreach (var sub in hardware.SubHardware) sub.Accept(this);
        }
        public void VisitSensor(ISensor sensor) { }
        public void VisitParameter(IParameter parameter) { }
    }
}
```
