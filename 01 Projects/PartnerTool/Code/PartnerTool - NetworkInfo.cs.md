---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\NetworkInfo.cs
---

# PartnerTool\NetworkInfo.cs

```csharp
using System.Management;
using System.Net.NetworkInformation;
using System.Net.Sockets;

namespace PartnerTool;

public class AdapterSummary
{
    public string Name       { get; init; } = "";
    public string Type       { get; init; } = "";
    public string IpAddress  { get; init; } = "";
    public string SubnetMask { get; init; } = "";
    public string Gateway    { get; init; } = "";
    public string Dns        { get; init; } = "";
    public string Mac        { get; init; } = "";

    // DHCP / lease (Win32_NetworkAdapterConfiguration)
    public bool?     DhcpEnabled   { get; init; }
    public string    DhcpServer    { get; init; } = "";
    public DateTime? LeaseObtained { get; init; }
    public DateTime? LeaseExpires  { get; init; }

    public string IpAssignment => DhcpEnabled switch
    {
        true  => "DHCP",
        false => "Static",
        _     => "Unknown",
    };
}

public static class NetworkInfo
{
    private static readonly string[] VirtualKeywords =
        ["vmware", "virtualbox", "vethernet", "hyper-v", "virtual", "bluetooth",
         "wi-fi direct", "miniport", "tap-windows", "cisco", "nordvpn", "expressvpn"];

    /// <summary>
    /// The adapter the machine is actually using: up, with an IPv4 default gateway,
    /// not a virtual/VPN adapter, and — when several qualify — the one carrying the
    /// most traffic. Returns null if nothing qualifies.
    /// </summary>
    public static AdapterSummary? GetPrimaryAdapter()
    {
        try { return GetPrimaryAdapterCore(); }
        catch { return null; }   // a single misbehaving adapter shouldn't break the whole snapshot
    }

    private static AdapterSummary? GetPrimaryAdapterCore()
    {
        var ni = NetworkInterface.GetAllNetworkInterfaces()
            .Where(n =>
                n.OperationalStatus == OperationalStatus.Up &&
                n.NetworkInterfaceType != NetworkInterfaceType.Loopback &&
                n.NetworkInterfaceType != NetworkInterfaceType.Tunnel)
            .Where(n =>
            {
                // Must have a real IPv4 gateway — virtual adapters rarely do
                var props = n.GetIPProperties();
                return props.GatewayAddresses.Any(g =>
                    g.Address.AddressFamily == AddressFamily.InterNetwork);
            })
            .Where(n =>
            {
                var id = (n.Name + " " + n.Description).ToLowerInvariant();
                return !VirtualKeywords.Any(id.Contains);
            })
            // Prefer the adapter with the most traffic — the one actually in use
            .OrderByDescending(n =>
            {
                try { var s = n.GetIPStatistics(); return s.BytesSent + s.BytesReceived; }
                catch { return 0L; }
            })
            .FirstOrDefault();

        if (ni == null) return null;

        var props2 = ni.GetIPProperties();
        var ipv4   = props2.UnicastAddresses
            .FirstOrDefault(a => a.Address.AddressFamily == AddressFamily.InterNetwork);
        var gw  = props2.GatewayAddresses
            .FirstOrDefault(a => a.Address.AddressFamily == AddressFamily.InterNetwork)
            ?.Address.ToString() ?? "None";
        var dns = string.Join(", ", props2.DnsAddresses
            .Where(a => a.AddressFamily == AddressFamily.InterNetwork)
            .Select(a => a.ToString()));
        var mac = string.Join(":", ni.GetPhysicalAddress().GetAddressBytes()
            .Select(b => b.ToString("X2")));

        var ip = ipv4?.Address.ToString() ?? "";
        var (dhcpOn, dhcpServer, obtained, expires) = GetDhcp(ip);

        return new AdapterSummary
        {
            Name       = ni.Name,
            Type       = ni.NetworkInterfaceType.ToString(),
            IpAddress  = string.IsNullOrEmpty(ip) ? "N/A" : ip,
            SubnetMask = ipv4?.IPv4Mask.ToString() ?? "N/A",
            Gateway    = gw,
            Dns        = string.IsNullOrEmpty(dns) ? "N/A" : dns,
            Mac        = mac,
            DhcpEnabled   = dhcpOn,
            DhcpServer    = dhcpServer,
            LeaseObtained = obtained,
            LeaseExpires  = expires,
        };
    }

    /// <summary>Every up, non-loopback, non-tunnel adapter (for the full diagnostics report).</summary>
    public static List<AdapterSummary> AllAdapters()
    {
        var list = new List<AdapterSummary>();
        try
        {
            foreach (var ni in NetworkInterface.GetAllNetworkInterfaces()
                .Where(n => n.OperationalStatus == OperationalStatus.Up
                         && n.NetworkInterfaceType != NetworkInterfaceType.Loopback
                         && n.NetworkInterfaceType != NetworkInterfaceType.Tunnel))
            {
                try
                {
                    var props = ni.GetIPProperties();
                    var ipv4  = props.UnicastAddresses
                        .FirstOrDefault(a => a.Address.AddressFamily == AddressFamily.InterNetwork);
                    var gw    = props.GatewayAddresses
                        .FirstOrDefault(a => a.Address.AddressFamily == AddressFamily.InterNetwork)
                        ?.Address.ToString() ?? "None";
                    var dns   = string.Join(", ", props.DnsAddresses
                        .Where(a => a.AddressFamily == AddressFamily.InterNetwork).Select(a => a.ToString()));
                    var mac   = string.Join(":", ni.GetPhysicalAddress().GetAddressBytes().Select(b => b.ToString("X2")));
                    var ip    = ipv4?.Address.ToString() ?? "";
                    var (dhcpOn, dhcpServer, obtained, expires) = GetDhcp(ip);

                    list.Add(new AdapterSummary
                    {
                        Name       = ni.Name,
                        Type       = ni.NetworkInterfaceType.ToString(),
                        IpAddress  = string.IsNullOrEmpty(ip) ? "N/A" : ip,
                        SubnetMask = ipv4?.IPv4Mask.ToString() ?? "N/A",
                        Gateway    = gw,
                        Dns        = string.IsNullOrEmpty(dns) ? "N/A" : dns,
                        Mac        = mac,
                        DhcpEnabled   = dhcpOn,
                        DhcpServer    = dhcpServer,
                        LeaseObtained = obtained,
                        LeaseExpires  = expires,
                    });
                }
                catch { }
            }
        }
        catch { }
        return list;
    }

    /// <summary>DHCP state + lease times for the adapter holding <paramref name="ip"/>.</summary>
    private static (bool? enabled, string server, DateTime? obtained, DateTime? expires) GetDhcp(string ip)
    {
        if (string.IsNullOrEmpty(ip)) return (null, "", null, null);
        try
        {
            using var q = new ManagementObjectSearcher(
                "SELECT DHCPEnabled, DHCPServer, DHCPLeaseObtained, DHCPLeaseExpires, IPAddress " +
                "FROM Win32_NetworkAdapterConfiguration WHERE IPEnabled=True");
            foreach (ManagementObject o in q.Get())
            using (o)
            {
                if (o["IPAddress"] is not string[] ips || !ips.Contains(ip)) continue;
                bool?  en      = o["DHCPEnabled"] as bool?;
                string server  = o["DHCPServer"]?.ToString() ?? "";
                return (en, server, ToDt(o["DHCPLeaseObtained"]?.ToString()), ToDt(o["DHCPLeaseExpires"]?.ToString()));
            }
        }
        catch { }
        return (null, "", null, null);
    }

    private static DateTime? ToDt(string? wmi)
    {
        if (string.IsNullOrWhiteSpace(wmi)) return null;
        try { var dt = ManagementDateTimeConverter.ToDateTime(wmi); return dt.Year > 1601 ? dt : null; }
        catch { return null; }
    }
}
```
