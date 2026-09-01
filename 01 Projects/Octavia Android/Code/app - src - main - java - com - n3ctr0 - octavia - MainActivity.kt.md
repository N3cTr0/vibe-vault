---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\MainActivity.kt
---

# app\src\main\java\com\n3ctr0\octavia\MainActivity.kt

```kotlin
package com.n3ctr0.octavia

import android.Manifest
import android.content.pm.PackageManager
import android.os.Bundle
import android.view.KeyEvent
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.activity.result.ActivityResultLauncher
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.core.content.ContextCompat
import androidx.core.view.WindowCompat
import androidx.core.view.WindowInsetsCompat
import androidx.core.view.WindowInsetsControllerCompat
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.compose.LifecycleStartEffect
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.theme.OctaviaTheme
import com.n3ctr0.octavia.ui.main.FaceScreen
import com.n3ctr0.octavia.ui.main.FaceViewModel

class MainActivity : ComponentActivity() {

    private lateinit var model: FaceViewModel
    private lateinit var askMic: ActivityResultLauncher<String>

    /** Whether the floor is currently held. `ACTION_DOWN` repeats while a key is held, so
     *  without this every repeat would take the floor again. */
    private var talking = false

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val settings = Settings(applicationContext)

        /* The socket outlives a rotation, which is the whole reason it lives in a ViewModel
           rather than in a composable: reconnecting every time the screen turns would be
           visible, slow, and wrong. It is built here rather than inside `setContent` so the
           volume key can reach it — the store is the activity's either way, so this is the
           same instance a composable would have got. */
        model = ViewModelProvider(this, object : ViewModelProvider.Factory {
            @Suppress("UNCHECKED_CAST")
            override fun <T : ViewModel> create(modelClass: Class<T>): T =
                FaceViewModel(settings) as T
        })[FaceViewModel::class.java]

        askMic = registerForActivityResult(ActivityResultContracts.RequestPermission()) {}

        enableEdgeToEdge()

        /* Her page gets the whole panel, including behind the status and navigation bars.
           Transient-by-swipe rather than permanently hidden: the bars come back on a swipe
           from the edge and go away again on their own, so nothing is trapped. */
        WindowCompat.getInsetsController(window, window.decorView).apply {
            systemBarsBehavior = WindowInsetsControllerCompat.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE
            hide(WindowInsetsCompat.Type.systemBars())
        }

        setContent {
            OctaviaTheme {
                Surface(Modifier.fillMaxSize(), color = MaterialTheme.colorScheme.background) {

                    /* Connected while she is on screen and let go when she is not, rather
                       than connected for the life of the ViewModel. A rotation does not
                       pass through STOP, so the socket still survives one; leaving the app
                       does, and then nothing is spent on a conversation nobody is reading. */
                    LifecycleStartEffect(Unit) {
                        model.connect()
                        onStopOrDispose { model.release() }
                    }

                    val state by model.state.collectAsStateWithLifecycle()

                    // No inset padding: the face is meant to reach the edges. What still
                    // needs to clear the bars — the dialog — handles its own spacing rather
                    // than the whole screen paying for it.
                    FaceScreen(state, settings, model, Modifier.fillMaxSize())
                }
            }
        }
    }

    /**
     * Hold volume-up to talk.
     *
     * **Push-to-talk with no pixels.** The screen is her page and nothing else, so there is
     * nowhere to put a button that would not sit on top of her own controls — and a hardware
     * key is the more natural gesture for a walkie-talkie anyway.
     *
     * Push-to-talk rather than always-on, and it is not only about echo: a held key has
     * already answered *"was that addressed to me?"*, so her attention gate does not apply to
     * this stream at all, and one talker at a time means one transcription. The release is
     * the end of the utterance, so she never has to guess where the sentence stopped.
     *
     * **The cost is that volume-up no longer changes the volume while she is on screen.**
     * Volume-down still does, and its slider can be dragged back up, so nothing is one-way.
     */
    override fun dispatchKeyEvent(event: KeyEvent): Boolean {
        if (event.keyCode != KeyEvent.KEYCODE_VOLUME_UP) return super.dispatchKeyEvent(event)

        when (event.action) {
            KeyEvent.ACTION_DOWN -> {
                if (talking) return true    // auto-repeat while the key is held down

                if (ContextCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO)
                    != PackageManager.PERMISSION_GRANTED
                ) {
                    // Asked on the first press rather than at startup: a face that opens
                    // with a permission dialog in front of it is a worse first impression
                    // than one that asks when you reach for the microphone.
                    askMic.launch(Manifest.permission.RECORD_AUDIO)
                    return true
                }

                talking = true
                model.startTalking()
            }

            KeyEvent.ACTION_UP -> {
                if (!talking) return true
                talking = false
                model.stopTalking()
            }
        }
        return true
    }

    /** A key held while the app goes away would otherwise leave her holding the floor. */
    override fun onPause() {
        super.onPause()
        if (talking) {
            talking = false
            model.stopTalking()
        }
    }
}
```
