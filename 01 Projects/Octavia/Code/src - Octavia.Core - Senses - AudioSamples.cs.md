---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\AudioSamples.cs
---

# src\Octavia.Core\Senses\AudioSamples.cs

```csharp
using NAudio.Wave;

namespace Octavia.Senses;

/// Turning a WASAPI buffer into floats.
///
/// Shared because getting it wrong is silent and expensive. A shared-mode mix format is
/// almost always `WAVE_FORMAT_EXTENSIBLE`, whose `Encoding` reads `Extensible` and whose
/// real sample type is a sub-format GUID inside it. Code that tests
/// `Encoding == IeeeFloat` therefore concludes "not float" for the commonest format
/// there is, and if it then falls back to `ToInt16` it reads the **low two bytes of a
/// 32-bit sample** — the least significant bits, which behave like uniform noise.
///
/// That produced a captured peak of 1.000, an RMS of 0.577 and a crest factor of 1.73 —
/// uniform noise's own figures to three decimals — on every device tried, and it was
/// blamed in turn on Remote Desktop, on a virtual streaming endpoint and on a headset.
/// It was none of them. See [[Music]].
internal static class AudioSamples
{
    // KSDATAFORMAT_SUBTYPE_IEEE_FLOAT
    private static readonly Guid FloatSubFormat = new("00000003-0000-0010-8000-00aa00389b71");

    public static bool IsFloat(WaveFormat format) =>
        format.Encoding == WaveFormatEncoding.IeeeFloat ||
        (format is WaveFormatExtensible extensible && extensible.SubFormat == FloatSubFormat);

    /// One sample at `at`, whatever width the endpoint chose.
    public static float Read(ReadOnlySpan<byte> buffer, int at, int bytesPerSample, bool isFloat)
    {
        if (isFloat) return bytesPerSample >= 4 ? BitConverter.ToSingle(buffer[at..]) : 0f;

        return bytesPerSample switch
        {
            1 => (buffer[at] - 128) / 128f,
            2 => BitConverter.ToInt16(buffer[at..]) / 32768f,
            // Packed 24-bit, little-endian; the top byte carries the sign.
            3 => (buffer[at] | (buffer[at + 1] << 8) | ((sbyte)buffer[at + 2] << 16)) / 8388608f,
            4 => BitConverter.ToInt32(buffer[at..]) / 2147483648f,
            _ => 0f
        };
    }

    /// The loudest sample in a buffer, for level metering.
    public static float PeakOf(ReadOnlySpan<byte> buffer, int count, WaveFormat format)
    {
        var bytesPerSample = format.BitsPerSample / 8;
        if (bytesPerSample == 0) return 0f;

        var isFloat = IsFloat(format);
        var highest = 0f;

        for (var at = 0; at + bytesPerSample <= count; at += bytesPerSample)
        {
            var magnitude = Math.Abs(Read(buffer, at, bytesPerSample, isFloat));
            if (magnitude > highest) highest = magnitude;
        }

        return highest;
    }
}
```
