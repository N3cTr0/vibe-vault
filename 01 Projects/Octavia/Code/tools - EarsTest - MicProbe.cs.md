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
        /* The endpoint's own level, which is the commonest reason a working microphone
           delivers almost nothing. "Not muted" and "turned up" are different questions,
           and Windows keeps a per-device capture level that nothing in the app can see. */
        try
        {
            var volume = standard.AudioEndpointVolume;
            Console.WriteLine($"level: {volume.MasterVolumeLevelScalar:P0}" +
                              $"{(volume.Mute ? "  MUTED" : "")}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"level: could not read ({ex.Message})");
        }

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

        /* Three verdicts, matching the self-test. One threshold is what made this probe
           call a working headset dead: a quiet room and a dead device both came back
           SILENT, and only one of them is a fault. Any noise floor at all proves the
           path — it is digital zero that means something is wrong. */
        Console.WriteLine(peakSeen switch
        {
            > 0.02f => $"  SIGNAL PRESENT (peak {peakSeen:0.000}) — she can hear you",
            > 0.0005f => $"  WORKING, room noise only (peak {peakSeen:0.000}) — talk to see it rise",
            _ => "  DIGITALLY SILENT — not even a noise floor, which a live microphone always has"
        });

        foreach (var device in endpoints) device.Dispose();
        standard.Dispose();
    }
}
```
