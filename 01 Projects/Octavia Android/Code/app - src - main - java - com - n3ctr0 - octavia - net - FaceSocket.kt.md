---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\net\FaceSocket.kt
---

# app\src\main\java\com\n3ctr0\octavia\net\FaceSocket.kt

```kotlin
package com.n3ctr0.octavia.net

import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.Response
import okhttp3.WebSocket
import okhttp3.WebSocketListener
import okio.ByteString.Companion.toByteString
import org.json.JSONObject
import java.util.concurrent.TimeUnit
import kotlin.math.min

/**
 * The connection to Octavia, and the only thing in this app that talks to her.
 *
 * This is a *face*: it holds no key, runs no model, owns no conversation, and decides
 * nothing about what she says. Everything it knows arrived here as a JSON message. See
 * PROTOCOL.md in her repo — deliberately not copied into this one.
 *
 * **Messages are read defensively, never parsed into a fixed shape.** The protocol says a
 * face must ignore types and fields it does not recognise rather than failing, because new
 * ones may be added at any time within version 1. A generated parser with required fields
 * would turn her adding a field into this app crashing, which is exactly backwards — so
 * this uses `org.json` and reads what it wants with `opt*`.
 */
class FaceSocket(private val listener: Listener) {

    /** What the UI needs to know about the connection itself, as opposed to her.
     *
     *  `Refused` is separate from `Retrying` because the two need opposite behaviour:
     *  one is waiting for the world to improve, the other is waiting for a person. */
    enum class Link { Idle, Connecting, Up, Retrying, Refused }

    interface Listener {
        fun onLink(link: Link, detail: String?)

        /** One message from the host. Unknown types reach here too; ignoring them is the caller's job. */
        fun onMessage(message: JSONObject)

        /** One binary frame: raw PCM, in the format the last `hello` advertised. Per the
         *  protocol a binary frame is audio and nothing else, so there is nothing to parse. */
        fun onAudio(frame: ByteArray)
    }

    /** Message types this face cannot use, declined at connect time.
     *
     *  `viseme` is sent at phoneme rate and `level` about twenty times a second, and both
     *  exist to move a mouth and a meter this stage does not have. On a handset that is a
     *  battery rather than a feature — which is the whole reason `subscribe` exists. */
    private val skip = listOf("viseme", "level")

    private val client = OkHttpClient.Builder()
        // The host sends nothing at all while she is idle, so a read timeout would be a
        // disconnect on a working connection. Ping instead: it keeps a NAT or a dozing
        // radio from quietly dropping the socket, and it notices a dead peer.
        .readTimeout(0, TimeUnit.MILLISECONDS)
        .pingInterval(20, TimeUnit.SECONDS)
        .build()

    private var socket: WebSocket? = null
    private var attempt = 0

    @Volatile
    private var wanted = false

    /**
     * Which connection attempt is the current one.
     *
     * **Found by re-pairing against a running host, which is the only way it shows up.** A
     * retry is a sleeping thread holding the host, port and credential it was created with.
     * `disconnect()` clearing a single `wanted` flag does not reach it: the thread wakes
     * *after* `connect()` has set that flag back to true, sees a live client, and reconnects
     * with the **old** credential. Every re-pair then adds another lineage, so the stale
     * ones never die and the host logs a refusal every couple of seconds forever.
     *
     * Bumping a generation on every `connect`/`disconnect` invalidates all of them at once:
     * a callback or a retry belonging to a superseded attempt returns instead of acting.
     */
    @Volatile
    private var generation = 0

    @Volatile
    var protocol: Int = 0
        private set

    companion object {
        /**
         * The per-run token is 32 hex characters; the remote key is groups of five from an
         * alphabet with no `0`/`O` or `1`/`I` in it. They go in **different query
         * parameters** and the host treats them very differently — the token is refused
         * from anything but loopback — so telling them apart matters more than one text
         * field looks like it should.
         *
         * Shared rather than duplicated because the WebView panel needs exactly the same
         * decision when it loads her page, and getting it wrong there produced a refusal
         * that looked like the *host's* fault: over a USB tunnel everything is loopback and
         * a token works, so `?token=` was fine right up until the phone moved to WiFi.
         */
        fun credentialParam(credential: String): String {
            val trimmed = credential.trim()
            val isToken = trimmed.length == 32 &&
                trimmed.all { it.isDigit() || it in 'a'..'f' || it in 'A'..'F' }
            return if (isToken) "token=$trimmed" else "key=$trimmed"
        }
    }

    /** Whether to ask for her voice. Off unless this is the device meant to make noise —
     *  see the note on `subscribe` below. */
    @Volatile
    var wantAudio: Boolean = false
        private set

    fun connect(host: String, port: Int, credential: String, wantAudio: Boolean) {
        wanted = true
        attempt = 0
        this.wantAudio = wantAudio
        open(host, port, credential, ++generation)
    }

    private fun open(host: String, port: Int, credential: String, gen: Int) {
        if (!wanted || gen != generation) return

        // A superseded attempt may still hold an open socket. Close it before opening
        // another, or a re-pair leaves the previous connection attached to the host.
        socket?.cancel()

        listener.onLink(if (attempt == 0) Link.Connecting else Link.Retrying, null)

        val url = "ws://$host:$port/?${credentialParam(credential)}"
        val request = Request.Builder().url(url).build()

        socket = client.newWebSocket(request, object : WebSocketListener() {
            override fun onOpen(webSocket: WebSocket, response: Response) {
                if (gen != generation) { webSocket.cancel(); return }

                attempt = 0
                listener.onLink(Link.Up, null)

                /* Opt out of what this face cannot use, and opt *in* to what it wants.
                   Audio is the one stream the host will not send unasked, because it is a
                   physical output rather than a rendering hint — a face that draws her
                   mouth has not thereby claimed the right to make noise. Over a desk cable
                   this device is in the same room as her speakers, so asking is a decision.

                   `ready` goes second because it triggers `hello`, and the capabilities
                   should not arrive before the host has been told what to withhold. */
                val subscribe = JSONObject()
                    .put("type", "subscribe")
                    .put("skip", org.json.JSONArray(skip))
                if (wantAudio) subscribe.put("want", org.json.JSONArray(listOf("audio")))
                webSocket.send(subscribe.toString())
                webSocket.send(JSONObject().put("type", "ready").put("faceBuilt", true).toString())
            }

            override fun onMessage(webSocket: WebSocket, text: String) {
                if (gen != generation) return

                val parsed = try {
                    JSONObject(text)
                } catch (e: Exception) {
                    // A message this face cannot even parse is hers to fix, and telling her
                    // is more useful than dying: `faceError` reaches her log, which is the
                    // only console a remote face has.
                    webSocket.send(JSONObject().put("type", "faceError")
                        .put("text", "unparseable message: ${e.message}").toString())
                    return
                }
                listener.onMessage(parsed)
            }

            /** A binary frame is audio and nothing else — the protocol is explicit, so
             *  there is no type to check and no header to strip. */
            override fun onMessage(webSocket: WebSocket, bytes: okio.ByteString) {
                if (gen != generation) return
                listener.onAudio(bytes.toByteArray())
            }

            override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
                /* **A 401 is terminal, and this is not a theoretical distinction.** Her
                   per-run token is regenerated every time she starts, so restarting her
                   silently un-pairs the phone. Treating that as a transient fault meant
                   this client sat at its fifteen-second backoff cap retrying a credential
                   that could never work — for over an hour, once, burning radio and
                   writing a refusal into her log every fifteen seconds.

                   Nothing the client can wait for fixes a wrong secret. Only a person
                   can, so stop and ask one. `Retry` in the UI still forces another go,
                   for the case where the host was fixed rather than the phone. */
                if (response?.code == 401) {
                    wanted = false
                    generation++
                    listener.onLink(Link.Refused, "Wrong token or key — she may have restarted.")
                    return
                }

                retry(host, port, credential, t.message ?: "could not reach her", gen)
            }

            override fun onClosed(webSocket: WebSocket, code: Int, reason: String) {
                retry(host, port, credential, "She closed the connection.", gen)
            }
        })
    }

    /**
     * Back off, but not forever-long. A phone leaves and rejoins a network constantly, and
     * a client that has crept up to a five minute wait is one that looks broken for five
     * minutes after the wifi comes back.
     */
    private fun retry(host: String, port: Int, credential: String, why: String, gen: Int) {
        if (!wanted || gen != generation) return

        val delay = min(1000L shl min(attempt, 4), 15_000L)
        attempt++

        listener.onLink(Link.Retrying, why)

        Thread {
            try {
                Thread.sleep(delay)
            } catch (e: InterruptedException) {
                Thread.currentThread().interrupt()
                return@Thread
            }
            // Checked again on waking, not only before sleeping: the whole point is that
            // the world may have changed while this thread was asleep.
            open(host, port, credential, gen)
        }.apply { isDaemon = true }.start()
    }

    /** Type something at her. The only thing this stage can send that she acts on. */
    fun say(text: String) {
        // `text`, not `value` — `setKey`/`setVoice` and friends use `value`, `say` does not,
        // and getting it wrong is a message she accepts and silently does nothing with.
        socket?.send(JSONObject().put("type", "say").put("text", text).toString())
    }

    fun send(type: String) {
        socket?.send(JSONObject().put("type", type).toString())
    }

    /**
     * Take or release the floor.
     *
     * `true` begins a binary stream; `false` is the **end-of-utterance marker**, so her
     * voice detector never has to guess where the sentence stopped. Deliberately not
     * `listen`, which toggles her *own* microphone and has to keep working independently.
     *
     * She refuses this if another face already holds the floor, or if her ears are not open
     * yet — and answers with a `notice` rather than silence, which is why the caller does
     * not need to guess and this returns nothing.
     */
    fun talking(value: Boolean) {
        socket?.send(JSONObject().put("type", "talking").put("value", value).toString())
    }

    /** One frame of microphone PCM. A binary frame is audio in both directions. */
    fun sendAudio(frame: ByteArray) {
        socket?.send(frame.toByteString(0, frame.size))
    }

    fun noteProtocol(version: Int) {
        protocol = version
    }

    fun disconnect() {
        wanted = false
        generation++            // invalidates every sleeping retry, not just the live socket
        socket?.close(1000, null)
        socket = null
        listener.onLink(Link.Idle, null)
    }
}
```
