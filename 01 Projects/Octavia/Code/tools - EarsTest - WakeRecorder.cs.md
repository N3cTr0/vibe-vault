---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\WakeRecorder.cs
---

# tools\EarsTest\WakeRecorder.cs

```csharp
using NAudio.CoreAudioApi;
using NAudio.Wave;
using Octavia.Core;
using Octavia.Senses;

/// Records the owner saying the wake phrase, for training the wake word on his actual voice.
///
/// **She has one user, and every training clip so far has been of somebody else.** The piper
/// generator makes synthetic speech from a 904-speaker model, which is a reasonable proxy for
/// *anyone's* voice and a poor one for his: his ordinary voice scored between 0.00 and 0.08 on
/// eleven attempts out of twelve, against a model that fires happily on three synthetic voices.
///
/// Real recordings go straight into `positive_train` beside the generated ones, and the
/// training pipeline augments them the same way — reverb from 270 impulse responses, background
/// from AudioSet and music, random gain. So fifty clips do not become fifty training examples;
/// they become fifty spread across the acoustic conditions of a room.
///
/// **The microphone here is deliberately opened by a dev tool and not by her.** Devices belong
/// to clients, not the server, and this is EarsTest — the same bargain `MicProbe` makes.
internal static class WakeRecorder
{
    /// What the trainer expects, and what `WakeWord` reads: 16 kHz mono 16-bit.
    private const int Rate = 16000;

    public static async Task<int> RunAsync(string phrase, int count, string outputDir, float floor = 0.01f)
    {
        Directory.CreateDirectory(outputDir);

        using var enumerator = new MMDeviceEnumerator();
        MMDevice device;
        try
        {
            device = enumerator.GetDefaultAudioEndpoint(DataFlow.Capture, Role.Multimedia);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"no capture device: {ex.Message}");
            return 1;
        }

        Console.WriteLine($"microphone : {device.FriendlyName}");
        Console.WriteLine($"saving to  : {outputDir}");
        Console.WriteLine();
        Console.WriteLine($"Say \"{phrase}\" {count} times, once per prompt.");
        Console.WriteLine();
        Console.WriteLine("**Vary it deliberately.** Normal, quieter, faster, slower, from further");
        Console.WriteLine("away, turned away from the microphone, mid-sentence. A model trained on");
        Console.WriteLine("fifty identical readings learns that one reading, which is the mistake the");
        Console.WriteLine("synthetic clips already made.");
        Console.WriteLine();
        Console.Write("Press Enter when ready...");
        Console.ReadLine();

        var existing = Directory.GetFiles(outputDir, "*.wav").Length;
        var kept = 0;
        var quiet = 0;

        for (var i = 0; i < count; i++)
        {
            /* A second between clips, not a quarter of one. Fifty repetitions back to back
               with no gap produces fifty identical hurried readings, which is the failure this
               whole exercise is trying to undo — the synthetic clips were already too alike. */
            Console.Write($"  [{i + 1}/{count}]  ready...");
            await Task.Delay(900);
            Console.Write("\r  [{0}/{1}]  SPEAK NOW      ".Replace("{0}", (i + 1).ToString())
                                                          .Replace("{1}", count.ToString()));

            var (samples, peak) = await CaptureAsync(device, TimeSpan.FromSeconds(2));

            /* A clip with nothing in it is worse than one fewer clip: it teaches the model
               that silence is the phrase. Rejected rather than counted, and said out loud so
               the run does not quietly end up half empty. */
            if (peak < floor)
            {
                quiet++;
                Console.WriteLine($"too quiet (peak {peak:0.000}) — not saved, say it again");
                i--;
                if (quiet > count) { Console.WriteLine("giving up: nothing is reaching the microphone"); break; }
                continue;
            }

            var path = Path.Combine(outputDir, $"owner_{existing + kept:D4}.wav");
            WriteWav(path, samples);
            kept++;
            Console.WriteLine($"ok (peak {peak:0.000})");
        }

        Console.WriteLine();
        Console.WriteLine($"{kept} clips saved, {Directory.GetFiles(outputDir, "*.wav").Length} in the folder.");
        device.Dispose();
        return kept == 0 ? 1 : 0;
    }

    /// One clip, resampled to 16 kHz mono as it arrives.
    private static async Task<(short[] Samples, float Peak)> CaptureAsync(MMDevice device, TimeSpan window)
    {
        var collected = new List<short>();
        var highest = 0f;

        var recorder = await new WasapiRecorderBuilder().WithDevice(device).BuildAsync();
        var format = recorder.WaveFormat;
        var bytesPerSample = format.BitsPerSample / 8;
        var isFloat = AudioSamples.IsFloat(format);
        var channels = format.Channels;

        // Nearest-neighbour, which is enough here: these clips are augmented with reverb and
        // noise afterwards, so resampler quality is well below the noise floor of the task.
        var step = format.SampleRate / (double)Rate;
        var position = 0.0;
        var frameIndex = 0;

        // buffer.Length is the byte count. The third parameter is a cumulative device
        // position, not a length -- measured, after using it as one truncated every clip to
        // about 55 ms of its 2 seconds and the WAVs looked plausible at a glance.
        recorder.DataAvailable += (buffer, flags, devicePosition, timestamp) =>
        {
            if ((flags & AudioClientBufferFlags.Silent) != 0) return;

            var frames = buffer.Length / (bytesPerSample * channels);
            for (var f = 0; f < frames; f++, frameIndex++)
            {
                // Average the channels rather than take the first: on a stereo headset the
                // left channel is sometimes the dead one.
                var sum = 0f;
                for (var c = 0; c < channels; c++)
                    sum += AudioSamples.Read(buffer, (f * channels + c) * bytesPerSample,
                                             bytesPerSample, isFloat);
                var mono = sum / channels;

                if (Math.Abs(mono) > highest) highest = Math.Abs(mono);

                while (position <= frameIndex)
                {
                    collected.Add((short)Math.Clamp(mono * 32767f, short.MinValue, short.MaxValue));
                    position += step;
                }
            }
        };

        recorder.StartRecording();
        await Task.Delay(window);
        recorder.StopRecording();
        recorder.Dispose();

        return (collected.ToArray(), highest);
    }

    private static void WriteWav(string path, short[] samples)
    {
        using var stream = new FileStream(path, FileMode.Create);
        using var writer = new BinaryWriter(stream);

        var dataBytes = samples.Length * 2;
        writer.Write("RIFF"u8.ToArray());
        writer.Write(36 + dataBytes);
        writer.Write("WAVE"u8.ToArray());
        writer.Write("fmt "u8.ToArray());
        writer.Write(16);                       // PCM header length
        writer.Write((short)1);                 // PCM
        writer.Write((short)1);                 // mono
        writer.Write(Rate);
        writer.Write(Rate * 2);                 // byte rate
        writer.Write((short)2);                 // block align
        writer.Write((short)16);                // bits
        writer.Write("data"u8.ToArray());
        writer.Write(dataBytes);
        foreach (var sample in samples) writer.Write(sample);
    }
}
```
