---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.Core\Senses\SystemSpeechRecognizer.cs
---

# src\Octavia.Core\Senses\SystemSpeechRecognizer.cs

```csharp
using System.Globalization;
using System.Speech.Recognition;
using Octavia.Core;

namespace Octavia.Senses;

/// The recognizer built into Windows. No download, no cloud, mediocre accuracy.
/// Good enough to prove the pipeline; Whisper replaces this behind the same interface.
internal sealed class SystemSpeechRecognizer : ISpeechRecognizer
{
    private readonly SpeechRecognitionEngine _engine;
    private bool _wantListening;
    private bool _muted;
    private bool _disposed;

    public event Action<string, float>? Recognized;
    public event Action<string>? Hypothesised;

#pragma warning disable CS0067 // the Windows recognizer has no silence watchdog
    public event Action<string>? Trouble;
#pragma warning restore CS0067

    public string EngineName => "Windows Speech Recognition";
    public bool IsListening => _wantListening && !_muted;

    public SystemSpeechRecognizer(string culture)
    {
        var wanted = new CultureInfo(culture);
        var installed = SpeechRecognitionEngine.InstalledRecognizers();

        if (installed.Count == 0)
            throw new InvalidOperationException(
                "Windows has no speech recognition engine installed. " +
                "Add one under Settings > Time & language > Speech.");

        var recognizer = installed.FirstOrDefault(r => r.Culture.Equals(wanted))
                         ?? installed.FirstOrDefault(r => r.Culture.TwoLetterISOLanguageName == wanted.TwoLetterISOLanguageName)
                         ?? installed[0];

        if (!recognizer.Culture.Equals(wanted))
            Log.Write($"no recognizer for {culture}; falling back to {recognizer.Culture.Name}");

        _engine = new SpeechRecognitionEngine(recognizer);
        _engine.LoadGrammar(new DictationGrammar());
        _engine.SetInputToDefaultAudioDevice();
        _engine.SpeechRecognized += OnRecognized;
        _engine.SpeechHypothesized += OnHypothesised;
        _engine.RecognizeCompleted += OnRecognizeCompleted;
    }

    public void Start()
    {
        _wantListening = true;
        if (!_muted) Engage();
    }

    public void Stop()
    {
        _wantListening = false;
        Disengage();
    }

    public void Mute()
    {
        if (_muted) return;
        _muted = true;
        Disengage();
    }

    public void Unmute()
    {
        if (!_muted) return;
        _muted = false;
        if (_wantListening) Engage();
    }

    private void Engage()
    {
        if (_disposed) return;
        try
        {
            _engine.RecognizeAsync(RecognizeMode.Multiple);
        }
        catch (InvalidOperationException)
        {
            // already running
        }
        catch (Exception ex)
        {
            Log.Write($"recognizer start failed: {ex.Message}");
        }
    }

    private void Disengage()
    {
        if (_disposed) return;
        try
        {
            _engine.RecognizeAsyncCancel();
        }
        catch (Exception ex)
        {
            Log.Write($"recognizer stop failed: {ex.Message}");
        }
    }

    private void OnRecognized(object? sender, SpeechRecognizedEventArgs e)
    {
        if (_muted || e.Result is null) return;
        var text = e.Result.Text?.Trim();
        if (!string.IsNullOrEmpty(text)) Recognized?.Invoke(text, e.Result.Confidence);
    }

    private void OnHypothesised(object? sender, SpeechHypothesizedEventArgs e)
    {
        if (_muted) return;
        var text = e.Result?.Text?.Trim();
        if (!string.IsNullOrEmpty(text)) Hypothesised?.Invoke(text);
    }

    private void OnRecognizeCompleted(object? sender, RecognizeCompletedEventArgs e)
    {
        // RecognizeMode.Multiple still completes on error; restart if we still want to hear.
        if (_disposed || _muted || !_wantListening) return;
        if (e.Error is not null) Log.Write($"recognizer error: {e.Error.Message}");
        Engage();
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        try
        {
            _engine.RecognizeAsyncCancel();
            _engine.Dispose();
        }
        catch (Exception ex)
        {
            Log.Write($"recognizer dispose failed: {ex.Message}");
        }
    }
}
```
