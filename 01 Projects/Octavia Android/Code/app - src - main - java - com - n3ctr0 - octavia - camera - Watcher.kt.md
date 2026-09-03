---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\camera\Watcher.kt
---

# app\src\main\java\com\n3ctr0\octavia\camera\Watcher.kt

```kotlin
package com.n3ctr0.octavia.camera

import android.content.Context
import android.util.Log
import android.util.Size
import androidx.camera.core.ImageAnalysis
import androidx.camera.core.ImageProxy
import androidx.camera.core.resolutionselector.ResolutionSelector
import androidx.camera.core.resolutionselector.ResolutionStrategy
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.core.content.ContextCompat
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleOwner
import androidx.lifecycle.LifecycleRegistry
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlinx.coroutines.withContext
import kotlin.coroutines.resume
import kotlin.math.abs

/**
 * She looks at you, from this device.
 *
 * **A port of her `watch.js`, not a reimplementation of it.** Same 64×36 grid, same
 * thresholds, same smoothing, same mirroring, same rate — because it is the same feature and
 * it should feel identical whichever face you are standing in front of. The only reason it
 * exists twice is that `getUserMedia` does not run on a plain `http://<lan-ip>` origin, so
 * the page cannot get the pixels here and something has to hand them over.
 *
 * **One difference is deliberate, and it is named here because the last one was not.** Her
 * vertical reference is the constant 0.42; this one is measured over the first couple of
 * seconds of a watch, because a handset looks up at you from below and a webcam clipped to a
 * monitor does not. It is *fixed after that*, exactly as hers is — an earlier version let it
 * keep drifting, which quietly subtracted the person out of the signal and left her tracking
 * only movement. A header claiming "same smoothing" while the code did something else is
 * what made that hard to see, so the exception lives here now. See `onFrame`.
 *
 * **The port is smaller than the original.** `watch.js` builds a greyscale by averaging R, G
 * and B; `YUV_420_888`'s Y plane already *is* luma, so the conversion disappears.
 *
 * **Nothing leaves this class but two numbers.** No frame is kept, none is written, and
 * none reaches the socket — the same promise `watch.js` makes in its own header, and the
 * reason watching is deliberately absent from the protocol.
 *
 * The privacy marker is **not** drawn here. The page owns it and raises it when `watch(true)`
 * resolves; a second marker drawn natively would be a second thing to keep in step.
 */
class Watcher(private val context: Context) {

    companion object {
        private const val TAG = "Watcher"

        /** Her grid, and therefore ours. */
        private const val W = 64
        private const val H = 36

        /** How different a cell must be to count as movement rather than sensor noise, and
         *  how much total movement means a person rather than a flicker. Hers exactly. */
        private const val CELL_STEP = 0.06f
        private const val ENOUGH = 0.8f

        /** ~8 Hz. Eyes only need to keep up with someone shifting in a chair. */
        private const val PERIOD_MS = 120L

        /**
         * How many readings establish where "level" is on this device, at [PERIOD_MS] apart.
         *
         * Twenty is about two and a half seconds — long enough for the centroid's own
         * smoothing to have settled on the person rather than on the first frame's noise,
         * and short enough that nobody is standing still waiting for it. After this the
         * reference stops moving, which is the point: see `onFrame`.
         */
        private const val SETTLE_FRAMES = 20

        private val ANALYSIS = Size(320, 180)
    }

    private var provider: ProcessCameraProvider? = null
    private var owner: WatchLife? = null

    private var previous: FloatArray? = null
    private var lastAt = 0L

    /** When the motion reading was last written down. See `onFrame`. */
    private var lastSaid = 0L

    /* Smoothed centroid in frame fractions, starting where a person usually is. Held
       across frames, which is why this is state rather than a local. */
    private var x = 0.5f
    private var y = 0.42f

    /* Where level is on this device, settled over the first [SETTLE_FRAMES] readings of a
       watch and fixed from then on. Hers is the constant 0.42; this is the same thing
       measured, because a handset is not clipped to the top of a monitor. */
    private var centreY = 0.42f
    private var settling = 0

    val running: Boolean get() = owner != null

