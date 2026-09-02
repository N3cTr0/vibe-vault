---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\androidTest\java\com\n3ctr0\octavia\camera\CameraStillTest.kt
---

# app\src\androidTest\java\com\n3ctr0\octavia\camera\CameraStillTest.kt

```kotlin
package com.n3ctr0.octavia.camera

import android.Manifest
import android.content.Context
import android.util.Base64
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import androidx.test.platform.app.InstrumentationRegistry
import kotlinx.coroutines.runBlocking
import kotlinx.coroutines.withTimeout
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith

/**
 * Her eyes, against a real camera.
 *
 * **Instrumented rather than a unit test, because there is nothing here worth testing
 * against a fake.** The interesting failures are all device-shaped: a camera provider that
 * never starts, a JPEG that arrives in a plane this code does not read, a lifecycle that
 * holds the device open after the shot. A mock would pass while every one of those was
 * broken.
 *
 * **This opens the camera and takes a picture.** Nothing is written to storage and nothing
 * leaves the device.
 */
@RunWith(AndroidJUnit4::class)
class CameraStillTest {

    private val context: Context get() = ApplicationProvider.getApplicationContext()

    /**
     * Granted here rather than by whoever runs the suite.
     *
     * The first run of this file failed three-for-three on a missing permission that had
     * been granted by hand and then wiped by the test APK's install — which looked exactly
     * like the camera code being broken. A test that arranges its own preconditions cannot
     * lie about that.
     */
    @Before
    fun grantTheCamera() {
        InstrumentationRegistry.getInstrumentation().uiAutomation
            .grantRuntimePermission(context.packageName, Manifest.permission.CAMERA)
    }

    /**
     * The contract `PROTOCOL.md` states: *always* answer, with an image or with an error.
     *
     * A device with no camera is a legal face and must fail rather than hang, so both
     * outcomes pass — what is being asserted is that it **answered at all**, inside a
     * timeout shorter than the twenty seconds the host waits.
     */
    @Test
    fun alwaysAnswers() = runBlocking {
        val shot = withTimeout(15_000) { CameraStill(context).take(Lens.Front) }
        assertTrue(
            "answered with neither an image nor a reason: $shot",
            shot is CameraStill.Shot.Image || shot is CameraStill.Shot.Failed,
        )
    }

    /**
     * When there is a picture it is really a JPEG, and it is really base64.
     *
     * The bytes are read straight out of `planes[0]` on the assumption that `ImageCapture`
     * with no output file hands back an encoded JPEG rather than YUV. That assumption is
     * the single most likely thing in this file to be quietly wrong — it would produce a
     * plausible-looking base64 string of raw planar garbage that only fails much later,
     * inside her brain, as "the image could not be read".
     */
    @Test
    fun aPictureIsAJpeg() = runBlocking {
        val shot = withTimeout(15_000) { CameraStill(context).take(Lens.Front) }
        if (shot !is CameraStill.Shot.Image) return@runBlocking   // no camera on this device

        val bytes = Base64.decode(shot.base64, Base64.NO_WRAP)
        assertTrue("suspiciously small for a photograph: ${bytes.size} bytes", bytes.size > 1024)

        // SOI, and the EOI at the tail. A truncated buffer passes the first and fails this.
        assertEquals("not a JPEG start", 0xFF.toByte(), bytes[0])
        assertEquals("not a JPEG start", 0xD8.toByte(), bytes[1])
        assertEquals("not a JPEG end", 0xFF.toByte(), bytes[bytes.size - 2])
        assertEquals("not a JPEG end", 0xD9.toByte(), bytes[bytes.size - 1])
    }

    /**
     * Twice in a row.
     *
     * **This is the test for "stopped in the same breath".** If the short-lived lifecycle
     * did not release the device, the second capture would find the camera still bound and
     * fail — and it would fail exactly the way a real second question does, minutes later,
     * rather than in the first one anybody tries.
     */
    @Test
    fun releasesTheCameraBetweenShots() = runBlocking {
        val first = withTimeout(15_000) { CameraStill(context).take(Lens.Front) }
        if (first !is CameraStill.Shot.Image) return@runBlocking

        val second = withTimeout(15_000) { CameraStill(context).take(Lens.Front) }
        assertTrue("the camera was not released after the first shot: $second",
            second is CameraStill.Shot.Image)
    }
}
```
