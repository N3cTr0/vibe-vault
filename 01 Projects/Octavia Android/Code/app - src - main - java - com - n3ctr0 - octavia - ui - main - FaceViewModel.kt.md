---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\ui\main\FaceViewModel.kt
---

# app\src\main\java\com\n3ctr0\octavia\ui\main\FaceViewModel.kt

```kotlin
package com.n3ctr0.octavia.ui.main

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.n3ctr0.octavia.audio.MicRecorder
import com.n3ctr0.octavia.audio.VoicePlayer
import com.n3ctr0.octavia.camera.CameraStill
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.net.FaceSocket
import kotlinx.coroutines.launch
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
    /** Whether this device is streaming continuously — Stage 14 item 6. Separate from
     *  `talking`, which is a finger on a button. */
    val listening: Boolean = false,
    /** Whether the *host* can stream at all — false on the Windows voice, which cannot be
     *  teed. Reported so this face can say why it is silent rather than look broken. */
    val audioAvailable: Boolean = false,
    val audioRate: Int = 0,
    /** Whether this device asked for her voice, and has a track to play it on. */
    val playingHere: Boolean = false,
    /** Whether the *host* will take audio from a face at all. A microphone button that
     *  could only fail should not be offered. */
    val micAccepted: Boolean = false,
    /** Whether the button is down and this device holds her ears. */
    val talking: Boolean = false,
    /** Whether the camera is open **right now**. The protocol requires a face to show this
     *  unmistakably for as long as it is true, which is why it is state and not a local
     *  variable inside the capture. */
    val cameraLive: Boolean = false,
)

/**
 * Holds what she has told us. Every field here arrived in a message; none of it is decided
 * on this device. If something in this class ever starts *choosing*, it belongs in the host.
 */
class FaceViewModel(private val settings: Settings) : ViewModel(), FaceSocket.Listener {

    private val _state = MutableStateFlow(FaceState())
    val state: StateFlow<FaceState> = _state.asStateFlow()

    private val socket = FaceSocket(this)
    private val voice = VoicePlayer()

    /**
     * The microphone, and **the gate that stops her transcribing herself**.
     *
     * Frames go straight out as they are read — there is no buffering to do, because she is
     * the one deciding what counts as an utterance and holding audio here would only add
     * latency to a decision made at the other end. The one exception is the gate below.
     *
     * Stage 14 item 6. While her voice is coming out of this device's speaker, the frames
     * are dropped rather than sent — the microphone stays open, so there is nothing to
     * restart and no gap at the end of her sentence, but nothing she said travels back to
     * her ears as something somebody in the room said.
     *
     * **Only while listening continuously.** A held button is a person deliberately speaking
     * over her, and the first thing `startTalking` does is hush her — so gating a press would
     * throw away the very words it was pressed for.
     */
    private val mic = MicRecorder { frame ->
        if (listeningOn && !pressing && voice.audible) echoFramesDropped++
        else socket.sendAudio(frame)
    }

    /** True while this device is streaming continuously rather than on a held button. */
    @Volatile private var listeningOn = false

    /**
     * True while a finger is on the button.
     *
     * **The gate must let a press through even while she is speaking**, and leaving this out
     * cost a whole test round: holding the button during her answer and saying "yes" reached
     * her ears as *nothing voiced at all*, because the gate was busy suppressing her own
     * voice and took the words with it. A press is somebody deliberately talking over her —
     * `startTalking` hushes her for exactly that reason — so it is the one thing the gate
     * must never swallow.
     */
    @Volatile private var pressing = false

    /** Counted rather than inferred, and reported when listening stops: "she started
     *  answering herself" and "the gate is too keen" look the same from the outside. */
    @Volatile private var echoFramesDropped = 0L

    /**
     * How this face takes a picture, supplied by the activity.
     *
     * A lambda rather than a `CameraStill` held here, because taking one involves a
     * permission prompt and that has to be launched from an `Activity`. It also keeps a
     * `Context` out of a `ViewModel` that outlives rotations.
     *
     * Null means this face has no eyes, which is a legal face — `look` is then answered
     * with an error rather than with silence.
     */
    var eyes: (suspend () -> CameraStill.Shot)? = null

    fun connect() {
        if (!settings.configured) return

        /* Already attached, so leave it alone. Before "stay in the background" this could
           not happen — every start followed a release. Now returning to the app finds a live
           socket, and reconnecting it would take a new `FaceId`, re-announce, and re-enter
           the room for no reason a person could see. `reconnect()` disconnects first, so the
           Retry button and Set up still force a fresh one. */
        if (_state.value.link == FaceSocket.Link.Up ||
            _state.value.link == FaceSocket.Link.Connecting
        ) return

        socket.connect(
            settings.host, settings.port, settings.credential, settings.playAudio,
            room = settings.room,
            // Declared rather than assumed: this is the connection that owns the hardware.
            // The WebView panel is a separate face and answers for neither.
            senses = buildList {
                add("mic")
                if (eyes != null) add("camera")
            },
        )
    }

    fun reconnect() {
        socket.disconnect()
        connect()
    }

    /**
     * Let go of her while the app is not on screen.
     *
     * Stage 1 is a foreground client and nothing here needs to keep listening — so holding
     * a socket and a retry thread through a doze is spending radio and battery to learn
     * things nobody is looking at. Android 9 restricts background network anyway, so the
     * retries would mostly fail and back off to nothing useful.
     *
     * The day she needs to reach the phone *unprompted* this becomes a foreground service
     * with a notification, which is a deliberate, visible thing rather than a socket left
     * running by accident.
     */
    fun release() {
        stopTalking()
        // An open microphone must not outlive the screen, whether a finger or a switch
        // opened it. She closes her side when the face departs.
        stopListening()
        socket.disconnect()
        // The track holds a hardware buffer; leaving it open through a doze would keep the
        // audio path awake for a conversation nobody is listening to.
        voice.stop()
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

    /**
     * The button went down.
     *
     * **Pressing while she is speaking hushes her**, because that is what talking over
     * someone means — the alternative is recording over the top of her own voice and asking
     * her to transcribe the result.
     *
     * The floor is requested before the microphone opens, not after: she may refuse it —
     * another face is holding it, or her ears are not open yet — and she answers with a
     * `notice`, which arrives in the transcript. Opening the device first would mean
     * capturing audio with nowhere to go.
     */
    /**
     * Returns null when the floor was taken, or **why not**.
     *
     * It used to return silently, which was survivable while the only way in was a volume
     * key — you press a key, nothing happens, you shrug. It stopped being survivable when
     * her page put a microphone button back on the screen: a button that does nothing and
     * says nothing is the exact failure `micAccepted` exists to prevent.
     */
    fun startTalking(): String? {
        val now = _state.value
        if (now.link != FaceSocket.Link.Up) return "not connected to her"

        /* Her ears are not merely off — they have not been started, and **a room face
           cannot start them**. That takes `listen`, which belongs to the machine she runs
           on and has been refused from another room since her v0.24.0. So this is not a
           thing pressing harder will fix, and saying so is the only useful response. */
        if (!now.micAccepted) return "her ears are not open — they have to be started on the machine she runs on"

        if (now.her == "speaking") socket.send("hush")
        socket.talking(true)

        if (!mic.start()) {
            // The device refused. Give the floor straight back rather than holding ears we
            // cannot feed — she has no other way to know the microphone never opened.
            socket.talking(false)
            append(Line("", "This device would not open its microphone.", Line.Kind.Notice))
            return "this device would not open its microphone"
        }
        pressing = true
        _state.update { it.copy(talking = true) }
        return null
    }

    /**
     * Start or stop always-on listening in this room — Stage 14 item 6.
     *
     * Returns null when it was done, or **why not**, which her page shows to the person who
     * tapped. The host is told with `listen`, which from a room means *"transcribe what I am
     * sending"* rather than *"open the microphone on the machine you run on"*.
     *
     * **The microphone opens with echo cancellation asked for**, which push-to-talk does not:
     * see `MicRecorder.start`. Gating does the real work either way.
     */
    fun setListening(on: Boolean): String? {
        if (_state.value.link != FaceSocket.Link.Up) return "not connected to her"
        if (on == listeningOn) return null

        if (!on) {
            stopListening()
            return null
        }

        // A press already owns the microphone; leave it alone rather than fighting it.
        if (_state.value.talking) return "let go of the button first"

        if (!mic.start(cancelEcho = true)) {
            append(Line("", "This device would not open its microphone.", Line.Kind.Notice))
            return "this device would not open its microphone"
        }

        listeningOn = true
        echoFramesDropped = 0
        socket.send("listen")
        _state.update { it.copy(listening = true) }
        return null
    }

    private fun stopListening() {
        if (!listeningOn) return
        listeningOn = false

        // Told first, so she stops expecting audio before it stops arriving.
        socket.send("listen")
        if (!_state.value.talking) mic.stop()

        android.util.Log.i("FaceViewModel", "stopped listening; $echoFramesDropped frame(s) held back while she spoke")
        _state.update { it.copy(listening = false) }
    }

    /** The button came up, the app went away, or the link dropped. All three mean release. */
    fun stopTalking() {
        if (!_state.value.talking && !mic.active) return

        pressing = false

        // The stream outlives the press when the room was left listening: the press was a
        // louder claim laid on top of it, exactly as the host treats the floor.
        if (!listeningOn) mic.stop()

        socket.talking(false)
        _state.update { it.copy(talking = false) }
    }

    fun forget() = socket.send("forget")

    override fun onCleared() {
        socket.disconnect()
        voice.stop()
        super.onCleared()
    }

    override fun onLink(link: FaceSocket.Link, detail: String?) {
        if (link != FaceSocket.Link.Up) {
            voice.flush()
            // She treats a face that vanishes mid-stream as a release, but the microphone
            // is ours to close — leaving it open would keep recording into a dead socket.
            // That applies to always-on listening as much as to a held button, and the host
            // closes its side when the face departs.
            listeningOn = false
            mic.stop()
            _state.update { it.copy(talking = false, listening = false) }
        }
        _state.update { it.copy(link = link, linkDetail = detail) }
    }

    override fun onAudio(frame: ByteArray) = voice.play(frame)

    /**
     * **Unknown types fall through untouched, on purpose.** Within protocol version 1 she
     * may add message types and fields at any time, and a face is required to ignore what
     * it does not know rather than fail. So this is a `when` with no `else` branch that
     * complains — a new message type is not an error here, it is simply not yet used.
     */
    override fun onMessage(message: JSONObject) {
        when (message.optString("type")) {
            "hello" -> {
                /* The rate belongs to the voice model and changes when the voice does, so
                   it is re-read on every `hello` rather than cached once. `configure` is
                   idempotent when nothing moved — rebuilding the track mid-sentence would
                   be audible. */
                if (settings.playAudio) {
                    voice.configure(
                        available = message.optBoolean("audioAvailable", false),
                        sampleRate = message.optInt("audioRate"),
                        bits = message.optInt("audioBits"),
                        channels = message.optInt("audioChannels"),
                    )
                }

                _state.update {
                    it.copy(
                        model = message.optString("model"),
                        profile = message.optString("profile"),
                        protocol = message.optInt("protocol"),
                        her = message.optString("state", it.her),
                        needKey = !message.optBoolean("hasKey", true),
                        audioAvailable = message.optBoolean("audioAvailable", false),
                        audioRate = message.optInt("audioRate"),
                        playingHere = settings.playAudio && voice.ready,
                        micAccepted = message.optBoolean("micAccepted", false),
                    )
                }
            }

            "state" -> {
                val value = message.optString("value", "idle")

                // "A face must throw away buffered audio on any state that is not
                // `speaking`." Otherwise she carries on talking here after going quiet in
                // the room, which is worse than not being heard at all.
                if (value != "speaking") voice.flush()

                _state.update { it.copy(her = value) }
            }

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

            "look" -> look()
        }
    }

    /**
     * She asked to see.
     *
     * **This always answers.** Every branch ends in a `sight`, because the host is holding a
     * promise open for twenty seconds and silence spends all of it before she gives up and
     * answers blind anyway. A refused permission is an `error` — an answer, not a failure.
     *
     * `cameraLive` brackets the whole thing rather than only the shutter: the permission
     * prompt is part of "the camera is being opened", and the marker going up late would be
     * a marker that lies.
     */
    private fun look() {
        val takePicture = eyes
        if (takePicture == null) {
            socket.sight(null, "this face has no camera")
            return
        }

        viewModelScope.launch {
            _state.update { it.copy(cameraLive = true) }
            try {
                when (val shot = takePicture()) {
                    is CameraStill.Shot.Image -> socket.sight(shot.base64, null)
                    is CameraStill.Shot.Failed -> socket.sight(null, shot.why)
                }
            } catch (e: Exception) {
                socket.sight(null, e.message ?: "the camera failed")
            } finally {
                _state.update { it.copy(cameraLive = false) }
            }
        }
    }

    private fun append(line: Line) {
        if (line.text.isBlank()) return
        // A phone scrollback is not a log. She runs for days; this does not.
        _state.update { it.copy(lines = (it.lines + line).takeLast(200)) }
    }
}
```