    /**
     * Open the camera and start following.
     *
     * **Throws if it cannot open.** The page awaits `watch(true)` and leaves its marker down
     * on a rejection, so failing quietly here would put her in a state where the camera is
     * off and the interface says otherwise — which is the one thing a privacy marker must
     * never do.
     */
    suspend fun start(lens: Lens, onGaze: (Float, Float) -> Unit) = withContext(Dispatchers.Main) {
        if (running) return@withContext

        val executor = ContextCompat.getMainExecutor(context)
        val cameras = ProcessCameraProvider.getInstance(context).let { future ->
            suspendCancellableCoroutine<ProcessCameraProvider?> { cont ->
                future.addListener({
                    cont.resume(try { future.get() } catch (e: Exception) { null })
                }, executor)
            }
        } ?: throw IllegalStateException("this device would not start its camera")

        val selector = cameras.selectorFor(lens)
            ?: throw IllegalStateException("no camera on this device")

        val analysis = ImageAnalysis.Builder()
            // Only the newest frame matters. A backlog of stale motion is worse than a gap.
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .setResolutionSelector(
                ResolutionSelector.Builder()
                    .setResolutionStrategy(
                        ResolutionStrategy(ANALYSIS, ResolutionStrategy.FALLBACK_RULE_CLOSEST_HIGHER_THEN_LOWER)
                    )
                    .build()
            )
            .build()

        previous = null
        x = 0.5f; y = 0.42f; centreY = 0.42f; settling = 0; lastAt = 0L

        analysis.setAnalyzer(executor) { image ->
            try { onFrame(image, onGaze) } finally { image.close() }
        }

        val life = WatchLife()
        try {
            life.resume()
            cameras.bindToLifecycle(life, selector, analysis)
        } catch (e: Exception) {
            life.finish()
            throw IllegalStateException("this device would not open its camera", e)
        }

        provider = cameras
        owner = life
        Log.i(TAG, "watching")
    }

    /**
     * Idempotent, because it is called from more paths than it is started from.
     *
     * **Main thread only**, and that is `LifecycleRegistry`'s rule rather than a preference:
     * it throws otherwise. Every caller in the app happened to already be on it, so this
     * was correct by luck until a test called it from somewhere else — which is exactly the
     * kind of thing that survives until the one path nobody exercised.
     */
    @androidx.annotation.MainThread
    fun stopNow() {
        val life = owner ?: return
        owner = null
        try { provider?.unbindAll() } catch (e: Exception) { Log.w(TAG, "unbind: ${e.message}") }
        life.finish()
        provider = null
        previous = null
        Log.i(TAG, "stopped watching")
    }

    /** The same thing from anywhere, for callers that are already suspending. */
    suspend fun stop() = withContext(Dispatchers.Main) { stopNow() }

    private fun onFrame(image: ImageProxy, onGaze: (Float, Float) -> Unit) {
        val now = System.currentTimeMillis()
        if (now - lastAt < PERIOD_MS) return
        lastAt = now

        val grey = sample(image) ?: return
        val last = previous
        previous = grey
        if (last == null) return

        var sum = 0f; var sx = 0f; var sy = 0f
        for (i in grey.indices) {
            val d = abs(grey[i] - last[i])
            if (d > CELL_STEP) {
                sum += d
                sx += (i % W) * d
                sy += (i / W) * d
            }
        }
        /* Throttled to once every two seconds, and it earns its place: "she is not following
           me" is otherwise unanswerable from here. `sum` says whether the camera saw motion
           at all, and the gaze says where it put her — two very different failures that look
           identical from in front of the screen. */
        if (now - lastSaid > 2000) {
            lastSaid = now
            Log.i(TAG, "motion %.2f (needs > %.2f), gaze %.2f, %.2f"
                .format(sum, ENOUGH, (0.5f - x) * 0.8f, (centreY - y) * 0.55f))
        }

        if (sum <= ENOUGH) return

        x += (sx / sum / W - x) * 0.25f
        y += (sy / sum / H - y) * 0.25f

        /* **Where "level" is: measured once, then held.**
         *
         * `watch.js` treats 0.42 down the frame as eye level, which is right for a webcam
         * clipped to a monitor and wrong for a handset — propped on a desk or held in a
         * hand, it looks up at you from below. Measured on the 11T Pro the centroid sat at
         * ~0.71 for a solid fifteen seconds, so she was told to look **down**, hard, the
         * entire time. She was tracking perfectly and staring at the floor, which from in
         * front of the screen is indistinguishable from not tracking at all.
         *
         * **The first fix for that was a drift, and it traded one invisible failure for
         * another.** A centre that keeps moving towards the person subtracts the person out
         * of the signal: within about six seconds `centreY - y` is back to zero however far
         * up or down they actually are, so a still person gets no vertical gaze at all and
         * only *movement* deflects her. Reported from in front of the screen as no
         * difference against the desktop client, which is exactly right — the desktop's
         * reference is a constant, so its deflection persists and this one evaporated.
         *
         * So the reference is **calibrated rather than assumed, and then frozen**: the first
         * couple of seconds of a watch settle it on wherever this device is looking from,
         * and after that it is as fixed as `watch.js`'s constant is. Sustained position
         * becomes sustained gaze again, which is the whole feature.
         *
         * Re-measured on every `start`, so standing the phone somewhere else and switching
         * watching off and on is the recalibration gesture, and needs no control.
         *
         * Horizontal is left alone: left and right of a phone are real directions, and the
         * measurements showed no bias there to correct. */
        if (settling < SETTLE_FRAMES) {
            // The centroid's own smoothing, so "level" is found about as fast as the
            // centroid itself settles rather than lagging a second behind it.
            centreY += (y - centreY) * 0.25f
            settling++
        }

        /* Mirrored, like looking in a mirror: move to your left and her eyes go to your
           left. The vertical span is smaller because standing up should raise her gaze,
           not roll her eyes at the ceiling. Both spans are hers. */
        onGaze((0.5f - x) * 0.8f, (centreY - y) * 0.55f)
    }

