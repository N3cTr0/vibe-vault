---
project: Octavia
tags: [octavia, code]
source-path: tools\EarsTest\ModelProbe.cs
---

# tools\EarsTest\ModelProbe.cs

```csharp
using System.Net.Http;
using System.Diagnostics;
using System.Net.Http.Json;
using System.Text.Json;

/// Which local model she should think with, measured rather than argued about.
///
/// Three numbers matter and they pull against each other:
///
///   **Gate latency** — paid on every utterance the room produces, including the ones
///   she is about to ignore. This is the number that decides whether she feels slow.
///
///   **Reply speed** — paid once per turn, and only when someone is actually waiting.
///   Wall clock, not tokens per second: a chattier model that ignores "be brief" finishes
///   later than a slower one that stops talking.
///
///   **Tool calling** — whether it picks the right tool with the right argument. A model
///   that is delightful company and cannot reliably call `set_light` is no use for Stage
///   12, and this is the axis where small models fall down hardest.
///
/// Run it again when the graphics card changes: every figure here is CPU-only.
internal static class ModelProbe
{
    private const string Endpoint = "http://localhost:11434/v1/";

    private const string GateInstruction = """
        A microphone is always on in someone's home. Decide whether this line is a
        question or a request that an assistant should answer.

        YES: questions, requests and instructions, even with no name in them.
        NO: television, lyrics, one side of a phone call, people talking to each other,
        muttering, fragments.

        Reply YES or NO, then a dash and at most four words. Nothing else.
        """;

    /// Deliberately mixed: two that must be answered, two that must not.
    private static readonly (string Line, bool Answer)[] GateCases =
    [
        ("what is the weather doing tomorrow", true),
        ("can you turn the kitchen light off", true),
        ("and that's a beautiful strike from thirty yards out", false),
        ("so then she said she wasn't coming after all", false)
    ];

    /// The tools from tools\mock-mcp.ps1, in the shape the OpenAI-compatible endpoint
    /// wants. The expected answers are unambiguous on purpose — this measures whether a
    /// model can call a tool at all, not whether it can read minds.
    private static readonly (string Ask, string Expect)[] ToolCases =
    [
        ("turn the kitchen light off", "house_set_light"),
        ("is the hall lamp on?", "house_get_state"),
        ("switch on the lamp in the bedroom", "house_set_light"),
        ("what's the state of the porch light", "house_get_state")
    ];

    public static async Task RunAsync(string[] models)
    {
        using var http = new HttpClient { BaseAddress = new Uri(Endpoint), Timeout = TimeSpan.FromMinutes(5) };

        Console.WriteLine($"{"model",-26} {"gate",8} {"gate ok",8} {"reply",8} {"tools",7}");
        Console.WriteLine(new string('-', 62));

        foreach (var model in models)
        {
            try
            {
                var gate = await GateAsync(http, model);
                var reply = await ReplyAsync(http, model);
                var tools = await ToolsAsync(http, model);

                Console.WriteLine($"{model,-26} {gate.Median + " ms",8} {gate.Correct + "/4",8} " +
                                  $"{reply + " s",8} {tools + "/4",7}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"{model,-26} failed: {ex.Message}");
            }
        }

        Console.WriteLine();
        Console.WriteLine("gate    median warm latency for a yes/no judgement, and how many of four it got right");
        Console.WriteLine("reply   wall clock for a short spoken answer, which is what someone waits through");
        Console.WriteLine("tools   how many of four unambiguous requests produced the right tool call");
    }

    private static async Task<(long Median, int Correct)> GateAsync(HttpClient http, string model)
    {
        var times = new List<long>();
        var correct = 0;

        // One throwaway call first: the first one loads the model, which is not what an
        // utterance pays.
        await AskAsync(http, model, GateInstruction, "warm up", 24);

        foreach (var (line, expected) in GateCases)
        {
            var watch = Stopwatch.StartNew();
            var answer = await AskAsync(http, model, GateInstruction, line, 24);
            watch.Stop();

            times.Add(watch.ElapsedMilliseconds);
            var said = answer.TrimStart().StartsWith("YES", StringComparison.OrdinalIgnoreCase);
            if (said == expected) correct++;
        }

        times.Sort();
        return (times[times.Count / 2], correct);
    }

    private static async Task<double> ReplyAsync(HttpClient http, string model)
    {
        var watch = Stopwatch.StartNew();
        await AskAsync(http, model,
            "You are Octavia. Answer in two short sentences, spoken aloud. Never use lists.",
            "what should I do with a spare hour this evening?", 200);
        watch.Stop();

        return Math.Round(watch.Elapsed.TotalSeconds, 1);
    }

    private static async Task<int> ToolsAsync(HttpClient http, string model)
    {
        var tools = new object[]
        {
            Tool("house_get_state", "Read the current state of a device in the house.",
                 new { entity = new { type = "string", description = "Which device" } }, ["entity"]),
            Tool("house_set_light", "Turn a light on or off.",
                 new { entity = new { type = "string" }, on = new { type = "boolean" } }, ["entity", "on"])
        };

        var right = 0;
        foreach (var (ask, expect) in ToolCases)
        {
            try
            {
                var response = await http.PostAsJsonAsync("chat/completions", new
                {
                    model,
                    messages = new object[]
                    {
                        new { role = "system", content = "You control a house. Use a tool when asked about or to change a device." },
                        new { role = "user", content = ask }
                    },
                    tools,
                    stream = false,
                    temperature = 0
                });

                var body = await response.Content.ReadFromJsonAsync<JsonElement>();
                var message = body.GetProperty("choices")[0].GetProperty("message");

                if (!message.TryGetProperty("tool_calls", out var calls) || calls.GetArrayLength() == 0) continue;

                var called = calls[0].GetProperty("function").GetProperty("name").GetString();
                if (called == expect) right++;
            }
            catch
            {
                // A model that cannot be asked for a tool call scores zero for that case,
                // which is the honest result rather than an exception up the stack.
            }
        }

        return right;
    }

    private static object Tool(string name, string description, object properties, string[] required) => new
    {
        type = "function",
        function = new { name, description, parameters = new { type = "object", properties, required } }
    };

    private static async Task<string> AskAsync(
        HttpClient http, string model, string system, string user, int maxTokens)
    {
        var response = await http.PostAsJsonAsync("chat/completions", new
        {
            model,
            messages = new object[]
            {
                new { role = "system", content = system },
                new { role = "user", content = user }
            },
            stream = false,
            temperature = 0,
            max_tokens = maxTokens
        });

        var body = await response.Content.ReadFromJsonAsync<JsonElement>();
        return body.GetProperty("choices")[0].GetProperty("message")
                   .GetProperty("content").GetString() ?? "";
    }
}
```
