---
project: Octavia
tags: [octavia, code]
source-path: tools\VoiceAudition\Program.cs
---

# tools\VoiceAudition\Program.cs

```csharp
using System.Diagnostics;
using SherpaOnnx;
using VoiceAudition;

// Stage 16. The roadmap says the deliverable of this stage is "a shortlist he can listen
// to - the same sentence in each candidate - rather than a recommendation argued from
// datasheets". This is that: it renders one paragraph in every candidate voice into
// `data\auditions`, named so the folder reads as a menu.
//
//   dotnet run --project tools/VoiceAudition            everything
//   dotnet run --project tools/VoiceAudition -- piper   the incumbent engine only
//   dotnet run --project tools/VoiceAudition -- kokoro  the challenger only

/* Chosen to expose the things that make a voice sound like a person rather than a reader:
   contractions, a dash where the thought turns, a subordinate clause, and a question that
   has to rise at the end. A voice can say "the quick brown fox" convincingly and still be
   unbearable over a whole evening. */
const string Script =
    "I'm Octavia. I've been listening while you were out - the house stayed quiet, and " +
    "nobody came to the door. Do you want the whole of it now, or should that wait until " +
    "you've had your coffee?";

var what = args.Length > 0 ? args[0].ToLowerInvariant() : "all";

Directory.CreateDirectory(Paths.Auditions);
Console.WriteLine($"script: {Script}");
Console.WriteLine($"into:   {Paths.Auditions}");
Console.WriteLine();

var rendered = new List<string>();

if (what is "all" or "piper") rendered.AddRange(await Piper.RenderAsync(Script));
if (what is "all" or "kokoro") rendered.AddRange(await Kokoro.RenderAsync(Script));

Console.WriteLine();
Console.WriteLine($"{rendered.Count} rendered. Play the folder in order and pick one:");
Console.WriteLine($"  explorer \"{Paths.Auditions}\"");

// So the folder is self-describing a week from now, when the WAV names alone will not say
// which was the incumbent or what any of it cost.
await File.WriteAllTextAsync(Path.Combine(Paths.Auditions, "what these are.txt"),
    $"""
     Stage 16 - a voice she actually wants to hear.

     Every file below is this same paragraph:

       {Script}

     Piper is what she speaks with today; en_US-amy-medium is the incumbent and the one to
     judge the rest against. Piper is free, offline, and already integrated.

     Kokoro is the challenger: 82M parameters, also free, also offline, also streamable, and
     a much larger model than any Piper voice. It costs a 400 MB download and a new engine
     in the voice folder. Nothing else about her changes - it produces raw PCM at 24 kHz,
     so her mouth still reads off the waveform exactly as it does now.

     Pick by ear. That is the only criterion at this stage.
     """);
```