    /**
     * The Y plane, reduced to her grid.
     *
     * **Rotation and mirroring are handled here rather than by asking CameraX to rotate the
     * buffer**, which would copy every pixel to save arithmetic on four thousand of them.
     * A browser hands `watch.js` an already-upright, already-mirrored video; a sensor does
     * neither, and getting this wrong is not a crash — it is her looking the wrong way,
     * which is much harder to notice in a log.
     */
    private fun sample(image: ImageProxy): FloatArray? {
        val plane = image.planes.getOrNull(0) ?: return null
        val buffer = plane.buffer
        val rowStride = plane.rowStride
        val pixelStride = plane.pixelStride
        val sw = image.width
        val sh = image.height
        if (sw <= 0 || sh <= 0) return null

        val rotation = ((image.imageInfo.rotationDegrees % 360) + 360) % 360
        val out = FloatArray(W * H)

        for (row in 0 until H) {
            for (col in 0 until W) {
                // Fractions in the upright image the page would have been given.
                var u = (col + 0.5f) / W
                var v = (row + 0.5f) / H

                /* **No mirror here, and that is the fix for her following you backwards.**
                 *
                 * This used to flip a front camera's `u`, on the stated grounds that "a
                 * browser hands `watch.js` an already-mirrored video". It does not.
                 * `getUserMedia` delivers raw sensor pixels, and the mirroring people
                 * associate with a selfie preview is CSS on the element — `drawImage`
                 * never sees it. So `watch.js` reads an *unmirrored* frame and applies
                 * exactly one flip, in `(0.5 - x)` on the way out.
                 *
                 * Doing it here as well made two, and two cancel: move to your left and
                 * she looked to your right. Reported exactly that way.
                 *
                 * The flip stays where hers is — in `onGaze` — so there is one mirror on
                 * this path and one on hers, for either lens, and the two clients agree.
                 */

                // Undo the sensor rotation: map upright (u,v) back to sensor (su,sv).
                val su: Float
                val sv: Float
                when (rotation) {
                    90 -> { su = v; sv = 1f - u }
                    180 -> { su = 1f - u; sv = 1f - v }
                    270 -> { su = 1f - v; sv = u }
                    else -> { su = u; sv = v }
                }

                val px = (su * sw).toInt().coerceIn(0, sw - 1)
                val py = (sv * sh).toInt().coerceIn(0, sh - 1)
                val index = py * rowStride + px * pixelStride
                if (index < 0 || index >= buffer.limit()) return null

                // Y is luma already, so there is nothing to convert - only to scale.
                out[row * W + col] = (buffer.get(index).toInt() and 0xFF) / 255f
            }
        }
        return out
    }

    /** Alive exactly as long as she is watching, so stopping it is what frees the camera. */
    private class WatchLife : LifecycleOwner {
        private val registry = LifecycleRegistry(this)
        override val lifecycle: Lifecycle get() = registry
        fun resume() { registry.currentState = Lifecycle.State.RESUMED }
        fun finish() { registry.currentState = Lifecycle.State.DESTROYED }
    }
}
```
