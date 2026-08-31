---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\MicProbe.cs
---

# tools\EarsTest\MicProbe.cs

```csharp
using NAudio.CoreAudioApi;
using NAudio.Wave;

/// Diagnostic: what capture devices exist, which one Windows considers default,
/// and whether any signal is actually arriving. Run this when she cannot hear you.
internal static class MicProbe
{
    public static void Run(int seconds = 5)
    {
        Console.WriteLine("legacy WaveIn devices (what she used to open):");
        if (WaveIn.DeviceCount == 0)
            Console.WriteLine("  none — the legacy API sees no capture device at all");
        for (var i = 0; i < WaveIn.DeviceCount; i++)
            Console.WriteLine($"  [{i}] {WaveIn.GetCapabilities(i).ProductName}");

        using var enumerator = new MMDeviceEnumerator();

        MMDevice? standard = null;
        try { standard = enumerator.GetDefaultAudioEndpoint(DataFlow.Capture, Role.Multimedia); }
        catch (Exception ex) { Console.WriteLine($"  no default capture endpoint: {ex.Message}"); }

        Console.WriteLine();
        Console.WriteLine("WASAPI capture endpoints:");
        var endpoints = enumerator.EnumerateAudioEndPoints(DataFlow.Capture, DeviceState.Active).ToList();
        if (endpoints.Count == 0)
            Console.WriteLine("  none active");

        foreach (var device in endpoints)
        {
            var mark = device.ID == standard?.ID ? " <= Windows default" : string.Empty;
            Console.WriteLine($"  {device.FriendlyName}{mark}");
        }

        if (standard is null) return;

        Console.WriteLine();
        Console.WriteLine($"listening to '{standard.FriendlyName}' for {seconds}s — make some noise:");

        // This used to read AudioMeterInformation.MasterPeakValue, on the reasoning
        // that Windows' own meter shows signal without opening a capture. It does not:
        // that meter is zero unless something already holds the device open, so it
        // reported a working microphone as silent. Peak opens a real capture now.
        var peakSeen = 0f;
        for (var i = 0; i < seconds; i++)
        {
            var peak = Octavia.Diagnostics.SystemReport.PeakAsync(standard, TimeSpan.FromSeconds(1)).GetAwaiter().GetResult();
            if (peak > peakSeen) peakSeen = peak;
            Console.WriteLine($"  {i + 1}s  peak so far {peakSeen:0.000}");
        }

        Console.WriteLine(peakSeen > 0.01f
            ? $"  SIGNAL PRESENT (peak {peakSeen:0.000}) — audio reaches Windows"
            : "  SILENT — nothing arrived on that device while the capture was open");

        foreach (var device in endpoints) device.Dispose();
        standard.Dispose();
    }
}
```
