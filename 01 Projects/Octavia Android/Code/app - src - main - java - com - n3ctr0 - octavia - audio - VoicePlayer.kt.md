---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\audio\VoicePlayer.kt
---

# app\src\main\java\com\n3ctr0\octavia\audio\VoicePlayer.kt

```kotlin
package com.n3ctr0.octavia.audio

import android.media.AudioAttributes
import android.media.AudioFormat
import android.media.AudioTrack
import android.util.Log
import java.util.concurrent.ArrayBlockingQueue
import java.util.concurrent.TimeUnit

/**
 * Her voice, out of this device's speaker.
 *
 * Raw PCM arrives as binary WebSocket frames in the format `hello` advertises — and the rate
 * belongs to the voice model, so it changes when the voice does. Nothing here assumes 22050;
 * [configure] is called from every `hello` and rebuilds the track when the format moves.
 *
 * **The frames are already in sync with the visemes.** The host tees them at the moment they
 * reach its own sound card, which is the same point the mouth shapes are read from. Nothing
 * on this side needs to align anything; it only has to not fall behind.
 */
class VoicePlayer {

    private companion object {
        const val TAG = "VoicePlayer"

        /** How long after the last written frame she is still assumed to be audible: the
         *  track's own buffer (~180 ms at her rates) plus a beat of room reverb. */
        const val TAIL_MS = 400L

        /** Matches the host's own bound. Old audio is worthless: a device that cannot keep
         *  up should hear a gap and catch up, rather than drift further behind for the rest
         *  of the utterance. The host drops the oldest for the same reason. */
        const val QUEUE_FRAMES = 16
    }

    private val queue = ArrayBlockingQueue<ByteArray>(QUEUE_FRAMES)

    private var track: AudioTrack? = null
    private var writer: Thread? = null

    @Volatile private var rate = 0
    @Volatile private var running = false

    /** True once a track exists, so the UI can say whether she can actually be heard here. */
    val ready: Boolean get() = track != null

    /**
     * Take the format from `hello`. Cheap and idempotent when nothing changed, because
     * `hello` arrives on every settings change and rebuilding the track mid-sentence would
     * be audible.
     */
    @Synchronized
    fun configure(available: Boolean, sampleRate: Int, bits: Int, channels: Int) {
        if (!available || sampleRate <= 0) { stop(); return }

        // 16-bit mono is the only shape the host produces today. Refusing anything else is
        // better than guessing at a format and emitting noise.
        if (bits != 16 || channels != 1) {
            Log.w(TAG, "unsupported format ${bits}-bit/${channels}ch; not playing")
            stop()
            return
        }

        if (track != null && rate == sampleRate) return

        stop()
        rate = sampleRate

        val min = AudioTrack.getMinBufferSize(
            sampleRate, AudioFormat.CHANNEL_OUT_MONO, AudioFormat.ENCODING_PCM_16BIT
        )
        if (min <= 0) { Log.w(TAG, "no buffer size for ${sampleRate}Hz"); return }

        track = try {
            AudioTrack.Builder()
                .setAudioAttributes(
                    AudioAttributes.Builder()
                        .setUsage(AudioAttributes.USAGE_MEDIA)
                        // She is a voice. This is what lets the system duck music for her
                        // and route her sensibly to a headset.
                        .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
                        .build()
                )
                .setAudioFormat(
                    AudioFormat.Builder()
                        .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
                        .setSampleRate(sampleRate)
                        .setChannelMask(AudioFormat.CHANNEL_OUT_MONO)
                        .build()
                )
                // Room for a few frames beyond the minimum: the minimum is the point at
                // which it underruns, not a comfortable place to sit on a phone whose
                // radio wakes up between packets.
                .setBufferSizeInBytes(min * 4)
                .setTransferMode(AudioTrack.MODE_STREAM)
                .build()
        } catch (e: Exception) {
            Log.e(TAG, "could not build the track", e)
            null
        } ?: return

        track?.play()
        running = true

        /* A dedicated writer, because `AudioTrack.write` blocks until the buffer has room.
           Doing that on the socket's reader thread would stall every other message she
           sends — captions and state would arrive late behind her own voice. */
        writer = Thread({
            while (running) {
                val frame = try {
                    queue.poll(200, TimeUnit.MILLISECONDS)
                } catch (e: InterruptedException) {
                    Thread.currentThread().interrupt(); break
                } ?: continue

                try {
                    track?.write(frame, 0, frame.size)
                    lastWrote = android.os.SystemClock.elapsedRealtime()
                } catch (e: Exception) {
                    Log.w(TAG, "write failed: ${e.message}")
                }
            }
        }, "octavia-voice").apply { isDaemon = true; start() }

        Log.i(TAG, "voice track at ${sampleRate}Hz, buffer ${min * 4}")
    }

    /** Bytes taken in since the last flush, and frames dropped for being too late.
     *
     *  Worth counting rather than inferring: `AudioTrack` reports `state:started` from the
     *  moment `play()` is called, whether or not a single byte has been written, so "the
     *  track is started" is not evidence that she was heard. Bytes are. */
    @Volatile var bytesThisUtterance = 0L; private set
    @Volatile var dropped = 0L; private set

    /** When a frame was last handed to the track. */
    @Volatile private var lastWrote = 0L

    /**
     * Whether her voice is coming out of this device's speaker **right now**.
     *
     * **This is the whole of Stage 14 item 6's defence, and it lives here for a reason.** The
     * host knows when it *sent* audio; it does not know when this handset's speaker emitted
     * it, nor when it stopped — the queue, the track's own buffer and the radio all sit in
     * between. That gap is exactly why her in-process `Mute()`/`Unmute()` does not survive a
     * network, and why always-on listening in a room needed solving here rather than there.
     *
     * This side knows precisely. The tail covers what has been written but not yet played —
     * the track buffer is about 180 ms — plus a moment of room reverb, because a microphone
     * hears the wall a beat after the speaker has finished.
     */
    val audible: Boolean
        get() = queue.isNotEmpty() ||
            (lastWrote != 0L && android.os.SystemClock.elapsedRealtime() - lastWrote < TAIL_MS)

    /** One binary frame from the socket. Never blocks the caller. */
    fun play(frame: ByteArray) {
        if (track == null) return
        bytesThisUtterance += frame.size

        // Bounded, dropping the oldest — the same rule the host applies at its end.
        if (!queue.offer(frame)) {
            queue.poll()
            dropped++
            queue.offer(frame)
        }
    }

    /**
     * She has stopped, so anything still held is a tail she has already finished.
     *
     * Required by the protocol on any `state` that is not `speaking`. Without it she carries
     * on talking here after going quiet in the room, which is worse than not being heard at
     * all — it is the sort of thing that makes a device feel haunted.
     */
    @Synchronized
    fun flush() {
        if (bytesThisUtterance > 0) {
            // 16-bit mono, so two bytes a sample.
            val ms = if (rate > 0) bytesThisUtterance * 1000 / (rate * 2) else 0
            Log.i(TAG, "utterance: $bytesThisUtterance bytes (~${ms}ms), $dropped frame(s) dropped")
            bytesThisUtterance = 0
            dropped = 0
        }

        queue.clear()
        val t = track ?: return
        try {
            t.pause()
            t.flush()
            t.play()
        } catch (e: Exception) {
            Log.w(TAG, "flush failed: ${e.message}")
        }
    }

    @Synchronized
    fun stop() {
        running = false
        writer?.interrupt()
        writer = null
        queue.clear()

        track?.let {
            try { it.pause(); it.flush(); it.stop(); it.release() }
            catch (e: Exception) { Log.w(TAG, "stop failed: ${e.message}") }
        }
        track = null
        rate = 0
    }
}
```
