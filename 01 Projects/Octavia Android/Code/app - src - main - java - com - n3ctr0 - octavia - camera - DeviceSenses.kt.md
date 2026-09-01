---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\camera\DeviceSenses.kt
---

# app\src\main\java\com\n3ctr0\octavia\camera\DeviceSenses.kt

```kotlin
package com.n3ctr0.octavia.camera

import android.Manifest
import android.content.Context
import android.content.pm.PackageManager
import android.util.Log
import androidx.core.content.ContextCompat
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

/**
 * What this device can lend her page, and the one place that owns the camera.
 *
 * **A still and a watch want the same hardware, and that is the trap this class exists for.**
 * `look` arrives from the host at any moment; `CameraStill` binds and then calls
 * `unbindAll()`, which would tear a running watcher out from under itself and leave her
 * staring at the last place she saw somebody — a failure that looks like her losing interest
 * rather than like a bug. So both go through here, one at a time, and a still politely puts
 * the watcher back afterwards.
 *
 * `PROTOCOL.md` states the rule under *Watching* now, so the next renderer inherits it
 * instead of rediscovering it.
 */
class DeviceSenses(
    private val context: Context,
    /** Raises the runtime permission prompt. Suspends until a person answers. */
    private val askForCamera: suspend () -> Boolean,
) {

    private val still = CameraStill(context)
    private val watcher = Watcher(context)

    /** One camera, one user at a time. */
    private val gate = Mutex()

    /**
     * Where her eyes should go, in the page. `null` hands her back her own saccades — the
     * page's `Face.look(null)` — and is sent whenever watching stops **for any reason**,
     * including a still interrupting it.
     */
    var onGaze: ((Float?, Float?) -> Unit)? = null

    val watching: Boolean get() = watcher.running

    /** What the embedder advertises to the page. Not the same list as `ready.senses`, which
     *  describes a *face* — the panel still reports nothing there, so `look` keeps going to
     *  the native client rather than to the page. */
    fun lends(): List<String> = buildList {
        add("mic")
        if (context.packageManager.hasSystemFeature(PackageManager.FEATURE_CAMERA_ANY)) add("camera")
    }

    /**
     * Take the still she asked for, putting watching back afterwards if it was on.
     *
     * Always returns rather than throwing: every path out is an answer she can be given,
     * because the alternative is her waiting twenty seconds for a frame that is not coming.
     */
    suspend fun takeStill(): CameraStill.Shot = gate.withLock {
        val wasWatching = watcher.running
        if (wasWatching) {
            // Her gaze goes home before the camera does, so she is not frozen mid-glance
            // for the second the shutter takes.
            watcher.stop()
            onGaze?.invoke(null, null)
        }

        try {
            still.take()
        } finally {
            if (wasWatching) {
                try {
                    watcher.start { x, y -> onGaze?.invoke(x, y) }
                } catch (e: Exception) {
                    /* The page raised its marker when `watch(true)` resolved and has no way
                       to hear that the camera has since gone. It will show her watching
                       while nothing is. The seam has no callback for this — worth raising
                       rather than papering over with a synthetic click on her button. */
                    Log.e(TAG, "could not resume watching after a still; the page's marker is now wrong", e)
                }
            }
        }
    }

    /**
     * Start or stop following.
     *
     * **Throws on failure, and that is the contract.** The page awaits `watch(true)` and
     * leaves its marker down if this rejects; swallowing the error would put the interface
     * and the camera into different states, which is the one thing a privacy marker must
     * never do.
     */
    suspend fun watch(on: Boolean) = gate.withLock {
        if (!on) {
            watcher.stop()
            onGaze?.invoke(null, null)
            return@withLock
        }

        if (watcher.running) return@withLock

        if (ContextCompat.checkSelfPermission(context, Manifest.permission.CAMERA)
            != PackageManager.PERMISSION_GRANTED
        ) {
            if (!askForCamera()) throw IllegalStateException("the camera was not allowed on this device")
        }

        watcher.start { x, y -> onGaze?.invoke(x, y) }
    }

    /**
     * Called when the panel goes away. Watching must not outlive the thing that started it.
     *
     * **Main thread only** — it is called from `onRelease`, which already is, and it stops
     * the camera synchronously so nothing is left running behind a composable that no
     * longer exists.
     */
    @androidx.annotation.MainThread
    fun release() {
        watcher.stopNow()
        onGaze?.invoke(null, null)
    }

    private companion object {
        const val TAG = "DeviceSenses"
    }
}
```
