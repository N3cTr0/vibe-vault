---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\ui\main\FacePanel.kt
---

# app\src\main\java\com\n3ctr0\octavia\ui\main\FacePanel.kt

```kotlin
package com.n3ctr0.octavia.ui.main

import android.annotation.SuppressLint
import android.net.Uri
import android.view.ViewGroup
import android.webkit.WebView
import android.webkit.WebViewClient
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.viewinterop.AndroidView
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.net.FaceSocket

/**
 * Her actual face, rendered by the same page her desktop window uses.
 *
 * **This is what route A was chosen for.** Her socket serves `wwwroot` over HTTP since her
 * v0.20.0, so this loads *her* renderer rather than a reimplementation of it — the room, the
 * VRM, the lighting and the lip sync all arrive as they are, and every future change to her
 * face reaches this device without any Android work.
 *
 * **The page brings its own socket.** `bridge.js` addresses the origin it was served from,
 * so this WebView attaches as a face in its own right, separate from the native connection
 * that owns the microphone, the floor and her voice. Two connections from one handset is
 * slightly odd in her logs and otherwise clean: the page never asks for `audio`, so there is
 * no doubling, and the native side keeps everything sensory.
 *
 * **It is destroyed when hidden, not merely hidden.** A WebView holding a WebGL context and
 * a VRM is not something to leave running behind a collapsed panel on a phone.
 *
 * The parameter is named `config` rather than `settings` on purpose: inside `apply`, the
 * WebView's own `settings` property wins, and a silent shadow there would be a genuinely
 * nasty bug to read.
 */
@SuppressLint("SetJavaScriptEnabled")
@Composable
fun FacePanel(config: Settings, modifier: Modifier = Modifier) {

    /* Not `?token=` — the credential may be the durable remote key, and the host refuses a
       *token* from anything but loopback. Over a USB tunnel every connection is loopback so
       the distinction never showed; it appeared the moment this moved to WiFi.

       `room` rides along because **this panel is a face in its own right**, with its own
       socket to her. Without it the app's own two connections would sit in different spaces
       and this would render the desktop's conversation while the native side held ours.
       The page passes it on in `ready`; the host ignores it until her rooms work lands. */
    val url = buildString {
        append("http://${config.host}:${config.port}/?")
        append(FaceSocket.credentialParam(config.credential))
        if (config.room.isNotBlank()) append("&room=${Uri.encode(config.room)}")
    }

    AndroidView(
        modifier = modifier,
        factory = { context ->
            // Lets `chrome://inspect` reach it from this machine, which is the only way to
            // see a WebGL failure that presents as a blank rectangle.
            WebView.setWebContentsDebuggingEnabled(true)

            WebView(context).apply {
                layoutParams = ViewGroup.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT,
                    ViewGroup.LayoutParams.MATCH_PARENT
                )
                settings.javaScriptEnabled = true
                settings.domStorageEnabled = true

                // Keep her inside. Nothing in the page navigates away, and a face that can
                // wander off to the open web is a different and much worse thing.
                webViewClient = WebViewClient()

                setBackgroundColor(android.graphics.Color.BLACK)
                loadUrl(url)
            }
        },
        onRelease = { web ->
            web.loadUrl("about:blank")
            web.destroy()
        },
    )
}
```
