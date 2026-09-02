---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\BrainChecks.cs
---

# tools\EarsTest\BrainChecks.cs

```csharp
using SixLabors.ImageSharp;
using System.Text;
using Octavia.Brain;

internal static class BrainChecks
{
    public static int Run()
    {
        var failures = 0;

        void Check(string name, string actual, string expected)
        {
            var ok = actual == expected;
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")} {name}: '{actual}'");
            if (!ok)
            {
                Console.WriteLine($"       expected: '{expected}'");
                failures++;
            }
        }

        Console.WriteLine("think filter (streamed in chunks):");

        // A reasoning model's scratchpad must never reach the voice, even when the
        // tags are split across chunk boundaries.
        Check("whole tag in one chunk",
            Feed(["<think>hmm let me see</think>Hello there."]),
            "Hello there.");

        Check("tag split mid-token",
            Feed(["<thi", "nk>secret", " reasoning</thi", "nk>Out loud."]),
            "Out loud.");

        Check("text before and after",
            Feed(["Sure. <think>why</think>Here it is."]),
            "Sure. Here it is.");

        Check("no think tags at all",
            Feed(["Just ", "a plain ", "reply."]),
            "Just a plain reply.");

        Check("unclosed think block is dropped",
            Feed(["Fine.<think>never closed"]),
            "Fine.");

        Check("lone angle bracket survives",
            Feed(["5 < 6 is true."]),
            "5 < 6 is true.");

        // Regression: a reply shorter than the opening tag must not be swallowed.
        Check("very short reply is not held",
            Feed(["Hi."]),
            "Hi.");

        Check("text emitted without waiting for more",
            new Speech.ThinkFilter().Filter("Hello there."),
            "Hello there.");

        Check("only a real tag prefix is held back",
            new Speech.ThinkFilter().Filter("All done.<"),
            "All done.");

        Console.WriteLine("markdown stripping:");

        Check("bold and italics",
            Speech.Speakable("That is **very** _important_ indeed."),
            "That is very important indeed.");

        Check("heading and bullet",
            Speech.Speakable("## Notes\n- first point"),
            "Notes\nfirst point");

        Check("inline code",
            Speech.Speakable("Run `dotnet build` now."),
            "Run dotnet build now.");

        Check("link text kept, url dropped",
            Speech.Speakable("See [the docs](https://example.com) for more."),
            "See the docs for more.");

        Console.WriteLine("sentence splitting:");

        var pending = new StringBuilder("One. Two! Three? And a half");
        var sentences = Speech.DrainSentences(pending).ToList();
        Check("complete sentences", string.Join("|", sentences), "One.|Two!|Three?");
        Check("partial tail held", pending.ToString().Trim(), "And a half");

        var decimals = new StringBuilder("It costs 3.50 today.");
        Check("decimal point is not a sentence end",
            string.Join("|", Speech.DrainSentences(decimals)),
            "It costs 3.50 today.");

        /* The camera still's arithmetic, after the decoder changed underneath it.

           `Sight.Inspect` used WPF's `BitmapFrame`, which was the last thing keeping the core
           on a Windows target. It decodes with ImageSharp now, and these assert that the
           *numbers* did not move: mean and standard deviation of luminance on 0–1, which is
           what decides whether she says the room is dark or the lens is covered.

           Built here rather than loaded from a file, so the expected values are arithmetic
           rather than a property of whatever photograph happened to be checked in. */
        void Ok(string name, bool passed, string detail = "")
        {
            Console.WriteLine($"  {(passed ? "ok  " : "FAIL")} {name}{(detail.Length > 0 ? $": {detail}" : "")}");
            if (!passed) failures++;
        }

        static string Seen(Glance? g) =>
            g is { } v ? $"{v.Width}x{v.Height}, {v.Brightness:0.00} / {v.Spread:0.00}" : "would not decode";

        var black = Sight.Inspect(Jpeg(64, 32, 0));
        Ok("a black frame reads as black",
           black is { Brightness: < 0.02, Spread: < 0.02 }, Seen(black));

        var white = Sight.Inspect(Jpeg(64, 32, 255));
        Ok("a white frame reads as white",
           white is { Brightness: > 0.98, Spread: < 0.02 }, Seen(white));

        var mid = Sight.Inspect(Jpeg(96, 48, 128));
        Ok("a flat grey frame is mid-brightness and featureless",
           mid is { Brightness: > 0.45 and < 0.55, Spread: < 0.02 }, Seen(mid));

        Ok("the dimensions come back as they went in",
           mid is { Width: 96, Height: 48 }, Seen(mid));

        // Half black and half white: the mean sits in the middle and the spread is wide,
        // which is the pair of numbers that tells a covered lens from a dim room.
        var split = Sight.Inspect(Halves(64, 32));
        Ok("a high-contrast frame reads as detailed",
           split is { Brightness: > 0.4 and < 0.6, Spread: > 0.4 }, Seen(split));

        Ok("something that is not an image is null rather than a throw",
           Sight.Inspect(Convert.ToBase64String("not a picture"u8.ToArray())) is null);

        return failures;
    }

    /// A flat single-colour JPEG, base64'd exactly as a face would send one.
    private static string Jpeg(int width, int height, byte level)
    {
        using var image = new SixLabors.ImageSharp.Image<SixLabors.ImageSharp.PixelFormats.Rgba32>(
            width, height, new SixLabors.ImageSharp.PixelFormats.Rgba32(level, level, level));

        return Encode(image);
    }

    /// Black on the left, white on the right.
    private static string Halves(int width, int height)
    {
        using var image = new SixLabors.ImageSharp.Image<SixLabors.ImageSharp.PixelFormats.Rgba32>(width, height);

        image.ProcessPixelRows(rows =>
        {
            for (var y = 0; y < rows.Height; y++)
            {
                var row = rows.GetRowSpan(y);
                for (var x = 0; x < row.Length; x++)
                    row[x] = x < row.Length / 2
                        ? new SixLabors.ImageSharp.PixelFormats.Rgba32(0, 0, 0)
                        : new SixLabors.ImageSharp.PixelFormats.Rgba32(255, 255, 255);
            }
        });

        return Encode(image);
    }

    private static string Encode(SixLabors.ImageSharp.Image image)
    {
        using var stream = new MemoryStream();

        // Quality high enough that JPEG's own ringing does not show up as "detail" in a
        // frame that has none — the flat cases assert a standard deviation near zero.
        image.SaveAsJpeg(stream, new SixLabors.ImageSharp.Formats.Jpeg.JpegEncoder { Quality = 100 });
        return Convert.ToBase64String(stream.ToArray());
    }

    private static string Feed(string[] chunks)
    {
        var filter = new Speech.ThinkFilter();
        var output = new StringBuilder();
        foreach (var chunk in chunks) output.Append(filter.Filter(chunk));
        output.Append(filter.Flush());
        return output.ToString();
    }
}
```
