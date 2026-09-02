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
import androidx.camera.core.CameraSelector
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

        private val ANALYSIS = Size(320, 180)
    }

    private var provider: ProcessCameraProvider? = null
    private var owner: WatchLife? = null

    private var previous: FloatArray? = null
    private var lastAt = 0L

    /* Smoothed centroid in frame fractions, starting where a person usually is. Held
       across frames, which is why this is state rather than a local. */
    private var x = 0.5f
    private var y = 0.42f

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
        val front = selector == CameraSelector.DEFAULT_FRONT_CAMERA

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
        x = 0.5f; y = 0.42f; lastAt = 0L

        analysis.setAnalyzer(executor) { image ->
            try { onFrame(image, front, onGaze) } finally { image.close() }
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

    private fun onFrame(image: ImageProxy, front: Boolean, onGaze: (Float, Float) -> Unit) {
        val now = System.currentTimeMillis()
        if (now - lastAt < PERIOD_MS) return
        lastAt = now

        val grey = sample(image, front) ?: return
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
        if (sum <= ENOUGH) return

        x += (sx / sum / W - x) * 0.25f
        y += (sy / sum / H - y) * 0.25f

        /* Mirrored, like looking in a mirror: move to your left and her eyes go to your
           left. The vertical span is smaller because standing up should raise her gaze,
           not roll her eyes at the ceiling. Both spans are hers. */
        onGaze((0.5f - x) * 0.8f, (0.42f - y) * 0.55f)
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
    private fun sample(image: ImageProxy, front: Boolean): FloatArray? {
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

                // A front camera is shown mirrored, so match what the person sees.
                if (front) u = 1f - u

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
