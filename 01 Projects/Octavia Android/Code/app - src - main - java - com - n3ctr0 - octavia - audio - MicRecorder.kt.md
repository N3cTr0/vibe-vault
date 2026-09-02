---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\audio\MicRecorder.kt
---

# app\src\main\java\com\n3ctr0\octavia\audio\MicRecorder.kt

```kotlin
package com.n3ctr0.octavia.audio

import android.annotation.SuppressLint
import android.media.AudioFormat
import android.media.AudioRecord
import android.media.audiofx.AcousticEchoCanceler
import android.media.audiofx.NoiseSuppressor
import android.media.MediaRecorder
import android.util.Log

/**
 * This device's microphone, streamed to her while a button is held.
 *
 * **The format is fixed by contract, not negotiated:** 16 kHz, 16-bit, mono, little-endian.
 * That is what Silero and Whisper want, and the resampling burden sits here rather than in
 * the host — this handset has cycles to spare and she should not grow a resampler to save
 * it some work. `ENCODING_PCM_16BIT` on Android is native-endian, and every device that
 * runs this is little-endian, so the bytes go out as they come.
 *
 * Nothing here decides anything. It captures while told to and stops when told to; whether
 * she listens, and to whom, is hers to arbitrate — one face holds the floor at a time.
 */
class MicRecorder(private val onFrame: (ByteArray) -> Unit) {

    companion object {
        /** Silero's rate, and therefore hers. Not a preference. */
        const val SAMPLE_RATE = 16000

        private const val TAG = "MicRecorder"

        /** ~32 ms at 16 kHz, matching the host's own capture buffer. Small enough that
         *  releasing the button ends the utterance promptly, large enough not to wake the
         *  radio for every syllable. */
        private const val FRAME_BYTES = 1024
    }

    private var record: AudioRecord? = null
    private var canceller: AcousticEchoCanceler? = null
    private var suppressor: NoiseSuppressor? = null
    private var thread: Thread? = null

    @Volatile private var running = false

    @Volatile var bytesSent = 0L
        private set

    val active: Boolean get() = running

    /**
     * Begins capture. The caller must already hold `RECORD_AUDIO` — this returns false
     * rather than throwing if the device refuses, so a mic button can go back up instead of
     * the app dying in someone's hand.
     *
     * @param cancelEcho Ask the platform to cancel this device's own output.
     *
     * **Only for always-on listening, and it changes the audio source to get it.** The good
     * canceller on this hardware is Qualcomm's Fluence, in
     * `/vendor/lib/soundfx/libqcomvoiceprocessing.so`, and it attaches to
     * `VOICE_COMMUNICATION` — never to `VOICE_RECOGNITION`, which is otherwise the better
     * source for a recogniser and is what push-to-talk keeps. So the trade is made only
     * where it buys something: a held button has no echo problem worth paying telephony
     * gain and noise-shaping for.
     *
     * It is a bonus rather than a dependency. Gating on [VoicePlayer.audible] is what
     * actually stops her transcribing herself, on every device including one whose canceller
     * is a generic software effect. This buys **barge-in** — interrupting her mid-sentence —
     * where the hardware is good enough, and is absent rather than broken where it is not.
     */
    @SuppressLint("MissingPermission")
    fun start(cancelEcho: Boolean = false): Boolean {
        if (running) return true
        bytesSent = 0

        val min = AudioRecord.getMinBufferSize(
            SAMPLE_RATE, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT
        )
        if (min <= 0) { Log.w(TAG, "no buffer size at ${SAMPLE_RATE}Hz"); return false }

        val r = try {
            AudioRecord(
                // VOICE_RECOGNITION rather than MIC: it asks the platform for the
                // processing chain tuned for speech, and on most devices declines the
                // aggressive gain and noise shaping that flatter a voice memo and confuse
                // a recogniser. VOICE_COMMUNICATION only when the echo canceller is wanted,
                // because that is the only source the vendor effects attach to.
                if (cancelEcho) MediaRecorder.AudioSource.VOICE_COMMUNICATION
                else MediaRecorder.AudioSource.VOICE_RECOGNITION,
                SAMPLE_RATE,
                AudioFormat.CHANNEL_IN_MONO,
                AudioFormat.ENCODING_PCM_16BIT,
                min * 4
            )
        } catch (e: Exception) {
            Log.e(TAG, "could not open the microphone", e); return false
        }

        if (r.state != AudioRecord.STATE_INITIALIZED) {
            Log.w(TAG, "microphone did not initialise")
            try { r.release() } catch (e: Exception) { Log.w(TAG, "release: ${e.message}") }
            return false
        }

        if (cancelEcho) attachEffects(r.audioSessionId)

        record = r
        running = true

        try { r.startRecording() } catch (e: Exception) {
            Log.e(TAG, "startRecording failed", e); stop(); return false
        }

        thread = Thread({
            val buffer = ByteArray(FRAME_BYTES)
            while (running) {
                val read = try { r.read(buffer, 0, buffer.size) } catch (e: Exception) { -1 }
                if (read <= 0) continue

                bytesSent += read
                onFrame(if (read == buffer.size) buffer.copyOf() else buffer.copyOf(read))
            }
        }, "octavia-mic").apply { isDaemon = true; start() }

        Log.i(TAG, "microphone open at ${SAMPLE_RATE}Hz, buffer ${min * 4}")
        return true
    }

    /**
     * Attach the platform's echo canceller and noise suppressor, if this device has them.
     *
     * **Availability is not effectiveness.** `isAvailable()` was true on both handsets here —
     * Qualcomm Fluence on one, a generic NXP software effect on the other — and those are not
     * the same thing at all. Neither is required: every one of these failing leaves gating
     * doing the work it was always going to do.
     */
    private fun attachEffects(session: Int) {
        try {
            if (AcousticEchoCanceler.isAvailable()) {
                canceller = AcousticEchoCanceler.create(session)?.apply { enabled = true }
                Log.i(TAG, "echo canceller: ${if (canceller?.enabled == true) "on" else "refused"}")
            } else {
                Log.i(TAG, "no echo canceller on this device; gating alone")
            }

            if (NoiseSuppressor.isAvailable()) {
                suppressor = NoiseSuppressor.create(session)?.apply { enabled = true }
            }
        } catch (e: Exception) {
            // Never fatal. A microphone that records is worth more than one that cancels.
            Log.w(TAG, "could not attach the voice effects: ${e.message}")
        }
    }

    private fun releaseEffects() {
        try { canceller?.release() } catch (e: Exception) { Log.w(TAG, "canceller: ${e.message}") }
        try { suppressor?.release() } catch (e: Exception) { Log.w(TAG, "suppressor: ${e.message}") }
        canceller = null
        suppressor = null
    }

    fun stop() {
        if (!running && record == null) return
        releaseEffects()
        running = false
        thread?.interrupt()
        thread = null

        record?.let {
            try {
                if (it.state == AudioRecord.STATE_INITIALIZED) it.stop()
                it.release()
            } catch (e: Exception) {
                Log.w(TAG, "stop failed: ${e.message}")
            }
        }
        record = null

        // 16-bit mono, so two bytes a sample. Logged because "did anything actually leave
        // this device" is the first question when she says nothing back.
        if (bytesSent > 0) {
            Log.i(TAG, "sent $bytesSent bytes (~${bytesSent * 1000 / (SAMPLE_RATE * 2)}ms)")
        }
    }
}
```
