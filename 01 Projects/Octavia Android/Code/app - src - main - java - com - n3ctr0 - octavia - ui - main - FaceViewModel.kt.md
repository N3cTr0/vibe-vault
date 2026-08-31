---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\ui\main\FaceViewModel.kt
---

# app\src\main\java\com\n3ctr0\octavia\ui\main\FaceViewModel.kt

```kotlin
package com.n3ctr0.octavia.ui.main

import androidx.lifecycle.ViewModel
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.net.FaceSocket
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import org.json.JSONObject

/** One line in the transcript. */
data class Line(val who: String, val text: String, val kind: Kind) {
    enum class Kind { You, Her, Overheard, Notice }
}

data class FaceState(
    val link: FaceSocket.Link = FaceSocket.Link.Idle,
    val linkDetail: String? = null,
    /** Her state, from `state`: idle | listening | thinking | speaking. */
    val her: String = "idle",
    val caption: String = "",
    val captionTentative: Boolean = false,
    val lines: List<Line> = emptyList(),
    val model: String = "",
    val profile: String = "",
    val protocol: Int = 0,
    val needKey: Boolean = false,
)

/**
 * Holds what she has told us. Every field here arrived in a message; none of it is decided
 * on this device. If something in this class ever starts *choosing*, it belongs in the host.
 */
class FaceViewModel(private val settings: Settings) : ViewModel(), FaceSocket.Listener {

    private val _state = MutableStateFlow(FaceState())
    val state: StateFlow<FaceState> = _state.asStateFlow()

    private val socket = FaceSocket(this)

    fun connect() {
        if (!settings.configured) return
        socket.connect(settings.host, settings.port, settings.credential)
    }

    fun reconnect() {
        socket.disconnect()
        connect()
    }

    fun say(text: String) {
        val trimmed = text.trim()
        if (trimmed.isEmpty()) return

        /* Not shown locally. She answers `say` by sending `turn who:"you"` back before she
           starts thinking, so the echo is immediate and adding a local line would simply
           double it. Waiting for her is also the *correct* behaviour with more than one
           face attached: this way the transcript shows what was said at the desktop too,
           rather than only what was typed here. */
        socket.say(trimmed)
    }

    fun hush() = socket.send("hush")

    fun forget() = socket.send("forget")

    override fun onCleared() {
        socket.disconnect()
        super.onCleared()
    }

    override fun onLink(link: FaceSocket.Link, detail: String?) {
        _state.update { it.copy(link = link, linkDetail = detail) }
    }

    /**
     * **Unknown types fall through untouched, on purpose.** Within protocol version 1 she
     * may add message types and fields at any time, and a face is required to ignore what
     * it does not know rather than fail. So this is a `when` with no `else` branch that
     * complains — a new message type is not an error here, it is simply not yet used.
     */
    override fun onMessage(message: JSONObject) {
        when (message.optString("type")) {
            "hello" -> _state.update {
                it.copy(
                    model = message.optString("model"),
                    profile = message.optString("profile"),
                    protocol = message.optInt("protocol"),
                    her = message.optString("state", it.her),
                    needKey = !message.optBoolean("hasKey", true),
                )
            }

            "state" -> _state.update { it.copy(her = message.optString("value", "idle")) }

            "caption" -> _state.update {
                it.copy(
                    caption = message.optString("text"),
                    captionTentative = message.optBoolean("tentative", false),
                )
            }

            "turn" -> {
                val who = message.optString("who")
                append(
                    Line(
                        who = if (who == "you") "You" else "Octavia",
                        text = message.optString("text"),
                        kind = if (who == "you") Line.Kind.You else Line.Kind.Her,
                    )
                )
                // The caption is the line *being* said; a completed turn ends it.
                _state.update { it.copy(caption = "", captionTentative = false) }
            }

            // "She heard this and decided it was not addressed to her." The protocol says a
            // face should show it, faintly, and never swallow it — so it is a line here
            // rather than a log, with the reason attached.
            "overheard" -> append(
                Line(
                    who = "overheard",
                    text = message.optString("text") + "  —  " + message.optString("why"),
                    kind = Line.Kind.Overheard,
                )
            )

            "notice" -> append(Line("", message.optString("text"), Line.Kind.Notice))

            "needKey" -> _state.update { it.copy(needKey = true) }

            "cleared" -> _state.update { it.copy(lines = emptyList(), caption = "") }
        }
    }

    private fun append(line: Line) {
        if (line.text.isBlank()) return
        // A phone scrollback is not a log. She runs for days; this does not.
        _state.update { it.copy(lines = (it.lines + line).takeLast(200)) }
    }
}
```
