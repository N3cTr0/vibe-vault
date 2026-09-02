---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\camera\Lens.kt
---

# app\src\main\java\com\n3ctr0\octavia\camera\Lens.kt

```kotlin
package com.n3ctr0.octavia.camera

import androidx.camera.core.CameraSelector
import androidx.camera.lifecycle.ProcessCameraProvider

/**
 * Which camera this device lends her.
 *
 * **A client setting, not one of hers, and that is the architecture rather than convenience.**
 * Her page offers a camera list and shows *"Not known yet"* here, because it was loaded over
 * plain `http://` and a browser will not enumerate cameras for an insecure origin. The device
 * belongs to this app, so the choice does too — which is the same conclusion her Stage 15
 * item 3 reaches for `setMicrophone` and `setOutput`: what a client lends is the client's to
 * pick, and only *whether* she may look is hers.
 *
 * Both the still and the watcher read it, so `look` and her eyes cannot end up pointed at
 * different cameras.
 */
enum class Lens(val id: String, val label: String) {
    /** The protocol's reasoning for putting the camera in the face is that *"the face is
     *  where the person is"*. On a handset that is the selfie camera; on a wall tablet it is
     *  the one pointed at the room. */
    Front("front", "Front — the one looking at you"),
    Back("back", "Back — the one looking at the room");

    companion object {
        fun of(id: String): Lens = entries.firstOrNull { it.id == id } ?: Front
    }
}

/**
 * The selector for a preference, **falling back to whatever this device actually has**.
 *
 * A device with one camera is still a face, and a preference it cannot honour should not
 * become a refusal — she would be answered *"no camera on this device"* by a handset holding
 * one. Returns null only when there is genuinely nothing to open.
 */
fun ProcessCameraProvider.selectorFor(lens: Lens): CameraSelector? {
    val wanted = when (lens) {
        Lens.Front -> CameraSelector.DEFAULT_FRONT_CAMERA
        Lens.Back -> CameraSelector.DEFAULT_BACK_CAMERA
    }
    val other = when (lens) {
        Lens.Front -> CameraSelector.DEFAULT_BACK_CAMERA
        Lens.Back -> CameraSelector.DEFAULT_FRONT_CAMERA
    }

    return when {
        hasCamera(wanted) -> wanted
        hasCamera(other) -> other
        else -> null
    }
}
```
