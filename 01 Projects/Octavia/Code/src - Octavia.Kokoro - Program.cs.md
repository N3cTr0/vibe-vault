---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Kokoro\Program.cs
---

# src\Octavia.Kokoro\Program.cs

```csharp
using System.Collections.Concurrent;
using System.Runtime.InteropServices;
using System.Text;
using SherpaOnnx;

/* Her voice, out of process.

   The contract, which `KokoroVoice` on the other side depends on:

     in    one UTF-8 line per utterance, on stdin. A line starting with U+0001 is a control
           word rather than something to say, and there is one: `hush` abandons what is being
           spoken and everything queued behind it.
     out   raw 16-bit little-endian mono PCM on stdout, at the rate printed on stderr,
           written as it is generated rather than when the utterance is finished.
     log   everything else on stderr, which the host drains into her log.

   Deliberately the same shape Piper had, because that side was already written and proven:
   a long-lived process, sentences in, raw audio out, silence meaning the end. The one thing
   Piper could not do and this can is stop halfway - see the hush below. */

const char Control = (char)1;   // U+0001, the control-word marker

var audio = Console.OpenStandardOutput();

// Her text is prose and prose has accents in it. A redirected pipe otherwise decodes as the
// console's code page, which mangles them silently - she would still say something, just not
// the word that was written.
Console.InputEncoding = new UTF8Encoding(false);

string? Option(string name, string? fallback = null)
{
    var at = Array.IndexOf(args, name);
    return at >= 0 && at + 1 < args.Length ? args[at + 1] : fallback;
}

var modelDir = Option("--model-dir");
if (modelDir is null)
{
    Console.Error.WriteLine("usage: octavia-kokoro --model-dir <dir> [--sid N] [--speed 1.0] [--threads 4]");
    return 2;
}

var config = new OfflineTtsConfig();
config.Model.Kokoro.Model = Path.Combine(modelDir, "model.onnx");
config.Model.Kokoro.Voices = Path.Combine(modelDir, "voices.bin");
config.Model.Kokoro.Tokens = Path.Combine(modelDir, "tokens.txt");
config.Model.Kokoro.DataDir = Path.Combine(modelDir, "espeak-ng-data");

// The multilingual model wants the Chinese segmenter and both lexicons present even when
// every word is English; without them it loads happily and phonemises nothing.
var dict = Path.Combine(modelDir, "dict");
if (Directory.Exists(dict)) config.Model.Kokoro.DictDir = dict;

config.Model.Kokoro.Lexicon = string.Join(',',
    new[] { "lexicon-us-en.txt", "lexicon-zh.txt" }
        .Select(name => Path.Combine(modelDir, name))
        .Where(File.Exists));

config.Model.NumThreads = int.TryParse(Option("--threads"), out var threads) ? threads : 4;
config.Model.Provider = "cpu";
config.Model.Debug = 0;

var speaker = int.TryParse(Option("--sid"), out var sid) ? sid : 0;
var speed = float.TryParse(Option("--speed"), System.Globalization.CultureInfo.InvariantCulture, out var s) ? s : 1.0f;

OfflineTts tts;
try
{
    tts = new OfflineTts(config);
}
catch (Exception ex)
{
    Console.Error.WriteLine($"the voice model would not load: {ex.Message}");
    return 3;
}

Console.Error.WriteLine($"kokoro ready: {tts.SampleRate} Hz, {tts.NumSpeakers} speakers, speaker {speaker}");

/* Reading and speaking are on different threads on purpose.

   One sentence takes a second or two to generate, and if that ran on the thread reading
   stdin then a hush arriving during it could not be *seen* until it was already over -
   which is the whole difference between stopping her and waiting for her to finish. */
var queue = new BlockingCollection<string>(new ConcurrentQueue<string>());
var abandoned = 0;

var speaking = new Thread(() =>
{
    // Held in a local rather than made per utterance: a delegate passed to native code and
    // collected while native code still holds it is a crash, not an exception.
    var pump = new OfflineTtsCallback((samples, count) =>
    {
        if (Volatile.Read(ref abandoned) != 0) return 0;   // 0 tells sherpa to stop generating
        if (count <= 0) return 1;

        var floats = new float[count];
        Marshal.Copy(samples, floats, 0, count);

        var pcm = new byte[count * 2];

        for (var i = 0; i < count; i++)
        {
            // Clamped before scaling: a neural vocoder does overshoot 1.0 occasionally, and
            // a wrapped short is not a quiet artefact, it is a bang at full volume.
            var sample = (int)(Math.Clamp(floats[i], -1f, 1f) * short.MaxValue);
            pcm[i * 2] = (byte)sample;
            pcm[i * 2 + 1] = (byte)(sample >> 8);
        }

        try
        {
            audio.Write(pcm, 0, pcm.Length);
            audio.Flush();
        }
        catch (IOException)
        {
            // The host closed the pipe: she is being shut down mid-sentence, which is
            // ordinary. Stop generating rather than reporting it as a fault.
            Volatile.Write(ref abandoned, 1);
            return 0;
        }

        return 1;
    });

    foreach (var sentence in queue.GetConsumingEnumerable())
    {
        // A hush that arrived while this was queued: drop it. The flag clears only once the
        // queue is empty, so everything behind the hush goes with it rather than just one.
        if (Volatile.Read(ref abandoned) != 0)
        {
            if (queue.Count == 0) Volatile.Write(ref abandoned, 0);
            continue;
        }

        try
        {
            tts.GenerateWithCallback(sentence, speed, speaker, pump);
        }
        catch (Exception ex)
        {
            Console.Error.WriteLine($"could not say it: {ex.Message}");
        }

        if (Volatile.Read(ref abandoned) != 0 && queue.Count == 0) Volatile.Write(ref abandoned, 0);
    }
})
{ IsBackground = true, Name = "kokoro-speaking" };

speaking.Start();

for (var line = Console.In.ReadLine(); line is not null; line = Console.In.ReadLine())
{
    if (line.Length == 0) continue;

    if (line[0] == Control)
    {
        if (line.AsSpan(1).SequenceEqual("hush"))
        {
            Volatile.Write(ref abandoned, 1);
            while (queue.TryTake(out _)) { }
        }

        continue;
    }

    queue.Add(line);
}

// Standard input closed: the host is going away. Whatever is queued is no longer wanted.
Volatile.Write(ref abandoned, 1);
queue.CompleteAdding();
speaking.Join(TimeSpan.FromSeconds(2));
tts.Dispose();
return 0;
```
