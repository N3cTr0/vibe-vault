---
project: PartnerTool
tags: [partnertool, code]
source-path: PartnerTool\Dialog.cs
---

# PartnerTool\Dialog.cs

```csharp
using System.Windows;

namespace PartnerTool;

/// <summary>
/// One popup at a time.
///
/// <see cref="Window.ShowDialog"/> only disables its own owner, so a second dialog opened while the
/// first is up leaves BOTH the main window and the first dialog dead until the top one is closed -
/// the tech is stuck behind two windows with no obvious way back. Every content dialog (About,
/// Settings, BitLocker keys, SMART, power plans, the list popups) goes through <see cref="Show"/>,
/// which brings the open one to the front instead of stacking a new one on it.
///
/// Prompts - <see cref="MessageWindow"/> and the tech gate - deliberately do NOT use this: they
/// belong to whatever is on screen and must be able to appear over a dialog, or an error raised
/// from inside one would never be seen. They use <see cref="OwnerFor"/> instead so they open over
/// the dialog rather than behind it.
/// </summary>
public static class Dialog
{
    private static Window? _open;

    /// <summary>The content dialog currently up, if any.</summary>
    public static Window? Current => _open;

    /// <summary>
    /// Show a content dialog modally. If one is already open it's activated instead and this
    /// returns null without opening anything.
    /// </summary>
    public static bool? Show(Window dialog, Window? owner = null)
    {
        if (_open != null)
        {
            try { _open.Activate(); } catch { }
            return null;
        }

        dialog.Owner = owner ?? Application.Current?.MainWindow;
        if (dialog.Owner != null) dialog.WindowStartupLocation = WindowStartupLocation.CenterOwner;

        _open = dialog;
        try { return dialog.ShowDialog(); }
        finally { _open = null; }
    }

    /// <summary>
    /// Best owner for a prompt: the open dialog if there is one, so the prompt lands on top of it
    /// rather than behind it, otherwise the caller's choice, otherwise the main window.
    /// </summary>
    public static Window? OwnerFor(Window? preferred) =>
        _open ?? preferred ?? Application.Current?.MainWindow;
}
```
