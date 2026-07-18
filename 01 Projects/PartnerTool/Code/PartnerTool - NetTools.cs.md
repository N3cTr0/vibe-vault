---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\NetTools.cs
---

# PartnerTool\NetTools.cs

```csharp
using System.Net.Http;
using System.Net.Sockets;
using System.Text.RegularExpressions;
using System.Diagnostics;

namespace PartnerTool;

public record WifiNetwork(string Ssid, string Signal, string Radio, string Channel, string Auth);

/// <summary>
/// On-demand network diagnostic helpers: traceroute, DNS lookup, TCP port check, nearby
/// Wi-Fi scan and a quick download speed test. All best-effort and self-contained.
/// </summary>
public static class NetTools
{
    /// <summary>
    /// ONE HttpClient for the whole app. A new client per call is the documented .NET anti-pattern —
    /// each disposed client leaves its socket in TIME_WAIT, so repeated use can exhaust ports.
    /// Timeout is a per-CLIENT property, so per-request limits come from a CancellationToken instead;
    /// this client's own timeout is just a generous backstop.
    /// </summary>
    private static readonly HttpClient Http = new() { Timeout = TimeSpan.FromSeconds(60) };

    /// <summary>Traceroute with live output — one callback per line as each hop answers.</summary>
    public static Task<int> TracerouteLiveAsync(string host, Action<string> onLine, CancellationToken cancel = default)
        => ProcessRunner.RunAsync("tracert.exe", $"-d -h 20 -w 1000 {Sanitize(host)}", null, onLine, cancel: cancel);

    /// <summary>This machine's public IP, as seen from the internet.</summary>
    public static async Task<string> PublicIpAsync()
    {
        using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(8));
        return (await Http.GetStringAsync("https://api.ipify.org", cts.Token)).Trim();
    }

    public static Task<string> DnsLookupAsync(string host, string type)
        => ProcessRunner.RunCaptureAsync("nslookup.exe", $"-type={Sanitize(type)} {Sanitize(host)}", 15000);

    /// <summary>
    /// Test one TCP port. Distinguishes the three real outcomes: OPEN (connected), CLOSED (the host
    /// actively refused — RST, so something's there but nothing's listening) and FILTERED (no reply
    /// within the timeout — a firewall silently dropped it). The old version awaited WhenAny and only
    /// checked <c>client.Connected</c>, so a refusal and a timeout looked identical.
    /// </summary>
    public static async Task<string> PortCheckAsync(string host, int port)
    {
        var (state, detail) = await ProbePortAsync(host, port, 4000);
        return $"● {host}:{port} is {state}{(detail.Length > 0 ? $"  ({detail})" : "")}";
    }

    /// <summary>Check several ports on one host (concurrently) and return one line each.</summary>
    public static async Task<string> PortCheckManyAsync(string host, IReadOnlyList<int> ports)
    {
        var results = await Task.WhenAll(ports.Select(async p =>
        {
            var (state, detail) = await ProbePortAsync(host, p, 3000);
            return $"● {host}:{p,-5} {state}{(detail.Length > 0 ? $"  ({detail})" : "")}";
        }));
        var open = ports.Count == 0 ? 0 : results.Count(r => r.Contains(" OPEN"));
        return string.Join("\n", results) + $"\n\n{open} of {ports.Count} port(s) open.";
    }

    private static async Task<(string state, string detail)> ProbePortAsync(string host, int port, int timeoutMs)
    {
        try
        {
            using var client = new TcpClient();
            // Time out by CANCELLING the connect, not by racing it against Task.Delay and walking
            // away. An abandoned connect still faults later (SocketException 995 once the TcpClient
            // is disposed); with nobody awaiting it that lands in the unobserved-exception handler
            // and floods PartnerTool_errors.log — a scan of a /24 wrote megabytes of them.
            using var cts = new CancellationTokenSource(timeoutMs);
            await client.ConnectAsync(host, port, cts.Token);
            return ("OPEN", "");
        }
        catch (OperationCanceledException)
        {
            return ("FILTERED", $"no response in {timeoutMs / 1000}s — firewall likely dropping it");
        }
        catch (SocketException ex) when (ex.SocketErrorCode == SocketError.ConnectionRefused)
        {
            return ("CLOSED", "actively refused");
        }
        catch (SocketException ex) { return ("CLOSED", ex.SocketErrorCode.ToString()); }
        catch (Exception ex) { return ("error", ex.Message); }
    }

    // "80,443,3389" / "1000-1010" / "443" → a de-duplicated, ordered port list (capped for safety).
    public static List<int> ParsePorts(string text, out string error)
    {
        error = "";
        var set = new SortedSet<int>();
        foreach (var tokenRaw in (text ?? "").Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries))
        {
            if (tokenRaw.Contains('-'))
            {
                var ends = tokenRaw.Split('-', 2, StringSplitOptions.TrimEntries);
                if (!ValidPort(ends[0], out int a) || !ValidPort(ends[1], out int b) || b < a)
                { error = $"Invalid port range “{tokenRaw}”."; return new(); }
                for (int p = a; p <= b && set.Count <= 256; p++) set.Add(p);
            }
            else if (ValidPort(tokenRaw, out int p)) set.Add(p);
            else { error = $"Invalid port “{tokenRaw}”."; return new(); }
        }
        if (set.Count == 0) error = "Enter one or more ports (e.g. 80,443 or 1000-1010).";
        else if (set.Count > 256) { error = "Too many ports — 256 max."; return new(); }
        return set.ToList();
    }

    private static bool ValidPort(string s, out int port) => int.TryParse(s, out port) && port is >= 1 and <= 65535;

    public static async Task<List<WifiNetwork>> WifiScanAsync()
    {
        var list = new List<WifiNetwork>();
        try
        {
            var text = await ProcessRunner.RunCaptureAsync("netsh.exe", "wlan show networks mode=bssid", 15000);
            // Each SSID block: "SSID 1 : Name" then Authentication, then "Signal", "Radio type", "Channel".
            var blocks = Regex.Split(text, @"\r?\nSSID \d+ : ");
            foreach (var block in blocks.Skip(1))
            {
                var ssid = block.Split('\n')[0].Trim();
                string F(string key) => Regex.Match(block, $@"{Regex.Escape(key)}\s*:\s*(.+)", RegexOptions.IgnoreCase)
                    is { Success: true } m ? m.Groups[1].Value.Trim() : "";
                list.Add(new WifiNetwork(ssid.Length > 0 ? ssid : "(hidden)",
                    F("Signal"), F("Radio type"), F("Channel"), F("Authentication")));
            }
        }
        catch { }
        return list;
    }

    /// <summary>Rough download speed (Mbps) by pulling a sized payload from Cloudflare.</summary>
    public static async Task<string> SpeedTestAsync()
    {
        try
        {
            const long bytes = 25_000_000;
            using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
            var sw = Stopwatch.StartNew();
            var data = await Http.GetByteArrayAsync($"https://speed.cloudflare.com/__down?bytes={bytes}", cts.Token);
            sw.Stop();
            double mbps = data.LongLength * 8 / 1_000_000.0 / sw.Elapsed.TotalSeconds;
            return $"● Download: {mbps:F1} Mbps  ({data.LongLength / 1048576.0:F0} MB in {sw.Elapsed.TotalSeconds:F1}s)";
        }
        catch (Exception ex) { return $"● Speed test failed: {ex.Message}"; }
    }

    // Strip anything that could break out of the argument into another command/switch.
    private static string Sanitize(string s) => Regex.Replace(s ?? "", @"[^A-Za-z0-9\.\-_:]", "");
}
```
