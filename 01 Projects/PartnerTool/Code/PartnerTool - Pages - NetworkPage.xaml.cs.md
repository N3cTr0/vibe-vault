---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Pages\NetworkPage.xaml.cs
---

# PartnerTool\Pages\NetworkPage.xaml.cs

```csharp
using System.Net.Http;
using System.Net.NetworkInformation;
using System.Threading;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;

namespace PartnerTool.Pages;

public partial class NetworkPage : UserControl
{
    private bool _netBusy;

    private bool _netInfoLoaded;

    public NetworkPage()
    {
        InitializeComponent();
        TxtScanRange.Text = IpScanner.DefaultRange();   // prefill with this machine's own subnet
        Loaded += (_, _) => LoadAdapters();
        // Wi-Fi + connections use netsh/netstat, so load them once when first shown.
        IsVisibleChanged += async (_, _) =>
        {
            if (IsVisible && !_netInfoLoaded) { _netInfoLoaded = true; await LoadNetInfoAsync(); }
        };
    }

    // ── WI-FI + ACTIVE CONNECTIONS ────────────────────────────────────────
    private async void WifiRefresh_Click(object sender, RoutedEventArgs e) => await LoadWifiAsync();
    private async void ConnRefresh_Click(object sender, RoutedEventArgs e) => await LoadConnectionsAsync();

    private async Task LoadNetInfoAsync() => await Task.WhenAll(LoadWifiAsync(), LoadConnectionsAsync());

    private async Task LoadWifiAsync()
    {
        BtnWifiRefresh.IsEnabled = false;
        try
        {
            bool connected = WifiInfo.IsConnected();
            var profiles   = await WifiInfo.GetProfilesAsync();

            TxtWifiStatus.Text = connected
                ? "● Connected to a wireless network"
                : "● Not connected to a wireless network";
            TxtWifiStatus.Foreground = connected ? StatusColors.Green : StatusColors.Muted;

            IcWifiProfiles.ItemsSource = profiles;
            bool any = profiles.Count > 0;
            IcWifiProfiles.Visibility  = any ? Visibility.Visible : Visibility.Collapsed;
            TxtNoWifi.Visibility       = any ? Visibility.Collapsed : Visibility.Visible;
        }
        catch { TxtWifiStatus.Text = "Wi-Fi information unavailable."; }
        finally { BtnWifiRefresh.IsEnabled = true; }
    }

    private async Task LoadConnectionsAsync()
    {
        BtnConnRefresh.IsEnabled = false;
        try { IcConnections.ItemsSource = await ConnectionsInfo.CollectAsync(); }
        finally { BtnConnRefresh.IsEnabled = true; }
    }

    private async void ShowWifiPassword_Click(object sender, RoutedEventArgs e)
    {
        if (sender is not Button { Tag: string profile }) return;
        if (!MessageWindow.Confirm("Wi-Fi Password", $"Reveal the saved password for “{profile}”?",
                "The network password will be shown in clear text. Make sure no one is looking over your shoulder.",
                MessageKind.Warning, Window.GetWindow(this)))
            return;

        ActivityLog.Action("Network", $"Reveal saved Wi-Fi password for “{profile}”");
        var pwd = await WifiInfo.GetPasswordAsync(profile);
        MessageWindow.Show("Wi-Fi Password", profile, $"Password:\n\n{pwd}", MessageKind.Info, Window.GetWindow(this));
    }

    // ── DIAGNOSTIC TOOLS ──────────────────────────────────────────────────

    // Traceroute streams hop-by-hop. It can take a minute or two on a path with dead hops, and the
    // old wait-for-everything version looked frozen the whole time — techs assumed it was stuck.
    private CancellationTokenSource? _traceCts;

    private async void Traceroute_Click(object sender, RoutedEventArgs e)
    {
        if (_netBusy) return;
        string host = TxtNetHost.Text.Trim();
        _netBusy = true;
        PnlNetDiag.IsEnabled = false;
        ToolOutPanel.Visibility = Visibility.Visible;
        BtnToolStop.Visibility = Visibility.Visible;   // a wrong host shouldn't cost a 2-minute wait
        TxtToolOut.Text = $"Traceroute to {host} — hops appear as they answer…\n";
        using var cts = new CancellationTokenSource();
        _traceCts = cts;
        try
        {
            await NetTools.TracerouteLiveAsync(host, line => Dispatcher.Invoke(() =>
            {
                TxtToolOut.Text += line + "\n";
                ToolOutScroll.ScrollToEnd();
            }), cts.Token);
            if (cts.IsCancellationRequested)
                TxtToolOut.Text += "\nStopped.";
            // tracert prints its own "Trace complete." on success — only add ours when it didn't
            // (unresolvable host), so the tech still sees the run is over.
            else if (!TxtToolOut.Text.Contains("Trace complete."))
                TxtToolOut.Text += "\nTrace complete.";
        }
        catch (Exception ex) { TxtToolOut.Text += $"\nError: {ex.Message}"; }
        finally
        {
            _traceCts = null;
            _netBusy = false;
            PnlNetDiag.IsEnabled = true;
            BtnToolStop.Visibility = Visibility.Collapsed;
            ToolOutScroll.ScrollToEnd();
        }
    }

    private void ToolStop_Click(object sender, RoutedEventArgs e) => _traceCts?.Cancel();

    private async void ToolCopy_Click(object sender, RoutedEventArgs e)
    {
        try { Clipboard.SetText(TxtToolOut.Text); } catch { return; }   // clipboard can be locked by another app
        BtnToolCopy.Content = "Copied";
        await Task.Delay(1500);
        BtnToolCopy.Content = "Copy";
    }

    private async void DnsLookup_Click(object sender, RoutedEventArgs e)
        => await RunTool(() => NetTools.DnsLookupAsync(TxtNetHost.Text.Trim(), TxtDnsType.Text.Trim()),
                         $"DNS {TxtDnsType.Text.Trim()} lookup for {TxtNetHost.Text.Trim()}…");

    private async void PortCheck_Click(object sender, RoutedEventArgs e)
    {
        var ports = NetTools.ParsePorts(TxtPort.Text, out string err);
        if (err.Length > 0) { ToolOutPanel.Visibility = Visibility.Visible; TxtToolOut.Text = err; return; }
        string host = TxtNetHost.Text.Trim();
        await RunTool(() => NetTools.PortCheckManyAsync(host, ports),
            $"Checking {ports.Count} port(s) on {host}…");
    }

    private async Task RunTool(Func<Task<string>> action, string busyText)
    {
        if (_netBusy) return;
        _netBusy = true;
        PnlNetDiag.IsEnabled = false;
        ToolOutPanel.Visibility = Visibility.Visible;   // only show the output once a tool is run
        TxtToolOut.Text = busyText;
        try { TxtToolOut.Text = await action(); }
        catch (Exception ex) { TxtToolOut.Text = ex.Message; }
        finally { _netBusy = false; PnlNetDiag.IsEnabled = true; ToolOutScroll.ScrollToTop(); }
    }

    // Lives in the Wi-Fi card (it has nothing to do with the host-based diagnostic tools).
    private async void WifiScan_Click(object sender, RoutedEventArgs e)
    {
        if (_netBusy) return;
        _netBusy = true;
        BtnWifiScan.IsEnabled = false;
        PnlWifiScan.Visibility = Visibility.Visible;
        TxtWifiScanStatus.Text = "Scanning for nearby Wi-Fi networks…";
        try
        {
            var nets = await NetTools.WifiScanAsync();
            IcWifiScan.ItemsSource = nets;
            TxtWifiScanStatus.Text = nets.Count > 0 ? $"{nets.Count} network(s) found:" : "No networks found (Wi-Fi adapter off?).";
        }
        catch (Exception ex) { TxtWifiScanStatus.Text = ex.Message; }
        finally { _netBusy = false; BtnWifiScan.IsEnabled = true; }
    }

    // ── IP SCANNER ────────────────────────────────────────────────────────

    private CancellationTokenSource? _scanCts;

    private void ScanRange_KeyDown(object sender, KeyEventArgs e)
    {
        if (e.Key == Key.Enter) Scan_Click(sender, e);
    }

    private void ScanStop_Click(object sender, RoutedEventArgs e) => _scanCts?.Cancel();

    private async void Scan_Click(object sender, RoutedEventArgs e)
    {
        if (_scanCts is not null) return;   // already scanning

        var addresses = IpScanner.ParseRange(TxtScanRange.Text, out string error);
        if (error.Length > 0)
        {
            PnlScanResults.Visibility = Visibility.Collapsed;
            TxtScanStatus.Text = "● " + error;
            TxtScanStatus.Foreground = StatusColors.Red;
            return;
        }

        bool deep = ChkScanDeep.IsChecked == true;
        ActivityLog.Action("Network", $"IP scan {TxtScanRange.Text.Trim()} ({addresses.Count} addresses{(deep ? ", deep" : "")})");

        // The results collection is owned by the UI thread; the scanner marshals onto it.
        var hosts = new System.Collections.ObjectModel.ObservableCollection<IpHost>();
        IcScanHosts.ItemsSource   = hosts;
        PnlScanResults.Visibility = Visibility.Visible;
        BtnScan.IsEnabled         = false;
        BtnScanCopy.IsEnabled     = false;
        BtnScanStop.Visibility    = Visibility.Visible;
        ScanBusy.Visibility       = Visibility.Visible;
        ScanBusy.Value            = 0;
        TxtScanStatus.Foreground  = StatusColors.Muted;
        TxtScanStatus.Text        = $"Scanning {addresses.Count:N0} address(es)…";

        var cts = new CancellationTokenSource();
        _scanCts = cts;
        var sw = System.Diagnostics.Stopwatch.StartNew();
        int found = 0;
        bool stopped = false;

        try
        {
            found = await IpScanner.ScanAsync(addresses, deep,
                host => Dispatcher.BeginInvoke(() => InsertSorted(hosts, host)),
                (done, total) => Dispatcher.BeginInvoke(() =>
                {
                    ScanBusy.Value = done * 100.0 / total;
                    TxtScanStatus.Text = $"Scanning… {done:N0} / {total:N0} addresses  ·  {hosts.Count} host(s) found";
                }),
                cts.Token);
        }
        catch (Exception ex)
        {
            TxtScanStatus.Text = "● " + ex.Message;
            TxtScanStatus.Foreground = StatusColors.Red;
        }
        finally
        {
            sw.Stop();
            stopped = cts.IsCancellationRequested;   // read it before the source is disposed
            _scanCts = null;
            cts.Dispose();
            BtnScan.IsEnabled      = true;
            BtnScanStop.Visibility = Visibility.Collapsed;
            ScanBusy.Visibility    = Visibility.Collapsed;
        }

        // A cancelled scan still shows whatever it found before the tech hit Stop.
        BtnScanCopy.IsEnabled = hosts.Count > 0;
        if (TxtScanStatus.Foreground != StatusColors.Red)
        {
            TxtScanStatus.Foreground = StatusColors.Muted;
            TxtScanStatus.Text = stopped
                ? $"● Stopped — {hosts.Count} host(s) found so far."
                : $"● {hosts.Count} host(s) alive out of {addresses.Count:N0} scanned  ·  {sw.Elapsed.TotalSeconds:F1} s";
        }
        ActivityLog.Result("Network", stopped
            ? $"IP scan stopped — {hosts.Count} host(s) found"
            : $"IP scan complete — {found} host(s) alive in {sw.Elapsed.TotalSeconds:F1} s");
    }

    // Hosts answer out of order, so slot each one into its numeric place as it arrives.
    private static void InsertSorted(IList<IpHost> hosts, IpHost host)
    {
        int i = 0;
        while (i < hosts.Count && hosts[i].Sort < host.Sort) i++;
        hosts.Insert(i, host);
    }

    private void ScanCopy_Click(object sender, RoutedEventArgs e)
    {
        if (IcScanHosts.ItemsSource is not IEnumerable<IpHost> hosts) return;
        var text = string.Join(Environment.NewLine,
            hosts.Select(h => $"{h.Ip}\t{h.NameText}\t{h.MacText}\t{h.How}\t{h.PortsText}"));
        try { Clipboard.SetText(text); TxtScanStatus.Text = "● Copied to the clipboard."; }
        catch (Exception ex) { TxtScanStatus.Text = "● Couldn't copy: " + ex.Message; }
    }

    // ── NETWORK TOOLS (moved from the old Actions tab) ────────────────────
    private void NetLog(string line) => Dispatcher.Invoke(() =>
    {
        NetLogPanel.Visibility = Visibility.Visible;   // only show the log once a tool is run
        TxtNetLog.Text += line + "\n";
        NetLogScroll.ScrollToBottom();
    });

    private async Task NetGuarded(Func<Task> action)
    {
        if (_netBusy) return;
        _netBusy = true;
        PnlNetTools.IsEnabled = false;
        TxtNetLog.Text = "";   // start each command with a fresh log (previous output was confusing)
        try { await action(); }
        catch (Exception ex) { NetLog($"  Error: {ex.Message}"); }
        finally { _netBusy = false; PnlNetTools.IsEnabled = true; }
    }

    private async Task<int> NetRun(string exe, string args, string label)
    {
        NetLog($"▶ {label}");
        var code = await ProcessRunner.RunAsync(exe, args, null, l => NetLog($"  {l}"));
        NetLog(code == 0 ? "  ✓ done" : $"  finished (exit {code})");
        return code;
    }

    private async void FlushDns_Click(object sender, RoutedEventArgs e)
        => await NetGuarded(() => NetRun("ipconfig.exe", "/flushdns", "Flush DNS"));

    private async void RenewIp_Click(object sender, RoutedEventArgs e)
    {
        // Release/renew drops connectivity for a moment — enough to cut a remote-support session.
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        await NetGuarded(async () =>
        {
            await NetRun("ipconfig.exe", "/release", "Release IP");
            await NetRun("ipconfig.exe", "/renew", "Renew IP");
        });
    }

    private async void PublicIp_Click(object sender, RoutedEventArgs e)
        => await NetGuarded(async () =>
        {
            NetLog("▶ Public IP");
            NetLog($"  Public IP: {await NetTools.PublicIpAsync()}");
        });

    private async void SpeedTest_Click(object sender, RoutedEventArgs e)
        => await NetGuarded(async () =>
        {
            NetLog("▶ Download speed test");
            NetLog("  Testing… (pulling 25 MB from Cloudflare)");
            NetLog("  " + await NetTools.SpeedTestAsync());
        });

    private async void ConnTest_Click(object sender, RoutedEventArgs e)
        => await NetGuarded(async () =>
        {
            NetLog("▶ Connectivity test");
            var adapter = NetworkInfo.GetPrimaryAdapter();
            if (adapter != null && adapter.Gateway is { Length: > 0 } gw && gw != "None")
                await NetPing("Gateway", gw);
            await NetPing("Internet (1.1.1.1)", "1.1.1.1");
            await NetPing("Google DNS (8.8.8.8)", "8.8.8.8");
            try
            {
                var h = await System.Net.Dns.GetHostEntryAsync("www.microsoft.com");
                NetLog($"  ✓ DNS resolves (microsoft.com → {h.AddressList.FirstOrDefault()})");
            }
            catch { NetLog("  ✗ DNS resolution failed"); }
        });

    private async Task NetPing(string label, string host)
    {
        try
        {
            using var p = new Ping();
            var r = await p.SendPingAsync(host, 2000);
            NetLog(r.Status == IPStatus.Success
                ? $"  ✓ {label} — {r.RoundtripTime} ms"
                : $"  ✗ {label} — {r.Status}");
        }
        catch (Exception ex) { NetLog($"  ✗ {label} — {ex.Message}"); }
    }

    // ── NETWORK RESETS ────────────────────────────────────────────────────
    // Stack reset (moved from Repair) = winsock/TCP-IP reset, keeps adapters.
    // Full reset = Windows "Network reset": netcfg -d removes every adapter and
    // networking component; Windows reinstalls them on the next reboot.

    private async void NetReset_Click(object sender, RoutedEventArgs e)
    {
        if (_netBusy) return;
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!MessageWindow.Confirm("Network Stack Reset", "Reset the network stack?",
                "This resets Winsock and the TCP/IP stack and flushes DNS.\n\n" +
                "You won't lose connectivity — your adapters, internet, and remote session stay " +
                "active. A restart is needed to fully finish, but you can do it later. Continue?",
                MessageKind.Warning, Window.GetWindow(this)))
            return;

        ActivityLog.Action("Network", "Network stack reset (winsock reset + int ip reset + flushdns)");
        await NetGuarded(async () =>
        {
            SetStatus(TxtNetResetStatus, "Running…", StatusColors.Yellow);
            await NetRun("netsh.exe", "winsock reset", "Reset Winsock catalog");
            await NetRun("netsh.exe", "int ip reset", "Reset TCP/IP stack");
            await NetRun("ipconfig.exe", "/flushdns", "Flush DNS cache");
            SetStatus(TxtNetResetStatus, "● Done — connectivity kept; restart later to finish", StatusColors.Yellow);
        });
    }

    private async void NetFullReset_Click(object sender, RoutedEventArgs e)
    {
        if (_netBusy) return;
        if (!TechGate.Verify(Window.GetWindow(this))) return;
        if (!MessageWindow.Confirm("Network Reset", "Full network reset?",
                "This removes ALL network adapters and resets every networking component to " +
                "defaults — Windows reinstalls the adapters on the next reboot.\n\n" +
                "⚠ This WILL drop your remote session, and the PC will restart automatically to " +
                "finish (it reconnects after boot). It also clears VPN clients, static IP settings " +
                "and saved Wi-Fi networks.\n\nOnly use this as a last resort. Continue?",
                MessageKind.Warning, Window.GetWindow(this)))
            return;

        ActivityLog.Action("Network", "Full network reset (netcfg -d — removes all adapters, reinstalls on reboot)");
        await NetGuarded(async () =>
        {
            SetStatus(TxtNetFullResetStatus, "Running…", StatusColors.Yellow);
            var code = await NetRun("netcfg.exe", "-d", "Network reset (remove + reinstall adapters)");
            if (code != 0)
            {
                // Don't force a reboot on a reset that didn't take — show the failure instead.
                SetStatus(TxtNetFullResetStatus, $"● Reset failed (exit {code}) — not restarting", StatusColors.Red);
                return;
            }
            // The adapters only come back after a reboot, and remote connectivity is already going —
            // so finish the job for the tech: schedule an automatic restart (short delay so this
            // status paints and the action logs before the box goes down).
            SetStatus(TxtNetFullResetStatus, "● Done — restarting automatically in ~20s to reinstall adapters…", StatusColors.Yellow);
            ActivityLog.Action("Network", "Auto-restart after full network reset (shutdown /r /t 20)");
            await NetRun("shutdown.exe",
                "/r /t 20 /c \"PartnerTool: finishing the network reset. The PC will restart now and reconnect once Windows is back up.\"",
                "Restart automatically to finish");
        });
    }

    private static void SetStatus(TextBlock tb, string text, System.Windows.Media.Brush color)
    {
        tb.Text = text;
        tb.Foreground = color;
    }

    private void LoadAdapters()
    {
        var adapters = new List<AdapterVm>();
        try
        {
            foreach (var ni in NetworkInterface.GetAllNetworkInterfaces()
                .Where(n => n.OperationalStatus == OperationalStatus.Up
                         && n.NetworkInterfaceType != NetworkInterfaceType.Loopback
                         && n.NetworkInterfaceType != NetworkInterfaceType.Tunnel))
            {
                // Guard per adapter: a single misbehaving NIC shouldn't blank the whole list.
                try
                {
                    var props = ni.GetIPProperties();

                    // Skip the WFP/QoS "Lightweight Filter" pseudo-adapters Windows layers over a real
                    // NIC: they report Up and share the NIC's MAC but carry no usable IP, so they'd show
                    // as duplicate cards full of "N/A". Keep only adapters with a real IPv4 or global
                    // IPv6 address (a genuine link with a link-local-only address is not useful here).
                    bool hasUsableIp = props.UnicastAddresses.Any(a =>
                        a.Address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork ||
                        (a.Address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetworkV6
                         && !a.Address.IsIPv6LinkLocal));
                    if (!hasUsableIp) continue;

                    var ipv4  = props.UnicastAddresses
                        .FirstOrDefault(a => a.Address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork);
                    // Prefer the IPv4 default gateway so this matches the System Info page (which shows
                    // 10.1.1.1, not the fe80:: link-local that FirstOrDefault() would otherwise pick).
                    var gw    = (props.GatewayAddresses
                                    .FirstOrDefault(g => g.Address.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork)
                                 ?? props.GatewayAddresses.FirstOrDefault())?.Address.ToString() ?? "None";
                    var dns   = string.Join(", ", props.DnsAddresses
                        .Where(a => a.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork)
                        .Select(a => a.ToString()));
                    var mac   = string.Join(":", ni.GetPhysicalAddress().GetAddressBytes().Select(b => b.ToString("X2")));

                    adapters.Add(new AdapterVm
                    {
                        Header     = ni.Name.ToUpperInvariant(),
                        Type       = ni.NetworkInterfaceType.ToString(),
                        IpAddress  = ipv4?.Address.ToString() ?? "N/A",
                        SubnetMask = ipv4?.IPv4Mask.ToString()  ?? "N/A",
                        Gateway    = gw,
                        Dns        = string.IsNullOrEmpty(dns) ? "N/A" : dns,
                        Mac        = mac,
                    });
                }
                catch { }
            }
        }
        catch { }

        AdapterList.ItemsSource = adapters.Count > 0
            ? adapters
            : new List<AdapterVm> { new() { Header = "NO ACTIVE ADAPTERS", IpAddress = "—", SubnetMask = "—", Gateway = "—", Dns = "—", Mac = "—" } };
    }

    private async void Ping_Click(object sender, RoutedEventArgs e)
        => await RunTool(() => PingAsync(TxtNetHost.Text.Trim()), $"Pinging {TxtNetHost.Text.Trim()}…");

    private static async Task<string> PingAsync(string target)
    {
        if (string.IsNullOrEmpty(target)) return "Enter a host or IP.";
        using var ping = new Ping();
        var times = new List<long>();
        for (int i = 0; i < 4; i++)
        {
            var reply = await Task.Run(() => ping.Send(target, 2000));
            if (reply.Status == IPStatus.Success) times.Add(reply.RoundtripTime);
        }
        return times.Count > 0
            ? $"● Reachable — avg {times.Average():F0} ms  |  min {times.Min()} ms  |  max {times.Max()} ms  ({times.Count}/4 replies)"
            : "● Unreachable — all 4 packets lost";
    }
}

public class AdapterVm
{
    public string Header     { get; set; } = "";
    public string Type       { get; set; } = "";
    public string IpAddress  { get; set; } = "";
    public string SubnetMask { get; set; } = "";
    public string Gateway    { get; set; } = "";
    public string Dns        { get; set; } = "";
    public string Mac        { get; set; } = "";
}
```
