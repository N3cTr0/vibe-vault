---
project: Octavia
tags: [octavia, code]
source-path: src\Octavia.App\Face\StaticFiles.cs
---

# src\Octavia.App\Face\StaticFiles.cs

```csharp
using Octavia.Core;

namespace Octavia.Face;

/// The face, served over HTTP so a renderer that is not a WebView2 can load it.
///
/// **Why this exists at all.** The built-in page reaches `wwwroot` through
/// `CoreWebView2.SetVirtualHostNameToFolderMapping`, which is a WebView2 feature and not a
/// server — nothing in this process ever listened for a GET. That was fine while every face
/// was either the built-in page or a browser on this machine holding the same trick. It
/// stops being fine for a phone, which has no equivalent of a virtual host and cannot load
/// `https://octavia.face/` at all.
///
/// The alternative was vendoring `wwwroot` into the client, and it was rejected for a
/// specific reason rather than a stylistic one: a VRM is *user data*. It lives in her data
/// folder, it is chosen at runtime from the `avatars[]` list, and it is in no repository —
/// so it can never be baked into a client, and the host would have had to serve files
/// anyway. Two mechanisms and a sync script, to avoid one. See ROADMAP.md stage 13/14.
///
/// Two roots, mirroring the two virtual hosts they replace:
///
///   `/`         → `wwwroot`     (the page: html, css, js, the vendored three.js)
///   `/avatars/` → her data dir  (the characters, which are not in the install)
internal static class StaticFiles
{
    /// What a browser needs to be told, for the things she actually serves. Anything not
    /// named here is refused rather than guessed at: a wrong `Content-Type` on a module
    /// script fails in a way that reads as a syntax error, and hunting that twice is once
    /// too many.
    private static readonly Dictionary<string, string> Types = new(StringComparer.OrdinalIgnoreCase)
    {
        [".html"] = "text/html; charset=utf-8",
        [".css"] = "text/css; charset=utf-8",
        [".js"] = "text/javascript; charset=utf-8",
        [".mjs"] = "text/javascript; charset=utf-8",
        [".json"] = "application/json; charset=utf-8",
        [".svg"] = "image/svg+xml",
        [".png"] = "image/png",
        [".jpg"] = "image/jpeg",
        [".jpeg"] = "image/jpeg",
        [".webp"] = "image/webp",
        [".ico"] = "image/x-icon",
        [".woff2"] = "font/woff2",
        [".wasm"] = "application/wasm",
        [".vrm"] = "application/octet-stream",
        [".glb"] = "model/gltf-binary",
        [".gltf"] = "model/gltf+json",
        [".bin"] = "application/octet-stream",
        [".ktx2"] = "image/ktx2",
    };

    /// The file a request path names, or null if there isn't one it may have.
    ///
    /// Null covers three different refusals on purpose — missing, unreadable, and *not
    /// allowed* — because the caller answers all three with a 404. Telling a stranger
    /// apart from a typo is a courtesy this socket does not owe anyone.
    public static (string Path, string Type)? Resolve(string target)
    {
        // Strip the query and fragment; only the path names a file.
        var cut = target.IndexOfAny(['?', '#']);
        var path = cut >= 0 ? target[..cut] : target;

        try
        {
            path = Uri.UnescapeDataString(path);
        }
        catch (Exception ex)
        {
            Log.Write($"face http: undecodable path: {ex.Message}");
            return null;
        }

        if (path.Length == 0 || path[0] != '/') return null;
        if (path is "/" or "") path = "/index.html";

        string root, relative;

        if (path.StartsWith("/avatars/", StringComparison.OrdinalIgnoreCase))
        {
            root = Paths.AvatarDir;
            relative = path["/avatars/".Length..];
        }
        else
        {
            root = Paths.FaceRoot;
            relative = path[1..];
        }

        if (relative.Length == 0) return null;

        // Traversal is checked on the *resolved* path rather than by looking for "..",
        // because the ways of spelling that are endless and the ways of leaving a root
        // are not. A symlink out of wwwroot would defeat a textual check and does not
        // defeat this one.
        string full;
        try
        {
            full = Path.GetFullPath(Path.Combine(root, relative.Replace('/', Path.DirectorySeparatorChar)));
        }
        catch (Exception ex)
        {
            Log.Write($"face http: unresolvable path: {ex.Message}");
            return null;
        }

        var fence = Path.GetFullPath(root).TrimEnd(Path.DirectorySeparatorChar) + Path.DirectorySeparatorChar;
        if (!full.StartsWith(fence, StringComparison.OrdinalIgnoreCase))
        {
            Log.Warn($"face http: refused a path outside its root: {path}");
            return null;
        }

        if (!File.Exists(full)) return null;

        // An unknown extension is refused rather than sent as octet-stream. This server
        // exists to serve a known page and a known character format; it is not a file
        // share, and anything new arriving here should be a deliberate line in Types.
        if (!Types.TryGetValue(Path.GetExtension(full), out var type))
        {
            Log.Warn($"face http: refused an unserved file type: {path}");
            return null;
        }

        return (full, type);
    }
}
```
