---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\androidTest\java\com\n3ctr0\octavia\camera\DeviceSensesTest.kt
---

# app\src\androidTest\java\com\n3ctr0\octavia\camera\DeviceSensesTest.kt

```kotlin
package com.n3ctr0.octavia.camera

import android.Manifest
import android.content.Context
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.test.platform.app.InstrumentationRegistry
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.withTimeout
import org.junit.After
import org.junit.Assert.assertFalse
import org.junit.Assert.assertNull
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith

/**
 * A still and a watch want the same camera.
 *
 * **This is acceptance 6 of item 10**, and it is the client's half: `look` can arrive at any
 * moment, and `CameraStill` calls `unbindAll()`, which would tear a running watcher out from
 * under itself. The failure is not a crash — it is her quietly stopping following you, some
 * time after a question that had nothing obviously to do with it. That is the kind of thing
 * nobody finds by using the app.
 *
 * The host-side and page-side halves are done; this proves the pause-and-resume.
 *
 * **These open the camera.** Nothing is stored and nothing leaves the device.
 */
@RunWith(AndroidJUnit4::class)
class DeviceSensesTest {

    private val context: Context get() = ApplicationProvider.getApplicationContext()

    /** Nothing here may raise a prompt: an instrumented test cannot answer one. */
    private val senses by lazy { DeviceSenses(context) { false } }

    @Before
    fun grantTheCamera() {
        InstrumentationRegistry.getInstrumentation().uiAutomation
            .grantRuntimePermission(context.packageName, Manifest.permission.CAMERA)
    }

    // Through the suspending path, not `release()`, which is main-thread-only because
    // `LifecycleRegistry` is — and a test runs on neither.
    @After
    fun stop() = runBlocking { senses.watch(false) }

    @Test
    fun aStillPausesAndResumesWatching() = runBlocking {
        withTimeout(20_000) { senses.watch(true) }
        assertTrue("watching did not start", senses.watching)

        val shot = withTimeout(20_000) { senses.takeStill() }
        assertTrue("the still failed while watching: $shot", shot is CameraStill.Shot.Image)

        // The point of the whole class. Without the pause this is false, and it is false
        // silently.
        assertTrue("watching did not resume after the still", senses.watching)

        withTimeout(20_000) { senses.watch(false) }
        assertFalse("watching did not stop", senses.watching)
    }

    /**
     * Her gaze goes home before the camera does.
     *
     * A still takes about a second. Without this she holds whatever direction she was last
     * pointed in for the whole of it, which reads as her staring rather than glancing.
     */
    @Test
    fun theGazeIsReleasedAroundAStill() = runBlocking {
        val gazes = mutableListOf<Pair<Float?, Float?>>()
        senses.onGaze = { x, y -> gazes += x to y }

        withTimeout(20_000) { senses.watch(true) }
        withTimeout(20_000) { senses.takeStill() }

        assertTrue(
            "nothing told the page to release her gaze",
            gazes.any { it.first == null && it.second == null },
        )
        withTimeout(20_000) { senses.watch(false) }
    }

    /** Idempotent, because her page releases the floor and the watch from more paths than
     *  it starts them from — pointerup, pointerleave, pointercancel, blur, visibilitychange
     *  and its socket closing. */
    @Test
    fun stoppingTwiceIsHarmless() = runBlocking {
        withTimeout(20_000) { senses.watch(true) }
        withTimeout(20_000) { senses.watch(false) }
        withTimeout(20_000) { senses.watch(false) }
        assertFalse(senses.watching)
    }

    /** Starting twice must not open a second camera or lose the first. */
    @Test
    fun startingTwiceIsHarmless() = runBlocking {
        withTimeout(20_000) { senses.watch(true) }
        withTimeout(20_000) { senses.watch(true) }
        assertTrue(senses.watching)

        val shot = withTimeout(20_000) { senses.takeStill() }
        assertTrue("the camera was lost by starting twice: $shot", shot is CameraStill.Shot.Image)
        withTimeout(20_000) { senses.watch(false) }
    }

    /** What this device offers the page. Separate from `ready.senses`, which the panel
     *  reports empty so that `look` keeps going to the native client. */
    @Test
    fun itLendsAMicrophoneAndACamera() {
        val lent = senses.lends()
        assertTrue("no microphone offered", lent.contains("mic"))
        assertTrue("no camera offered on a device that has one", lent.contains("camera"))
        assertNull("nothing else should be advertised", lent.firstOrNull { it !in setOf("mic", "camera") })
    }
}
```
