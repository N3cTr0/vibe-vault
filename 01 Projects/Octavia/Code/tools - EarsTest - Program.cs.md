---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\Program.cs
---

# tools\EarsTest\Program.cs

```csharp
// Headless proof of the stage-1 pipeline: synthesized speech → Silero VAD → Whisper.
// Run with: dotnet run --project tools/EarsTest [-- model]
using System.Speech.AudioFormat;
using System.Speech.Synthesis;
using Octavia.Senses;

if (args.Length > 0 && args[0] == "mic") { MicProbe.Run(); return; }
if (args.Length > 0 && args[0] == "mouth") { MouthProbe.Run(args.Length > 1 ? args[1] : null); return; }
if (args.Length > 0 && args[0] == "music") { await MusicProbe.RunAsync(args.Length > 1 && args[1] == "demo", args.Length > 2 ? args[2] : null); return; }
if (args.Length > 0 && args[0] == "beats") { Environment.Exit(MusicChecks.Run()); }
if (args.Length > 0 && args[0] == "gate") { await GateProbe.RunAsync(); return; }
if (args.Length > 0 && args[0] == "compute")
{
    await ComputeProbe.RunAsync(args.Length > 1 ? args[1] : "auto", args.Length > 2 ? args[2] : "tiny.en",
        args.Length > 3 && int.TryParse(args[3], out var t) ? t : 0);
    return;
}

var model = args.Length > 0 ? args[0] : "tiny.en";
var phrase = "Hello Octavia, can you hear me clearly today?";
var failures = 0;

// 1. Make a spoken WAV at exactly the format the ears expect.
var wavPath = Path.Combine(Path.GetTempPath(), "octavia-earstest.wav");
using (var synth = new SpeechSynthesizer())
{
    synth.SetOutputToWaveFile(wavPath,
        new SpeechAudioFormatInfo(SileroVad.SampleRate, AudioBitsPerSample.Sixteen, AudioChannel.Mono));
    synth.Speak(phrase);
}

var bytes = File.ReadAllBytes(wavPath);
var samples = new float[(bytes.Length - 44) / 2];
for (var i = 0; i < samples.Length; i++)
    samples[i] = BitConverter.ToInt16(bytes, 44 + i * 2) / 32768f;
Console.WriteLine($"synthesized {samples.Length / (float)SileroVad.SampleRate:0.0}s of speech");

// 2. VAD should call the speech speech and the silence silence.
using (var vad = new SileroVad(WhisperModelStore.SileroPath))
{
    float SpeechFraction(float[] audio)
    {
        vad.Reset();
        int voiced = 0, frames = 0;
        var frame = new float[SileroVad.FrameSamples];
        for (var offset = 0; offset + frame.Length <= audio.Length; offset += frame.Length)
        {
            Array.Copy(audio, offset, frame, 0, frame.Length);
            if (vad.Probability(frame) >= 0.5f) voiced++;
            frames++;
        }
        return frames == 0 ? 0 : voiced / (float)frames;
    }

    var speech = SpeechFraction(samples);
    var silence = SpeechFraction(new float[SileroVad.SampleRate * 3]);
    Console.WriteLine($"VAD: speech {speech:P0} voiced, silence {silence:P0} voiced");

    if (speech < 0.4f) { Console.WriteLine("FAIL: VAD missed the speech"); failures++; }
    if (silence > 0.05f) { Console.WriteLine("FAIL: VAD voiced the silence"); failures++; }
}

// 3. Whisper should get the words.
Console.WriteLine($"ensuring whisper model '{model}'...");
var modelPath = await WhisperModelStore.EnsureAsync(model, m => Console.WriteLine($"  {m}"));

using (var whisper = new WhisperTranscriber(modelPath, "en"))
{
    var started = DateTime.Now;
    var result = await whisper.TranscribeAsync(samples);
    var took = (DateTime.Now - started).TotalSeconds;
    Console.WriteLine($"transcript ({took:0.0}s, confidence {result.Confidence:0.00}): {result.Text}");

    if (!result.Text.Contains("hear me", StringComparison.OrdinalIgnoreCase))
    { Console.WriteLine("FAIL: transcript missed the phrase"); failures++; }

    // 4. Pure silence must produce nothing — the hallucination test.
    var silent = await whisper.TranscribeAsync(new float[SileroVad.SampleRate * 3]);
    Console.WriteLine($"silence transcript: '{silent.Text}' (confidence {silent.Confidence:0.00})");
}

Console.WriteLine();
Console.WriteLine("config and profiles:");
failures += ConfigChecks.Run();

Console.WriteLine();
Console.WriteLine("diagnostics:");
failures += await DiagnosticsChecks.RunAsync();

Console.WriteLine();
Console.WriteLine("brain checks:");
failures += BrainChecks.Run();
Console.WriteLine();
Console.WriteLine("face and expression:");
failures += FaceChecks.Run();

Console.WriteLine();
Console.WriteLine("voice:");
failures += VoiceChecks.Run();

Console.WriteLine();
Console.WriteLine("music:");
failures += MusicChecks.Run();

Console.WriteLine();
Console.WriteLine("attention gate:");
failures += GateChecks.Run();

Console.WriteLine();
Console.WriteLine("face protocol:");
failures += await FaceProtocolChecks.RunAsync();

Console.WriteLine();
Console.WriteLine("tools:");
failures += await ToolChecks.RunAsync();

Console.WriteLine();
Console.WriteLine("local brain probe:");
failures += await LocalBrainProbe.RunAsync();
Console.WriteLine();
Console.WriteLine(failures == 0 ? "ALL CHECKS PASSED" : $"{failures} CHECK(S) FAILED");

// Hard exit: whisper.cpp's native teardown otherwise corrupts the process exit code.
Environment.Exit(failures);
```
