---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Audio\Fft.cs
---

# src\Octavia.Core\Audio\Fft.cs

```csharp
namespace Octavia.Audio;

/// A small in-place radix-2 FFT.
///
/// Written rather than taken from a package because it is thirty lines, it has no
/// native dependency to collide with Whisper's or the renderer's, and Stage 7's beat
/// detection wants exactly the same thing.
internal static class Fft
{
    /// Transforms `real`/`imaginary` in place. Length must be a power of two.
    public static void Transform(Span<double> real, Span<double> imaginary)
    {
        var n = real.Length;
        if (n <= 1) return;

        // Bit-reversal permutation.
        for (int i = 1, j = 0; i < n; i++)
        {
            var bit = n >> 1;
            for (; (j & bit) != 0; bit >>= 1) j ^= bit;
            j ^= bit;

            if (i < j)
            {
                (real[i], real[j]) = (real[j], real[i]);
                (imaginary[i], imaginary[j]) = (imaginary[j], imaginary[i]);
            }
        }

        for (var length = 2; length <= n; length <<= 1)
        {
            var angle = -2 * Math.PI / length;
            var stepReal = Math.Cos(angle);
            var stepImaginary = Math.Sin(angle);

            for (var start = 0; start < n; start += length)
            {
                double wReal = 1, wImaginary = 0;
                for (var k = 0; k < length / 2; k++)
                {
                    var a = start + k;
                    var b = a + length / 2;

                    var tReal = real[b] * wReal - imaginary[b] * wImaginary;
                    var tImaginary = real[b] * wImaginary + imaginary[b] * wReal;

                    real[b] = real[a] - tReal;
                    imaginary[b] = imaginary[a] - tImaginary;
                    real[a] += tReal;
                    imaginary[a] += tImaginary;

                    (wReal, wImaginary) = (wReal * stepReal - wImaginary * stepImaginary,
                                           wReal * stepImaginary + wImaginary * stepReal);
                }
            }
        }
    }

    /// A Hann window, so a frame's edges do not smear energy across the spectrum.
    public static double[] Hann(int length)
    {
        var window = new double[length];
        for (var i = 0; i < length; i++)
            window[i] = 0.5 * (1 - Math.Cos(2 * Math.PI * i / (length - 1)));
        return window;
    }
}
```
