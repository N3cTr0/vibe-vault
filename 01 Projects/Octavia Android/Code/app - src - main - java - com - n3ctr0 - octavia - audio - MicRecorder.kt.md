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

        /// Below this peak, nothing in the capture was loud enough to be anybody speaking.
        ///
        /// About 1% of full scale. A real room idles above this on room tone alone, so it
        /// separates *"the microphone is delivering silence"* — muted, misrouted, or a phone
        /// left in another room — from *"she did not understand you"*, which is a different
        /// problem with a different fix. It is deliberately not a voice detector: that is
        /// Silero's job, on her side, and duplicating it here would be a second opinion
        /// nobody asked for.
        private const val QUIET = 328
    }

    private var record: AudioRecord? = null
    private var canceller: AcousticEchoCanceler? = null
    private var suppressor: NoiseSuppressor? = null
    private var thread: Thread? = null

    @Volatile private var running = false

    @Volatile var bytesSent = 0L
        private set

    /**
     * The loudest sample seen since capture began, 0–32767.
     *
     * **"Bytes left this device" and "there was anything in them" are different questions,
     * and only the second one is the one being asked.** A microphone that opens, streams
     * ten seconds of digital silence and stops looks identical in the log to one that
     * worked — the byte count is the same either way. Every failure that matters here is
     * silent: a muted input, a phone in a different room from the person, a permission
     * granted but routed to a headset nobody is wearing.
     */
    @Volatile var peak = 0
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
        peak = 0

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
                measure(buffer, read)
                onFrame(if (read == buffer.size) buffer.copyOf() else buffer.copyOf(read))
            }
        }, "octavia-mic").apply { isDaemon = true; start() }

        Log.i(TAG, "microphone open at ${SAMPLE_RATE}Hz, buffer ${min * 4}")
        return true
    }

    /**
     * The loudest sample in one frame, kept as the loudest of the run.
     *
     * Little-endian 16-bit, which is the contract in both directions — the low byte needs
     * masking or Kotlin's sign extension turns every sample above 127 into a negative
     * number and the peak becomes noise about the encoding rather than about the room.
     */
    private fun measure(buffer: ByteArray, read: Int) {
        var loudest = peak

        var i = 0
        while (i + 1 < read) {
            val sample = ((buffer[i + 1].toInt() shl 8) or (buffer[i].toInt() and 0xff)).toShort().toInt()
            val size = if (sample == Short.MIN_VALUE.toInt()) Short.MAX_VALUE.toInt() else kotlin.math.abs(sample)
            if (size > loudest) loudest = size
            i += 2
        }

        peak = loudest
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
        // this device" is the first question when she says nothing back — and the peak
        // because it is the second one, and the byte count cannot answer it. Silence
        // streams exactly as many bytes as speech does.
        if (bytesSent > 0) {
            val loudest = peak * 100 / Short.MAX_VALUE.toInt()
            Log.i(TAG, "sent $bytesSent bytes (~${bytesSent * 1000 / (SAMPLE_RATE * 2)}ms), " +
                if (peak < QUIET) "heard nothing above $loudest% — the room was silent to it"
                else "peak $loudest%")
        }
    }
}
```
