---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\web\EmbedderBridge.kt
---

# app\src\main\java\com\n3ctr0\octavia\web\EmbedderBridge.kt

```kotlin
package com.n3ctr0.octavia.web

import android.net.Uri
import android.util.Log
import android.webkit.WebView
import androidx.webkit.JavaScriptReplyProxy
import androidx.webkit.WebMessageCompat
import androidx.webkit.WebViewCompat
import androidx.webkit.WebViewFeature
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.launch
import org.json.JSONArray
import org.json.JSONObject

/**
 * Lends her page this device's microphone and camera.
 *
 * Her renderer looks for `window.OctaviaEmbedder` once, at load. If it is there, a face in a
 * room gets its microphone and watch buttons back and calls in here instead of sending
 * `listen` — which the host refuses from another room, correctly. If it is not there, the
 * page behaves exactly as it does in a browser tab and hides both. Nothing about any of this
 * crosses the socket.
 *
 * **Origin-restricted, and that is the whole reason for the dependency.**
 * `addJavascriptInterface` hands its object to *every* script in the page, and this page
 * arrives over plain `http://` on a LAN — so anything that could inject a script into it
 * could open the microphone. `addWebMessageListener` and `addDocumentStartJavaScript` both
 * take an allow-list, and neither runs for an origin that is not on it.
 *
 * **Injected at document start**, because the page reads the embedder once as it loads. A
 * bridge that arrived after the first script would leave the buttons hidden for the life of
 * the page and look exactly like the feature not working.
 */
class EmbedderBridge(
    private val webView: WebView,
    private val origin: String,
    private val scope: CoroutineScope,
    private val senses: List<String>,
    private val onTalking: suspend (Boolean) -> Unit,
    private val onWatch: suspend (Boolean) -> Unit,
) {

    companion object {
        private const val TAG = "EmbedderBridge"

        /** The injected transport. Underscored because it is plumbing: the page is written
         *  against `window.OctaviaEmbedder`, which the shim below builds on top of it. */
        private const val PORT = "__octaviaEmbedderPort"

        /** Whether this WebView can do it at all. Both features are provided by the WebView
         *  implementation rather than the OS, so a device can be current and still lack them
         *  — in which case nothing is injected and her page hides the buttons, which is
         *  today's behaviour rather than a failure. */
        val supported: Boolean
            get() = WebViewFeature.isFeatureSupported(WebViewFeature.WEB_MESSAGE_LISTENER) &&
                WebViewFeature.isFeatureSupported(WebViewFeature.DOCUMENT_START_SCRIPT)

        /** `http://host:port` — the exact origin the panel was loaded from. */
        fun originOf(url: String): String? {
            val uri = Uri.parse(url)
            val scheme = uri.scheme ?: return null
            val host = uri.host ?: return null
            val port = uri.port
            return if (port > 0) "$scheme://$host:$port" else "$scheme://$host"
        }
    }

    private var script: androidx.webkit.ScriptHandler? = null

    fun attach() {
        if (!supported) {
            Log.w(TAG, "this WebView has no message listener; her page will hide its buttons")
            return
        }

        val rules = setOf(origin)

        WebViewCompat.addWebMessageListener(webView, PORT, rules) { _, message, _, _, reply ->
            handle(message, reply)
        }

        script = WebViewCompat.addDocumentStartJavaScript(webView, shim(), rules)
        Log.i(TAG, "embedder offered to $origin, lending ${senses.joinToString(", ")}")
    }

    fun detach() {
        if (!supported) return
        script?.remove()
        script = null
        try { WebViewCompat.removeWebMessageListener(webView, PORT) } catch (e: Exception) {
            Log.w(TAG, "removing the listener: ${e.message}")
        }
    }

    private fun handle(message: WebMessageCompat, reply: JavaScriptReplyProxy) {
        val body = message.data ?: return
        val call = try { JSONObject(body) } catch (e: Exception) {
            Log.w(TAG, "unparseable call from the page: $body"); return
        }

        val id = call.optInt("id", -1)
        val method = call.optString("method")
        val arg = call.optBoolean("arg", false)

        when (method) {
            // Idempotent on purpose: her page releases the floor on pointerup, pointerleave,
            // pointercancel, blur, visibilitychange and its socket closing, so `false`
            // arrives far more often than `true` does.
            "talking" -> scope.launch {
                try {
                    onTalking(arg)
                    answer(reply, id, null)
                } catch (e: Exception) {
                    // Her ears not being started is the common one, and it cannot be fixed
                    // from this room — so the person pressing needs the sentence, not a
                    // button that quietly does nothing.
                    Log.w(TAG, "talking($arg) refused: ${e.message}")
                    answer(reply, id, e.message ?: "this device could not take the floor")
                }
            }

            "watch" -> scope.launch {
                try {
                    onWatch(arg)
                    answer(reply, id, null)
                } catch (e: Exception) {
                    // The page awaits this and leaves its marker down on a rejection, which
                    // is why the failure has to travel rather than be logged and dropped.
                    Log.w(TAG, "watch($arg) refused: ${e.message}")
                    answer(reply, id, e.message ?: "the camera would not open")
                }
            }

            else -> answer(reply, id, "no such call")
        }
    }

    private fun answer(reply: JavaScriptReplyProxy, id: Int, error: String?) {
        val out = JSONObject().put("id", id).put("ok", error == null)
        if (error != null) out.put("error", error)
        try { reply.postMessage(out.toString()) } catch (e: Exception) {
            Log.w(TAG, "could not answer the page: ${e.message}")
        }
    }

    /**
     * The shim the page actually talks to.
     *
     * `senses` is baked in rather than fetched, because the page reads it synchronously at
     * load and a promise there would read as "no senses". The port is looked up **per call**
     * rather than captured, so this does not depend on the injected object existing before
     * this script runs — only before the first press.
     */
    private fun shim(): String {
        val list = JSONArray(senses).toString()
        return """
        (function () {
          if (window.OctaviaEmbedder) return;

          var next = 1;
          var pending = new Map();
          var wired = false;

          function port() {
            var p = window.$PORT;
            if (p && !wired) {
              wired = true;
              p.onmessage = function (event) {
                var m;
                try { m = JSON.parse(event.data); } catch (e) { return; }
                var waiting = pending.get(m.id);
                if (!waiting) return;
                pending.delete(m.id);
                if (m.ok) waiting.resolve();
                else waiting.reject(new Error(m.error || 'this device refused'));
              };
            }
            return p;
          }

          function call(method, arg) {
            return new Promise(function (resolve, reject) {
              var p = port();
              if (!p) { reject(new Error('this device is not listening')); return; }
              var id = next++;
              pending.set(id, { resolve: resolve, reject: reject });
              p.postMessage(JSON.stringify({ id: id, method: method, arg: arg }));
            });
          }

          window.OctaviaEmbedder = {
            senses: $list,
            talking: function (held) { return call('talking', !!held); },
            watch: function (on) { return call('watch', !!on); }
          };
        })();
        """.trimIndent()
    }
}
```
