---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\IpScanner.cs
---

# PartnerTool\IpScanner.cs

```csharp
using System.Net;
using System.Net.NetworkInformation;
using System.Net.Sockets;
using System.Runtime.InteropServices;
using System.Threading;

namespace PartnerTool;

/// <summary>One discovered host. <see cref="How"/> records what proved it was alive.</summary>
public class IpHost
{
    public required string Ip   { get; init; }
    public required uint   Sort { get; init; }   // IP as a host-order uint, so rows sort numerically
    public string Name   { get; set; } = "";
    public string Mac    { get; set; } = "";
    public string How    { get; set; } = "";     // "ping 2 ms" / "ARP" / "TCP 445"
    public string Note   { get; set; } = "";     // "this PC" / "gateway"
    public string Ports  { get; set; } = "";     // deep scan only: open TCP ports, e.g. "445, 3389"

    public string NameText  => Name.Length  > 0 ? Name  : "—";
    public string MacText   => Mac.Length   > 0 ? Mac   : "—";
    public string NoteText  => Note.Length  > 0 ? $"  ({Note})" : "";
    public string PortsText => Ports.Length > 0 ? Ports : "—";
}

/// <summary>
/// Advanced IP Scanner-style sweep of an IPv4 range: ICMP ping, then an ARP probe (which finds
/// hosts that block ping — the firewall default on Windows clients), then reverse DNS on whatever
/// answered. Optionally a TCP knock on common ports, which is the only thing that finds a
/// ping-blocking host on a DIFFERENT subnet (ARP is link-local, so it can't cross a router).
///
/// Read-only: it sends ICMP echo, ARP requests and — only in deep mode — TCP SYNs to ports we
/// immediately close. Nothing is written to any remote host.
/// </summary>
public static class IpScanner
{
    /// <summary>Refuse absurd ranges — a /16 sweep would take hours and look like an attack.</summary>
    public const int MaxHosts = 8192;

    private static readonly int[] DeepPorts = { 445, 3389, 139, 80, 443, 22 };

    // ── Range parsing ─────────────────────────────────────────────────────
    // Accepts "192.168.1.0/24", "192.168.1.1-192.168.1.254", "192.168.1.1-254", "192.168.1.50".

    public static List<uint> ParseRange(string text, out string error)
    {
        error = "";
        var list = new List<uint>();
        text = (text ?? "").Trim();

        try
        {
            if (text.Contains('/'))
            {
                var parts = text.Split('/');
                if (!IPAddress.TryParse(parts[0], out var baseIp) || !int.TryParse(parts[1], out int bits) || bits is < 8 or > 32)
                { error = "Not a valid CIDR range (e.g. 192.168.1.0/24)."; return list; }

                uint ip   = ToUInt(baseIp);
                uint mask = bits == 0 ? 0 : uint.MaxValue << (32 - bits);
                uint net  = ip & mask, bcast = net | ~mask;
                // Skip the network and broadcast addresses on anything wider than a /31.
                uint first = bits >= 31 ? net : net + 1;
                uint last  = bits >= 31 ? bcast : bcast - 1;
                // long counter: at the very top of the space (…/…255) a uint a++ would wrap to 0 and loop.
                for (long a = first; a <= last && list.Count <= MaxHosts; a++) list.Add((uint)a);
            }
            else if (text.Contains('-'))
            {
                var parts = text.Split('-', 2);
                if (!IPAddress.TryParse(parts[0].Trim(), out var from))
                { error = "Not a valid start address."; return list; }

                string endText = parts[1].Trim();
                // "192.168.1.1-254" → reuse the first three octets of the start address.
                if (!endText.Contains('.'))
                    endText = string.Join('.', from.GetAddressBytes().Take(3)) + "." + endText;

                if (!IPAddress.TryParse(endText, out var to))
                { error = "Not a valid end address."; return list; }

                uint a0 = ToUInt(from), b0 = ToUInt(to);
                if (b0 < a0) { error = "The end address is before the start address."; return list; }
                for (long a = a0; a <= b0 && list.Count <= MaxHosts; a++) list.Add((uint)a);
            }
            else
            {
                if (!IPAddress.TryParse(text, out var one)) { error = "Not a valid IP address or range."; return list; }
                list.Add(ToUInt(one));
            }
        }
        catch { error = "Not a valid IP address or range."; return new List<uint>(); }

        if (list.Count == 0) error = "That range contains no usable addresses.";
        else if (list.Count > MaxHosts) { error = $"That range is too large — {MaxHosts:N0} addresses max."; list.Clear(); }
        return list;
    }

    /// <summary>The local /24 to prefill the box with — e.g. "192.168.1.0/24".</summary>
    public static string DefaultRange()
    {
        var (ip, prefix) = LocalIPv4();
        if (ip is null) return "192.168.1.0/24";
        int bits = Math.Max(prefix, 24);               // never suggest a sweep wider than a /24
        uint mask = uint.MaxValue << (32 - bits);
        return $"{ToIp(ToUInt(ip) & mask)}/{bits}";
    }

    private static (IPAddress? ip, int prefix) LocalIPv4()
    {
        try
        {
            (IPAddress ip, int prefix)? fallback = null;   // first up IPv4, used only if nothing has a gateway

            foreach (var ni in NetworkInterface.GetAllNetworkInterfaces())
            {
                if (ni.OperationalStatus != OperationalStatus.Up) continue;
                if (ni.NetworkInterfaceType == NetworkInterfaceType.Loopback) continue;

                var props = ni.GetIPProperties();
                // A real, routable NIC has a default gateway; VMware/Hyper-V host-only adapters don't.
                bool hasGateway = props.GatewayAddresses.Any(g => g.Address.AddressFamily == AddressFamily.InterNetwork &&
                                                                  !g.Address.Equals(IPAddress.Any));
                foreach (var ua in props.UnicastAddresses)
                {
                    if (ua.Address.AddressFamily != AddressFamily.InterNetwork || IPAddress.IsLoopback(ua.Address)) continue;
                    if (hasGateway) return (ua.Address, ua.PrefixLength);   // prefer a gateway-bearing NIC
                    fallback ??= (ua.Address, ua.PrefixLength);
                }
            }
            if (fallback is { } f) return (f.ip, f.prefix);
        }
        catch { }
        return (null, 24);
    }

    /// <summary>Our own addresses and default gateways, so found rows can be labelled.</summary>
    private static (HashSet<string> mine, HashSet<string> gateways) LocalMarkers()
    {
        var mine = new HashSet<string>(StringComparer.Ordinal);
        var gws  = new HashSet<string>(StringComparer.Ordinal);
        try
        {
            foreach (var ni in NetworkInterface.GetAllNetworkInterfaces())
            {
                if (ni.OperationalStatus != OperationalStatus.Up) continue;
                var props = ni.GetIPProperties();
                foreach (var ua in props.UnicastAddresses)
                    if (ua.Address.AddressFamily == AddressFamily.InterNetwork) mine.Add(ua.Address.ToString());
                foreach (var gw in props.GatewayAddresses)
                    if (gw.Address.AddressFamily == AddressFamily.InterNetwork) gws.Add(gw.Address.ToString());
            }
        }
        catch { }
        return (mine, gws);
    }

    // ── The sweep ─────────────────────────────────────────────────────────

    /// <summary>
    /// Scan <paramref name="addresses"/>. <paramref name="found"/> fires (on a thread-pool thread)
    /// as each live host is confirmed; <paramref name="progress"/> reports addresses completed.
    /// </summary>
    public static async Task<int> ScanAsync(
        IReadOnlyList<uint> addresses, bool deep,
        Action<IpHost> found, Action<int, int> progress, CancellationToken ct)
    {
        var (mine, gateways) = LocalMarkers();
        int total = addresses.Count, done = 0, alive = 0;

        // ARP is link-local: it only resolves hosts on one of OUR subnets. Off-subnet addresses
        // skip it (SendARP would just block for its full timeout and fail).
        var localNets = LocalNetworks();

        using var gate = new SemaphoreSlim(64);
        var tasks = addresses.Select(async addr =>
        {
            await gate.WaitAsync(ct).ConfigureAwait(false);
            try
            {
                ct.ThrowIfCancellationRequested();
                var host = await ProbeAsync(addr, deep, OnLink(addr, localNets), ct).ConfigureAwait(false);
                if (host is not null)
                {
                    if (mine.Contains(host.Ip)) host.Note = "this PC";
                    else if (gateways.Contains(host.Ip)) host.Note = "gateway";
                    Interlocked.Increment(ref alive);
                    found(host);
                }
            }
            catch (OperationCanceledException) { }
            catch { }
            finally
            {
                gate.Release();
                progress(Interlocked.Increment(ref done), total);
            }
        });

        try { await Task.WhenAll(tasks).ConfigureAwait(false); }
        catch (OperationCanceledException) { }
        return alive;
    }

    private static async Task<IpHost?> ProbeAsync(uint addr, bool deep, bool onLink, CancellationToken ct)
    {
        string ip = ToIp(addr);

        // 1 + 2. ICMP echo and the ARP probe run CONCURRENTLY. A dead on-link host would otherwise
        // cost the ping timeout PLUS the ARP timeout back-to-back; in parallel we pay only the slower
        // of the two. ARP proves a host exists on our own segment even when it ignores ping (the
        // Windows-client default) and hands us the MAC for free. Neither task throws — both swallow.
        var pingTask = PingAsync(ip);
        var arpTask  = onLink ? Task.Run(() => ArpLookup(addr), ct) : Task.FromResult("");
        await Task.WhenAll(pingTask, arpTask).ConfigureAwait(false);

        string how = pingTask.Result;                 // "ping N ms" or ""
        string mac = arpTask.Result;                  // "AA:BB:…" or ""
        if (how.Length == 0 && mac.Length > 0) how = "ARP";

        // 3. TCP knock. In deep mode we knock ALL the common ports (concurrently) on EVERY host, so
        // the results list each host's open ports, not just the first hit. This also finds a
        // ping-blocking host across a router (ARP can't cross one), which is why an open port alone
        // counts a host as alive.
        var openPorts = deep ? await ScanPortsAsync(ip, ct).ConfigureAwait(false) : new List<int>();
        if (how.Length == 0 && openPorts.Count > 0) how = $"TCP {openPorts[0]}";

        if (how.Length == 0) return null;

        var host = new IpHost
        {
            Ip = ip, Sort = addr, Mac = mac, How = how,
            Ports = string.Join(", ", openPorts),
        };
        host.Name = await ReverseDnsAsync(ip).ConfigureAwait(false);
        return host;
    }

    /// <summary>ICMP echo → "ping N ms" if it answers, "" otherwise. Never throws.</summary>
    private static async Task<string> PingAsync(string ip)
    {
        try
        {
            using var ping = new Ping();
            var reply = await ping.SendPingAsync(ip, 700).ConfigureAwait(false);
            if (reply.Status == IPStatus.Success) return $"ping {reply.RoundtripTime} ms";
        }
        catch { }
        return "";
    }

    /// <summary>Knock every <see cref="DeepPorts"/> entry concurrently; return the open ones, ascending.</summary>
    private static async Task<List<int>> ScanPortsAsync(string ip, CancellationToken ct)
    {
        var results = await Task.WhenAll(DeepPorts.Select(async port =>
        {
            try
            {
                using var client = new TcpClient();
                // Cancel the connect on timeout instead of racing Task.Delay and abandoning it: an
                // abandoned connect faults once the TcpClient is disposed, and with nobody awaiting
                // it the unobserved-exception handler logs it. Six ports × 254 hosts made that a
                // megabytes-per-scan flood in PartnerTool_errors.log.
                using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
                cts.CancelAfter(400);
                await client.ConnectAsync(ip, port, cts.Token).ConfigureAwait(false);
                return port;
            }
            catch { }
            return 0;
        })).ConfigureAwait(false);
        return results.Where(p => p > 0).OrderBy(p => p).ToList();
    }

    private static async Task<string> ReverseDnsAsync(string ip)
    {
        try
        {
            // Same rule as the port probe: cancel, don't abandon. An orphaned reverse lookup faults
            // with "No such host is known" for every IP with no PTR record — which is most of them.
            using var cts = new CancellationTokenSource(900);
            var entry = await Dns.GetHostEntryAsync(ip, cts.Token).ConfigureAwait(false);
            return entry.HostName;
        }
        catch { }
        return "";
    }

    // ── ARP ───────────────────────────────────────────────────────────────

    [DllImport("iphlpapi.dll", ExactSpelling = true)]
    private static extern int SendARP(uint destIp, uint srcIp, byte[] macAddr, ref uint macAddrLen);

    /// <summary>MAC of an on-link host via an ARP request, or "" if it doesn't answer.</summary>
    private static string ArpLookup(uint addr)
    {
        try
        {
            var mac = new byte[6];
            uint len = 6;
            // SendARP wants the address in network byte order, which is what ToNetwork produces.
            if (SendARP(ToNetwork(addr), 0, mac, ref len) != 0 || len < 6) return "";
            if (mac.All(b => b == 0)) return "";
            return string.Join(":", mac.Take(6).Select(b => b.ToString("X2")));
        }
        catch { return ""; }
    }

    /// <summary>(network, mask) for every IPv4 subnet this machine is directly attached to.</summary>
    private static List<(uint net, uint mask)> LocalNetworks()
    {
        var nets = new List<(uint, uint)>();
        try
        {
            foreach (var ni in NetworkInterface.GetAllNetworkInterfaces())
            {
                if (ni.OperationalStatus != OperationalStatus.Up) continue;
                if (ni.NetworkInterfaceType == NetworkInterfaceType.Loopback) continue;
                foreach (var ua in ni.GetIPProperties().UnicastAddresses)
                {
                    if (ua.Address.AddressFamily != AddressFamily.InterNetwork) continue;
                    int bits = ua.PrefixLength;
                    if (bits is < 8 or > 32) continue;
                    uint mask = bits == 0 ? 0 : uint.MaxValue << (32 - bits);
                    nets.Add((ToUInt(ua.Address) & mask, mask));
                }
            }
        }
        catch { }
        return nets;
    }

    private static bool OnLink(uint addr, List<(uint net, uint mask)> nets)
        => nets.Any(n => (addr & n.mask) == n.net);

    // ── Conversions ───────────────────────────────────────────────────────
    // Internally an address is a HOST-order uint so it increments and sorts naturally.

    private static uint ToUInt(IPAddress ip)
    {
        var b = ip.GetAddressBytes();
        return ((uint)b[0] << 24) | ((uint)b[1] << 16) | ((uint)b[2] << 8) | b[3];
    }

    private static string ToIp(uint a) => $"{a >> 24}.{(a >> 16) & 0xFF}.{(a >> 8) & 0xFF}.{a & 0xFF}";

    private static uint ToNetwork(uint a)
        => (a >> 24) | ((a >> 8) & 0xFF00) | ((a << 8) & 0xFF0000) | (a << 24);
}
```
