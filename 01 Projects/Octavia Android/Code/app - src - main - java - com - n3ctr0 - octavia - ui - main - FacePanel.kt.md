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
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.viewinterop.AndroidView
import com.n3ctr0.octavia.camera.DeviceSenses
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.net.FaceSocket
import com.n3ctr0.octavia.web.DeviceSettings
import com.n3ctr0.octavia.web.EmbedderBridge

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
fun FacePanel(
    config: Settings,
    senses: DeviceSenses?,
    onTalking: suspend (Boolean) -> Unit,
    /** Always-on listening, from her microphone button's tap. Throws so a refusal reaches
     *  the page rather than leaving a button that did nothing. */
    onListening: suspend (Boolean) -> Unit,
    /** A setting was changed from inside her page. True when the connection has to be
     *  reopened for it to mean anything. */
    onChanged: (reconnect: Boolean) -> Unit,
    modifier: Modifier = Modifier,
) {
    val scope = rememberCoroutineScope()

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

    var bridge by remember { mutableStateOf<EmbedderBridge?>(null) }

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

                /* Her page's console, in `adb logcat`.
                   A WebView swallows `console.*` and every uncaught exception unless asked
                   for them, so a script error in her renderer presented here as a control
                   that quietly did nothing — which is exactly how an evening went. On the
                   desktop this is a keypress away in devtools; on a handset it is nothing at
                   all without this. */
                webChromeClient = object : android.webkit.WebChromeClient() {
                    override fun onConsoleMessage(m: android.webkit.ConsoleMessage): Boolean {
                        android.util.Log.i(
                            "FacePage",
                            "${m.messageLevel()} ${m.message()} (${m.sourceId()}:${m.lineNumber()})"
                        )
                        return true
                    }
                }

                setBackgroundColor(android.graphics.Color.BLACK)

                /* Offered before the page is loaded, because her renderer reads
                   `window.OctaviaEmbedder` once as it starts. A bridge attached afterwards
                   would leave the buttons hidden for the life of the page and look exactly
                   like the feature not working. */
                if (senses != null) {
                    EmbedderBridge.originOf(url)?.let { origin ->
                        bridge = EmbedderBridge(
                            webView = this,
                            origin = origin,
                            scope = scope,
                            senses = senses.lends(),
                            onTalking = onTalking,
                            onWatch = { on -> senses.watch(on) },
                            onListening = onListening,
                            describe = { DeviceSettings.describe(config) },
                            apply = { key, value -> DeviceSettings.apply(config, key, value, onChanged) },
                        ).also { it.attach() }
                    }

                    /* Her eyes, driven from this device's camera. The same call her own
                       `watch.js` makes — `Face.look(null)` hands her back her own saccades —
                       so the page cannot tell which side of the WebView the pixels came
                       from, which is the point. */
                    var pushedOne = false
                    senses.onGaze = { x, y ->
                        val call = if (x == null || y == null) "window.Face.look(null)"
                                   else "window.Face.look($x, $y)"
                        // Once, so "did anything actually reach her eyes" is answerable
                        // without a person standing in front of the camera to watch.
                        if (!pushedOne && x != null) {
                            pushedOne = true
                            android.util.Log.i("FacePanel", "first gaze pushed: $call")
                        }
                        post { evaluateJavascript(call, null) }
                    }
                }

                loadUrl(url)
            }
        },
        onRelease = { web ->
            // Watching must not outlive the panel that started it, and the page that owns
            // the marker is about to stop existing.
            senses?.onGaze = null
            senses?.release()
            bridge?.detach()
            bridge = null

            web.loadUrl("about:blank")
            web.destroy()
        },
    )
}
```
