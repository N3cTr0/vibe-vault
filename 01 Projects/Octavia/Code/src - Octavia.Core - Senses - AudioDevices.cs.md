---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\AudioDevices.cs
---

# src\Octavia.Core\Senses\AudioDevices.cs

```csharp
using NAudio.CoreAudioApi;
using NAudio.Wave;
using Octavia.Core;

namespace Octavia.Senses;

internal sealed record AudioDevice(string Id, string Name, bool IsDefault);

/// Choosing a device by name instead of inheriting the Windows default.
///
/// The default is not always the one you want. This machine boots with "Speakers
/// (Steam Streaming Speakers)" as the default render endpoint — a virtual device that
/// normalises everything to full scale, which leaves a beat detector nothing to find.
/// A microphone default can be just as wrong on a box with several inputs.
///
/// Matching is by substring, case-insensitive, so "Jabra" is enough. It has to be:
/// the legacy WaveIn API **truncates product names to 31 characters**, so the same
/// headset is "Headset Microphone (Jabra EVOLVE 20 MS)" to WASAPI and "Headset
/// Microphone (Jabra EVOLV" to WaveIn. Matching either whole name against the other
/// would fail.
internal static class AudioDevices
{
    /// Capture devices as WASAPI sees them.
    public static IReadOnlyList<AudioDevice> Capture() => Endpoints(DataFlow.Capture);

    /// Render devices, which is what the music loopback taps.
    public static IReadOnlyList<AudioDevice> Render() => Endpoints(DataFlow.Render);

    private static IReadOnlyList<AudioDevice> Endpoints(DataFlow flow)
    {
        var found = new List<AudioDevice>();

        try
        {
            using var enumerator = new MMDeviceEnumerator();
            string? defaultId = null;
            try
            {
                using var standard = enumerator.GetDefaultAudioEndpoint(flow, Role.Multimedia);
                defaultId = standard.ID;
            }
            catch { /* no default at all; every device is then simply non-default */ }

            foreach (var device in enumerator.EnumerateAudioEndPoints(flow, DeviceState.Active))
            {
                using (device) found.Add(new AudioDevice(device.ID, device.FriendlyName, device.ID == defaultId));
            }
        }
        catch (Exception ex)
        {
            Log.Warn($"could not list {flow} devices: {ex.Message}");
        }

        return found;
    }

    /// The WASAPI endpoint whose name contains `wanted`, or the Windows default when
    /// `wanted` is empty. Returns null when a name was asked for and nothing matched,
    /// so the caller can say so rather than silently using something else.
    ///
    /// The caller owns the returned device and must dispose it.
    public static MMDevice? Resolve(DataFlow flow, string? wanted)
    {
        try
        {
            var enumerator = new MMDeviceEnumerator();

            if (string.IsNullOrWhiteSpace(wanted))
            {
                using (enumerator)
                {
                    return enumerator.TryGetDefaultAudioEndpoint(flow, Role.Multimedia, out var standard)
                        ? standard
                        : null;
                }
            }

            using (enumerator)
            {
                foreach (var device in enumerator.EnumerateAudioEndPoints(flow, DeviceState.Active))
                {
                    if (Matches(device.FriendlyName, wanted)) return device;
                    device.Dispose();
                }
            }

            Log.Warn($"no active {flow} device matching '{wanted}'");
            return null;
        }
        catch (Exception ex)
        {
            Log.Warn($"could not resolve a {flow} device: {ex.Message}");
            return null;
        }
    }

    /// The legacy WaveIn index for a wanted name, or -1 to mean "let Windows choose".
    /// WaveIn indexes devices rather than naming them, and its names are truncated, so
    /// this is deliberately forgiving in both directions.
    public static int WaveInIndex(string? wanted)
    {
        if (string.IsNullOrWhiteSpace(wanted)) return -1;

        try
        {
            for (var i = 0; i < WaveIn.DeviceCount; i++)
            {
                if (Matches(WaveIn.GetCapabilities(i).ProductName, wanted)) return i;
            }

            Log.Warn($"no WaveIn device matching '{wanted}'; using the Windows default");
        }
        catch (Exception ex)
        {
            Log.Warn($"could not list WaveIn devices: {ex.Message}");
        }

        return -1;
    }

    /// True when either name contains the other, which survives WaveIn's truncation
    /// whichever side the short one came from.
    private static bool Matches(string name, string wanted)
    {
        var a = name.Trim();
        var b = wanted.Trim();
        if (a.Length == 0 || b.Length == 0) return false;

        return a.Contains(b, StringComparison.OrdinalIgnoreCase)
            || b.Contains(a, StringComparison.OrdinalIgnoreCase);
    }
}
```
