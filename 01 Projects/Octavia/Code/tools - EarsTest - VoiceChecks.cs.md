---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\VoiceChecks.cs
---

# tools\EarsTest\VoiceChecks.cs

```csharp
using Octavia.Voice;

/// The contract between her and the voice engine, which is two processes and two pipes.
///
/// **This is the seam most likely to rot silently.** `octavia-kokoro` writes an
/// end-of-utterance marker on stderr and `KokoroVoice` parses it; they are separate
/// projects, built separately, and nothing but this file says the two agree. A marker
/// that stopped being recognised would not throw, would not log, and would not stop her
/// speaking — the caption would simply go back to sitting still, which is exactly what it
/// did before the marker existed and is therefore invisible.
///
/// Free, and in the suite: no engine is started and no audio is made.
internal static class VoiceChecks
{
    public static int Run()
    {
        var failures = 0;
        void Check(string label, bool ok, string detail = "")
        {
            Console.WriteLine($"  {(ok ? "ok  " : "FAIL")}   {label}{(detail.Length > 0 ? $": {detail}" : "")}");
            if (!ok) failures++;
        }

        // Written the way the engine writes it — `Console.Error.WriteLine($"{Control}end
        // {produced}")` — rather than as a constant shared between the two, because a shared
        // constant would keep agreeing with itself after one side changed.
        const char control = (char)1;

        Check("the marker the engine writes is read as a boundary",
              KokoroVoice.Boundary($"{control}end 48000") == 48000);

        Check("...at zero, which is what an abandoned utterance produces",
              KokoroVoice.Boundary($"{control}end 0") == 0);

        Check("...and past what an int would hold, since it counts from launch",
              KokoroVoice.Boundary($"{control}end 3000000000") == 3000000000L);

        /* The other direction, and the one that matters more: her voice engine narrates to
           the same pipe, and a log line misread as a boundary would advance the caption to a
           sentence she is nowhere near. */
        foreach (var line in new[]
                 {
                     "kokoro ready: 24000 Hz, 53 speakers, speaker 3",
                     "could not say it: something went wrong",
                     "end 48000",              // the marker without its control character
                     $"{control}end",          // no count
                     $"{control}end ",         // empty count
                     $"{control}ended 48000",  // a different control word
                     $"{control}hush",
                     ""
                 })
            Check($"'{line.Replace($"{control}", "\\x01")}' is not a boundary",
                  KokoroVoice.Boundary(line) is null);

        return failures;
    }
}
```
