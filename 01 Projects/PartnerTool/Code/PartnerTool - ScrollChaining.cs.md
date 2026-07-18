---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\ScrollChaining.cs
---

# PartnerTool\ScrollChaining.cs

```csharp
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;

namespace PartnerTool;

/// <summary>
/// Fixes the "mouse over an inline log blocks page scrolling" annoyance: a WPF ScrollViewer
/// swallows every mouse-wheel event, even when it can't scroll any further (or has nothing to
/// scroll). With this behavior enabled, an inner scroller only consumes the wheel while it can
/// actually move in that direction — at its edge, the event is re-raised on its parent so the
/// page's outer ScrollViewer keeps scrolling naturally.
///
/// Enable per page with an implicit style:
///   &lt;Style TargetType="ScrollViewer"&gt;
///       &lt;Setter Property="pt:ScrollChaining.Enabled" Value="True"/&gt;
///   &lt;/Style&gt;
/// (Harmless on the outer scroller itself — at its edge the forwarded event just goes nowhere.)
/// </summary>
public static class ScrollChaining
{
    public static readonly DependencyProperty EnabledProperty = DependencyProperty.RegisterAttached(
        "Enabled", typeof(bool), typeof(ScrollChaining), new PropertyMetadata(false, OnEnabledChanged));

    public static void SetEnabled(DependencyObject d, bool value) => d.SetValue(EnabledProperty, value);
    public static bool GetEnabled(DependencyObject d) => (bool)d.GetValue(EnabledProperty);

    private static void OnEnabledChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is not ScrollViewer sv) return;
        if ((bool)e.NewValue) sv.PreviewMouseWheel += OnPreviewMouseWheel;
        else sv.PreviewMouseWheel -= OnPreviewMouseWheel;
    }

    private static void OnPreviewMouseWheel(object sender, MouseWheelEventArgs e)
    {
        if (sender is not ScrollViewer sv || e.Handled) return;

        bool canScrollDown = e.Delta < 0 && sv.VerticalOffset < sv.ScrollableHeight;
        bool canScrollUp   = e.Delta > 0 && sv.VerticalOffset > 0;
        if (canScrollDown || canScrollUp) return;   // the inner scroller can handle it — let it

        // At the edge (or nothing to scroll): hand the wheel to the parent chain instead.
        e.Handled = true;
        var forwarded = new MouseWheelEventArgs(e.MouseDevice, e.Timestamp, e.Delta)
        {
            RoutedEvent = UIElement.MouseWheelEvent,
            Source      = sender,
        };
        (sv.Parent as UIElement)?.RaiseEvent(forwarded);
    }
}
```
