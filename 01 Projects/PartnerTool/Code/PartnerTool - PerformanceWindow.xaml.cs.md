---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\PerformanceWindow.xaml.cs
---

# PartnerTool\PerformanceWindow.xaml.cs

```csharp
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;
using System.Windows.Media;
using System.Windows.Shapes;
using System.Windows.Threading;

namespace PartnerTool;

public record StatRow(string Label, string Value);

/// <summary>
/// Task-Manager-style live Performance view: a left rail (CPU / Memory / Disk / Network) with
/// sparklines, and a big live graph + advanced stats for the selected resource. Live numbers come
/// from <see cref="LivePerf"/> (sampled off the UI thread); the static detail from
/// <see cref="PerfDetails"/>. Opened from the System Info LIVE STATS tiles.
/// </summary>
public partial class PerformanceWindow : Window
{
    private const int Cap = 60;
    private readonly DispatcherTimer _timer = new() { Interval = TimeSpan.FromSeconds(1) };
    private readonly LivePerf _live = new();
    private readonly List<double> _cpu = new(), _ram = new(), _disk = new(), _net = new();
    private readonly List<List<double>> _coreHist = new();   // per-logical-processor history (CPU view)
    private readonly List<Polyline> _coreLines = new();

    private string _sel = "cpu";
    private bool _ticking;
    private int _tick;
    private PerfDetails? _d;
    private LiveSample? _last;
    private (int procs, int threads, int handles) _counts;

    private static readonly SolidColorBrush SelBorder = Frozen("#89B4FA");   // selected tile — bright accent
    private static readonly SolidColorBrush OffBorder = Frozen("#313244");   // unselected tile — subtle box

    public PerformanceWindow(string initial = "cpu")
    {
        InitializeComponent();
        _sel = Normalize(initial);

        Loaded += async (_, _) =>
        {
            Select(_sel);
            try { _d = await System.Threading.Tasks.Task.Run(PerfDetails.Collect); } catch { }
            try { _counts = await System.Threading.Tasks.Task.Run(PerfDetails.ProcessCounts); } catch { }
            BuildCoreGrid(_d?.Logical ?? 0);
            Select(_sel);                 // re-render stats with the details in
            _timer.Tick += (_, _) => Tick();
            _timer.Start();
            Tick();
        };
        Closed += (_, _) => _timer.Stop();
    }

    /// <summary>Switch the focused resource from outside (clicking a different tile on System Info).</summary>
    public void SelectResource(string res) => Select(Normalize(res));

    private static string Normalize(string r) => r.ToLowerInvariant() switch
    {
        "mem" or "memory"       => "mem",
        "disk"                  => "disk",
        "net" or "network"      => "net",
        "proc" or "processes"   => "proc",
        _                       => "cpu",
    };

    private void Tile_Click(object sender, RoutedEventArgs e)
    {
        if (sender is FrameworkElement { Tag: string tag }) Select(Normalize(tag));
    }

    private void Select(string res)
    {
        _sel = res;
        Mark(TileProc, res == "proc");
        Mark(TileCpu,  res == "cpu");
        Mark(TileMem,  res == "mem");
        Mark(TileDisk, res == "disk");
        Mark(TileNet,  res == "net");

        static void Mark(System.Windows.Controls.Button tile, bool on)
        {
            tile.BorderBrush     = on ? SelBorder : OffBorder;   // template-bound into the tile border
            tile.BorderThickness = new Thickness(on ? 1.8 : 1);
        }

        // Processes swaps the whole right side for the live process list (its own 2 s sampler
        // starts/stops with visibility). Created on first use so the window opens fast.
        bool proc = res == "proc";
        if (proc && ProcHost.Content == null) ProcHost.Content = new Pages.ProcessesPage();
        ProcHost.Visibility  = proc ? Visibility.Visible   : Visibility.Collapsed;
        GraphCard.Visibility = proc ? Visibility.Collapsed : Visibility.Visible;
        IcStats.Visibility   = proc ? Visibility.Collapsed : Visibility.Visible;
        if (proc)
        {
            TxtResTitle.Text = "Processes";
            TxtResSub.Text   = "";
            return;
        }

        // CPU shows the per-logical-processor grid; everything else shows one overall graph.
        bool cpu = res == "cpu" && _coreLines.Count > 0;
        SingleGraph.Visibility = cpu ? Visibility.Collapsed : Visibility.Visible;
        CorePanel.Visibility   = cpu ? Visibility.Visible   : Visibility.Collapsed;

        (string title, string sub, Brush stroke) = res switch
        {
            "mem"  => ("Memory", _d is { } d1 ? $"{d1.RamTotalGb:F1} GB {d1.RamType}".Trim() : "", Frozen("#A6E3A1")),
            "disk" => ("Disk", _d?.DiskModel ?? "", Frozen("#F9E2AF")),
            "net"  => ("Network", _d?.NetName ?? "", Frozen("#CBA6F7")),
            _      => ("CPU", _d?.CpuName ?? "", Frozen("#89B4FA")),
        };
        TxtResTitle.Text = title;
        TxtResSub.Text   = sub;
        BigLine.Stroke   = stroke;
        RenderStats();
        PlotBig();
    }

    private async void Tick()
    {
        if (_ticking) return;
        _ticking = true;
        try
        {
            var s = await System.Threading.Tasks.Task.Run(() => _live.Sample());
            _last = s;

            Push(_cpu, s.CpuPct); Push(_ram, s.RamPct); Push(_disk, s.DiskPct);
            Push(_net, s.NetDownMbps + s.NetUpMbps);

            TxtCpuRail.Text  = $"{s.CpuPct:F0}%";
            TxtMemRail.Text  = $"{s.RamPct:F0}%";
            TxtDiskRail.Text = $"{s.DiskPct:F0}%";
            TxtNetRail.Text  = $"↓{s.NetDownMbps:F1}  ↑{s.NetUpMbps:F1} Mbps";

            Plot(CpuMini,  _cpu,  100);
            Plot(MemMini,  _ram,  100);
            Plot(DiskMini, _disk, 100);
            Plot(NetMini,  _net,  NetMax());

            PlotBig();

            // Per-core graphs (CPU view only — extra WMI poll, so skip it otherwise).
            if (_sel == "cpu" && _coreLines.Count > 0)
            {
                var cores = await System.Threading.Tasks.Task.Run(PerfDetails.CoreLoads);
                int n = Math.Min(cores.Length, _coreLines.Count);
                for (int i = 0; i < n; i++)
                {
                    Push(_coreHist[i], cores[i]);
                    Plot(_coreLines[i], _coreHist[i], 100);
                }
            }

            if (_tick % 5 == 0)
                try { _counts = await System.Threading.Tasks.Task.Run(PerfDetails.ProcessCounts); } catch { }
            _tick++;
            TxtProcRail.Text = _counts.procs > 0 ? $"{_counts.procs} running" : "—";

            if (_sel != "proc") RenderStats();   // the Processes view has no stats panel
        }
        catch { }
        finally { _ticking = false; }
    }

    private void BuildCoreGrid(int count)
    {
        CoreGrid.Children.Clear();
        _coreLines.Clear();
        _coreHist.Clear();
        if (count <= 0) return;

        CoreGrid.Columns = (int)Math.Ceiling(Math.Sqrt(count));
        var stroke = Frozen("#89B4FA");
        var border = Frozen("#313244");
        for (int i = 0; i < count; i++)
        {
            var line = new Polyline { Stroke = stroke, StrokeThickness = 1.0 };
            CoreGrid.Children.Add(new Border
            {
                BorderBrush     = border,
                BorderThickness = new Thickness(1),
                CornerRadius    = new CornerRadius(3),
                Margin          = new Thickness(2),
                ClipToBounds    = true,
                Child           = line,
            });
            _coreLines.Add(line);
            _coreHist.Add(new List<double>());
        }
    }

    private double NetMax() => Math.Max(10, _net.Count > 0 ? _net.Max() : 10);

    private void PlotBig()
    {
        var (hist, max, label) = _sel switch
        {
            "mem"  => (_ram,  100.0,    "100%"),
            "disk" => (_disk, 100.0,    "100%"),
            "net"  => (_net,  NetMax(), $"{NetMax():F0} Mbps"),
            _      => (_cpu,  100.0,    "100%"),
        };
        TxtGraphMax.Text = label;
        Plot(BigLine, hist, max);
    }

    private void RenderStats()
    {
        var d = _d ?? new PerfDetails();
        var s = _last;
        var rows = _sel switch
        {
            "mem" => MemRows(d, s),
            "disk" => new List<StatRow>
            {
                new("Active",   s is { } ds ? $"{ds.DiskPct:F0}%" : "—"),
                new("Model",    d.DiskModel),
                new("Type",     d.DiskType),
                new("Capacity", d.DiskSizeGb > 0 ? $"{d.DiskSizeGb:F0} GB" : "—"),
            },
            "net" => new List<StatRow>
            {
                new("Receive",    s is { } ns ? $"{ns.NetDownMbps:F1} Mbps" : "—"),
                new("Send",       s is { } us ? $"{us.NetUpMbps:F1} Mbps"   : "—"),
                new("Adapter",    d.NetName),
                new("Type",       d.NetType),
                new("IP address", d.NetIp),
            },
            _ => new List<StatRow>
            {
                new("Utilization",       s is { } cs ? $"{cs.CpuPct:F0}%" : "—"),
                new("Base speed",        d.BaseGhz > 0 ? $"{d.BaseGhz:F2} GHz" : "—"),
                new("Sockets",           d.Sockets.ToString()),
                new("Cores",             d.Cores.ToString()),
                new("Logical processors",d.Logical.ToString()),
                new("Virtualization",    d.Virtualization),
                new("L1 cache",          d.L1),
                new("L2 cache",          d.L2),
                new("L3 cache",          d.L3),
                new("Processes",         _counts.procs.ToString()),
                new("Threads",           _counts.threads.ToString()),
                new("Handles",           _counts.handles.ToString()),
                new("Up time",           d.Uptime),
            },
        };
        IcStats.ItemsSource = rows;
    }

    private static List<StatRow> MemRows(PerfDetails d, LiveSample? s)
    {
        double used = s is { } ms ? d.RamTotalGb * ms.RamPct / 100.0 : 0;
        return new List<StatRow>
        {
            new("In use",     s is { } ms2 ? $"{used:F1} GB  ({ms2.RamPct:F0}%)" : "—"),
            new("Available",  d.RamTotalGb > 0 ? $"{Math.Max(0, d.RamTotalGb - used):F1} GB" : "—"),
            new("Total",      d.RamTotalGb > 0 ? $"{d.RamTotalGb:F1} GB" : "—"),
            new("Type",       d.RamType),
            new("Speed",      d.RamSpeed > 0 ? $"{d.RamSpeed} MHz" : "—"),
            new("Slots used", d.Slots),
        };
    }

    private static void Push(List<double> data, double v)
    {
        data.Add(v);
        if (data.Count > Cap) data.RemoveAt(0);
    }

    private static void Plot(Polyline line, List<double> data, double max)
    {
        if (line.Parent is not FrameworkElement host) { line.Points = null; return; }
        double w = host.ActualWidth, h = host.ActualHeight;
        if (w <= 0 || h <= 0 || data.Count < 2 || max <= 0) { line.Points = null; return; }
        var pts = new PointCollection();
        for (int i = 0; i < data.Count; i++)
        {
            double x = w * i / (Cap - 1);
            double y = h - Math.Clamp(data[i] / max, 0, 1) * h;
            pts.Add(new Point(x, y));
        }
        line.Points = pts;
    }

    private static SolidColorBrush Frozen(string hex)
    {
        var b = new SolidColorBrush((Color)ColorConverter.ConvertFromString(hex));
        b.Freeze();
        return b;
    }
}
```
