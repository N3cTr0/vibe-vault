---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\camera\CameraStill.kt
---

# app\src\main\java\com\n3ctr0\octavia\camera\CameraStill.kt

```kotlin
package com.n3ctr0.octavia.camera

import android.Manifest
import android.content.Context
import android.content.pm.PackageManager
import android.util.Base64
import android.util.Log
import android.util.Size
import androidx.camera.core.CameraSelector
import androidx.camera.core.ImageCapture
import androidx.camera.core.ImageCaptureException
import androidx.camera.core.ImageProxy
import androidx.camera.core.resolutionselector.ResolutionSelector
import androidx.camera.core.resolutionselector.ResolutionStrategy
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleOwner
import androidx.lifecycle.LifecycleRegistry
import androidx.core.content.ContextCompat
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlinx.coroutines.withContext
import kotlinx.coroutines.withTimeoutOrNull
import kotlinx.coroutines.Dispatchers
import kotlin.coroutines.resume
import kotlin.time.Duration.Companion.seconds

/**
 * One frame, and then the camera is off again.
 *
 * **The face owns the camera; the host owns the decision.** This opens nothing on its own —
 * it exists to answer `look`, which she sends only when the config allows it, the words
 * genuinely need eyes, and her brain has any. `PROTOCOL.md` makes four promises on a
 * conforming face's behalf and all four are kept here:
 *
 * - opened **only** on `look`, never on a timer and never speculatively
 * - **one** frame, and the camera stopped in the same breath
 * - the fact that it is live is shown, unmistakably, for as long as it is — that part is
 *   [com.n3ctr0.octavia.ui.main.FaceScreen]'s, driven by the flag this sets
 * - it **always answers**, with an image or with an error. Never silence.
 *
 * **No preview, and no `camera-view` dependency.** A viewfinder would be drawn over her
 * face, which is the wrong thing to be looking at, and `ImageCapture` binds perfectly well
 * without one.
 *
 * **The lifecycle is ours and it is deliberately short.** CameraX releases the device when
 * the owner it was bound to is destroyed, so a registry that goes to `RESUMED` for the
 * capture and `DESTROYED` immediately after is the most direct way to write "and stop it in
 * the same breath" — rather than binding to the activity and trusting a later `unbindAll`
 * to be reached on every path out.
 */
class CameraStill(private val context: Context) {

    sealed interface Shot {
        /** A base64 JPEG, ready for `sight`. */
        data class Image(val base64: String) : Shot

        /** Why there is no picture. Goes out as `sight { error }` — never as silence. */
        data class Failed(val why: String) : Shot
    }

    companion object {
        private const val TAG = "CameraStill"

        /** Big enough to read a room, small enough not to put four megabytes of base64
         *  through a phone's uplink. She is being asked what something looks like, not
         *  asked to count the stitching. */
        private val TARGET = Size(1280, 720)

        private const val JPEG_QUALITY = 80

        /**
         * A hard ceiling on the whole attempt, and it is not belt-and-braces.
         *
         * **CameraX does not fail when it cannot open the camera — it retries, forever.**
         * Without the permission the log reads `OPENING --> REOPENING`, `Attempting camera
         * re-open in 700ms`, and `takePicture` is simply never called back. That is silence,
         * which is the one answer `look` must never get: the host would spend its full
         * twenty seconds waiting and then answer blind anyway.
         *
         * Comfortably inside that twenty so the error still arrives as an error.
         */
        private val LIMIT = 12.seconds
    }

    /**
     * Take the picture.
     *
     * Returns rather than throws: every path out of here is an answer she can be given,
     * because the alternative is her waiting twenty seconds for a frame that is not coming.
     */
    suspend fun take(lens: Lens): Shot = withContext(Dispatchers.Main) {
        /* Checked here as well as by the caller, because the failure mode when it is
           missing is a hang rather than an exception — see LIMIT. A guard that turns
           silence into a sentence is worth duplicating. */
        if (ContextCompat.checkSelfPermission(context, Manifest.permission.CAMERA)
            != PackageManager.PERMISSION_GRANTED
        ) {
            Log.w(TAG, "asked to look without the camera permission")
            return@withContext Shot.Failed("the camera was not allowed on this device")
        }

        val owner = ShortLife()
        val executor = ContextCompat.getMainExecutor(context)

        val provider = try {
            ProcessCameraProvider.getInstance(context).let { future ->
                suspendCancellableCoroutine<ProcessCameraProvider?> { cont ->
                    future.addListener({
                        try {
                            cont.resume(future.get())
                        } catch (e: Exception) {
                            Log.e(TAG, "no camera provider", e)
                            cont.resume(null)
                        }
                    }, executor)
                }
            }
        } catch (e: Exception) {
            Log.e(TAG, "no camera provider", e)
            null
        } ?: return@withContext Shot.Failed("this device would not start its camera")

        /* The same lens her eyes follow — see `Lens`. Sharing the choice is the point: a
           still on one camera while the watcher tracks the other would have her answering
           about a room she is not looking at. */
        val selector = provider.selectorFor(lens)
            ?: return@withContext Shot.Failed("no camera on this device")

        val capture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
            .setJpegQuality(JPEG_QUALITY)
            .setResolutionSelector(
                ResolutionSelector.Builder()
                    .setResolutionStrategy(
                        ResolutionStrategy(TARGET, ResolutionStrategy.FALLBACK_RULE_CLOSEST_HIGHER_THEN_LOWER)
                    )
                    .build()
            )
            .build()

        try {
            provider.unbindAll()
            owner.resume()
            provider.bindToLifecycle(owner, selector, capture)

            withTimeoutOrNull(LIMIT) {
            // Explicit, because the two `resume` sites have different types and inference
            // otherwise pins this to whichever it resolves first.
            suspendCancellableCoroutine<Shot> { cont ->
                capture.takePicture(executor, object : ImageCapture.OnImageCapturedCallback() {
                    override fun onCaptureSuccess(image: ImageProxy) {
                        val shot = try {
                            // ImageCapture with no output file hands back JPEG already
                            // encoded, in one plane. There is nothing to compress here.
                            val buffer = image.planes[0].buffer
                            val bytes = ByteArray(buffer.remaining())
                            buffer.get(bytes)
                            Log.i(TAG, "one frame, ${bytes.size / 1024} KB")
                            Shot.Image(Base64.encodeToString(bytes, Base64.NO_WRAP))
                        } catch (e: Exception) {
                            Log.e(TAG, "could not read the frame", e)
                            Shot.Failed("the frame could not be read")
                        } finally {
                            image.close()
                        }
                        cont.resume(shot)
                    }

                    override fun onError(e: ImageCaptureException) {
                        Log.e(TAG, "capture failed", e)
                        cont.resume(Shot.Failed(e.message ?: "the camera failed"))
                    }
                })
            }
            } ?: run {
                Log.w(TAG, "the camera never answered within $LIMIT")
                Shot.Failed("this device's camera did not respond")
            }
        } catch (e: Exception) {
            Log.e(TAG, "could not open the camera", e)
            Shot.Failed("this device would not open its camera")
        } finally {
            // Both of these, in this order, on every path — including a cancellation.
            try { provider.unbindAll() } catch (e: Exception) { Log.w(TAG, "unbind: ${e.message}") }
            owner.finish()
        }
    }

    /**
     * A lifecycle that exists for one photograph.
     *
     * Destroying it is what releases the camera, so the "one frame and stop" promise is a
     * property of the object's lifetime rather than a call somebody has to remember.
     */
    private class ShortLife : LifecycleOwner {
        private val registry = LifecycleRegistry(this)
        override val lifecycle: Lifecycle get() = registry

        fun resume() {
            registry.currentState = Lifecycle.State.RESUMED
        }

        fun finish() {
            registry.currentState = Lifecycle.State.DESTROYED
        }
    }
}
```
