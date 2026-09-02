---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Brain\Situation.cs
---

# src\Octavia.Core\Brain\Situation.cs

```csharp
namespace Octavia.Brain;

/// <param name="Context">What is true in the room, in a sentence. Music today.</param>
/// <param name="Image">One still from the camera as base64 JPEG, when she was asked to look.</param>
///
/// What is happening *now*, as opposed to what was said. It rides with the current
/// question and is never written into the history, so nothing here can still be claimed
/// an hour after it stopped being true.
internal readonly record struct Situation(string? Context = null, string? Image = null)
{
    public bool IsEmpty => Context is null && Image is null;
}

/// What a captured frame actually contains, without keeping it.
///
/// The direct descendant of the silent microphone in Stage 4: a camera can open
/// successfully, report no error, and hand over a black rectangle — and from the outside
/// that is indistinguishable from her simply being wrong about what she saw. A lens cap,
/// a privacy shutter, a redirected device that never starts, and a room with the light
/// off all look like this.
///
/// Reports the picture's shape and how much *variation* is in it. A real photograph of
/// anything at all has spread; a dead frame does not. Nothing is stored — the numbers are
/// computed and the pixels dropped.
internal readonly record struct Glance(int Width, int Height, double Brightness, double Spread)
{
    /// Below this there is no detail in the frame worth sending anywhere.
    public bool LooksBlank => Spread < 0.02;

    public override string ToString() =>
        $"{Width}x{Height}, brightness {Brightness:0.00}, spread {Spread:0.000}" +
        (LooksBlank ? " - BLANK, the camera opened but produced no picture" : "");
}

/// When she needs to open her eyes.
///
/// Deliberately a word list rather than a model call. Opening a camera is the most
/// intrusive thing she does, so the decision to do it must be **legible** — a person
/// should be able to read this and know exactly what makes her look. A model would be
/// more accurate and nobody could audit it.
internal static class Sight
{
    /// Phrases that only make sense if she is meant to be looking. Every one requires
    /// eyes to answer; none of them is a thing you would say to a speaker.
    private static readonly string[] Asks =
    [
        "can you see", "do you see", "what do you see", "look at", "have a look",
        "what am i holding", "what is this", "what's this", "what colour is this",
        "what color is this", "how do i look", "what am i wearing", "read this",
        "what does this say", "am i", "take a look", "check this out", "see this"
    ];

    /// True when answering honestly requires a camera.
    ///
    /// Errs towards *not* looking. A false negative is her saying she cannot see; a
    /// false positive is a camera turning on in someone's home when they did not ask.
    /// Those are not comparable mistakes.
    public static bool WantsEyes(string text)
    {
        var lower = text.ToLowerInvariant();

        // "am i" is in the list because "am I holding this right?" needs eyes, but it is
        // also the opening of "am I boring you" — so it only counts near a looking word.
        foreach (var ask in Asks)
        {
            if (!lower.Contains(ask, StringComparison.Ordinal)) continue;
            if (ask != "am i") return true;

            if (lower.Contains("look", StringComparison.Ordinal)
                || lower.Contains("wearing", StringComparison.Ordinal)
                || lower.Contains("holding", StringComparison.Ordinal))
                return true;
        }

        return false;
    }

    /// Decodes a captured still far enough to describe it, then throws the pixels away.
    /// Null when it will not decode at all, which is its own kind of answer.
    ///
    /// **This one method was the whole of `UseWPF` in the core** — `BitmapFrame` is WIC
    /// underneath, works headless, and is Windows only. It decodes with ImageSharp now,
    /// which is managed and cross-platform, so the core no longer needs WPF. See Stage 15
    /// item 2 for the rest of what a Linux server would still want.
    ///
    /// The arithmetic is unchanged and deliberately so: mean and standard deviation of
    /// luminance, both on 0–1, so a `Glance` written before this reads the same as one
    /// written after. `L8` is the same single-channel grey `Gray8` produced, and ImageSharp
    /// applies the same BT.709 luminance weighting, so the numbers move by rounding at most.
    public static Glance? Inspect(string base64)
    {
        try
        {
            using var stream = new MemoryStream(Convert.FromBase64String(base64));
            using var grey = SixLabors.ImageSharp.Image.Load<SixLabors.ImageSharp.PixelFormats.L8>(stream);

            double sum = 0, sumSquares = 0;
            var counted = 0;

            grey.ProcessPixelRows(rows =>
            {
                for (var y = 0; y < rows.Height; y++)
                {
                    // A row at a time rather than a whole copied buffer: a 4K still is 8 MB
                    // of pixels she looks at once and never keeps.
                    foreach (var pixel in rows.GetRowSpan(y))
                    {
                        var value = pixel.PackedValue / 255.0;
                        sum += value;
                        sumSquares += value * value;
                        counted++;
                    }
                }
            });

            if (counted == 0) return null;

            var mean = sum / counted;
            var variance = Math.Max(0, sumSquares / counted - mean * mean);

            return new Glance(grey.Width, grey.Height, mean, Math.Sqrt(variance));
        }
        catch (Exception ex)
        {
            Core.Log.Warn($"could not read the captured frame: {ex.Message}");
            return null;
        }
    }
}
```
