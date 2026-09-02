---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\MainActivity.kt
---

# app\src\main\java\com\n3ctr0\octavia\MainActivity.kt

```kotlin
package com.n3ctr0.octavia

import android.Manifest
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.content.pm.PackageManager
import android.os.Build
import android.os.Bundle
import android.util.Log
import android.view.KeyEvent
import androidx.activity.ComponentActivity
import androidx.activity.OnBackPressedCallback
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
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import com.n3ctr0.octavia.camera.DeviceSenses
import com.n3ctr0.octavia.camera.Lens
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.service.OctaviaService
import com.n3ctr0.octavia.theme.OctaviaTheme
import com.n3ctr0.octavia.ui.main.FaceScreen
import com.n3ctr0.octavia.ui.main.FaceViewModel
import kotlinx.coroutines.CancellableContinuation
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlin.coroutines.resume

class MainActivity : ComponentActivity() {

    private lateinit var model: FaceViewModel
    private lateinit var senses: DeviceSenses
    private lateinit var settings: Settings
    private lateinit var talk: suspend (Boolean) -> Unit
    private lateinit var askMic: ActivityResultLauncher<String>
    private lateinit var askCamera: ActivityResultLauncher<String>
    private lateinit var askNotifications: ActivityResultLauncher<String>

    /** The `look` that is waiting to hear whether it may open the camera. */
    private var cameraAsked: CancellableContinuation<Boolean>? = null

    /** The press that is waiting to hear whether it may open the microphone. */
    private var micAsked: CancellableContinuation<Boolean>? = null

    /** Whether the floor is currently held. `ACTION_DOWN` repeats while a key is held, so
     *  without this every repeat would take the floor again. */
    private var talking = false

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        settings = Settings(applicationContext)

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

        askMic = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
            micAsked?.resume(granted)
            micAsked = null
        }

        askNotifications = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
            if (!granted) Log.w("MainActivity", "notifications refused; she will stay connected but the way back is the app icon")
        }

        askCamera = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
            // Resumed rather than dropped: `look` is waiting on this, and a refusal is an
            // answer she needs as much as a photograph.
            cameraAsked?.let { it.resume(granted) }
            cameraAsked = null
        }

        /* What this device can lend her. Built here rather than in the ViewModel because a
           permission prompt has to be launched from an activity, and because a `Context` in
           a ViewModel that outlives rotations is a leak waiting to be written.
           `DeviceSenses` owns the camera outright so a `look` and a watch cannot fight over
           it — see the note in that class. */
        senses = DeviceSenses(
            applicationContext,
            askForCamera = { askForCamera() },
            lens = { Lens.of(settings.camera) },
        )

        model.eyes = { senses.takeStill() }

        /* Her page's microphone button, and the permission it needs.
           **Built here rather than in the screen** because pressing it may have to raise a
           prompt, and only an activity can. The volume key asked for the microphone from
           the first version; the button restored in 0.9.0 did not, so on a fresh install it
           could only ever fail — the exact "a control that could only fail" that
           `micAccepted` exists to prevent, reintroduced by a different door. */
        talk = { held ->
            if (!held) model.stopTalking()
            else {
                if (!askForMic()) throw IllegalStateException("the microphone was not allowed on this device")
                model.startTalking()?.let { why -> throw IllegalStateException(why) }
            }
        }

        /* Back stops being the way she dies.
           Pressing it used to finish the activity, which took the WebView, the ViewModel and
           the socket with it — there is no other exit from a full-screen app, so the *only*
           gesture for "I am done looking at this" was the one that ended her. On the desktop
           the equivalent gesture puts her in the tray.

           Only when she is meant to stay. With the setting off, back finishes as before, so
           nothing about the frugal device changes. */
        onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
            override fun handleOnBackPressed() {
                if (settings.stayConnected) {
                    moveTaskToBack(true)
                } else {
                    isEnabled = false
                    onBackPressedDispatcher.onBackPressed()
                }
            }
        })

        /* *Let her go*, from the notification. It reaches here rather than being handled in
           the service because the socket belongs to this activity — the service is a
           lifetime anchor and a way back, and does not own the connection. */
        ContextCompat.registerReceiver(
            this,
            releaseRequested,
            IntentFilter(OctaviaService.ACTION_RELEASE),
            ContextCompat.RECEIVER_NOT_EXPORTED,
        )

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
                    val state by model.state.collectAsStateWithLifecycle()

                    // No inset padding: the face is meant to reach the edges. What still
                    // needs to clear the bars — the dialog — handles its own spacing rather
                    // than the whole screen paying for it.
                    FaceScreen(
                        state, settings, model, senses, talk,
                        onWantNotifications = ::askForNotifications,
                        modifier = Modifier.fillMaxSize(),
                    )
                }
            }
        }
    }

    /** The notification's *Let her go*: drop the connection and close, which is the tray's
     *  Quit. Finishing is what releases the socket — `onStopOrDispose` runs on the way out
     *  and, with the service already stopping, takes the frugal branch. */
    private val releaseRequested = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            Log.i("MainActivity", "let go from the notification")
            model.release()
            finish()
        }
    }

    /**
     * Connecting and letting go live here rather than in a `LifecycleStartEffect`.
     *
     * **They were composition effects and that was a real bug, not a tidiness point.** An
     * effect only registers once composition has settled, so backgrounding the app before
     * then — which is exactly what happens on the first launch after an install, while the
     * WebView and dexopt are still going — left her with no service and no release: she was
     * simply dropped, and the notification that says she is still there never appeared.
     * `onStart`/`onStop` always run.
     *
     * `connect()` is idempotent, so returning to a live socket costs nothing.
     */
    override fun onStart() {
        super.onStart()
        model.connect()

        // Coming back is the tray icon being clicked: the notification has done its job and
        // should not sit there implying she is in the background while she is on screen.
        OctaviaService.stop(this)
    }

    override fun onStop() {
        super.onStop()

        /* Her eyes close whatever else happens. The marker that promises the camera is live
           is drawn by her page, so it leaves with the screen — and watching that outlived
           its marker would break a promise the protocol makes on every face's behalf.
           Before "stay in the background" this was implicit, because leaving destroyed the
           panel that owned the camera. */
        senses.release()

        /* **Turning the phone is not looking away.**
         *
         * A rotation destroys this activity and builds it again in place, so `onStop` runs
         * for something the person experiences as the screen turning round. Treating that as
         * leaving started the tray service and then stopped it milliseconds later from the
         * new `onStart` — before the service had reached `startForeground` — and Android kills
         * an app that breaks that contract outright:
         * `ForegroundServiceDidNotStartInTimeException`, which is what a rotation looked like
         * from the outside. It also dropped and reopened her socket every time the screen
         * turned, which is the very churn keeping it in a ViewModel exists to avoid.
         *
         * The ViewModel survives a configuration change, so there is nothing to do here: she
         * is still connected on the other side of it. */
        if (isChangingConfigurations) return

        // Her tray. Off, this is the behaviour every version up to 0.9.2 had.
        if (settings.stayConnected) OctaviaService.start(this) else model.release()
    }

    override fun onDestroy() {
        super.onDestroy()
        // Nothing is left holding the process open once the activity that owned the socket
        // is gone; a notification outliving it would be a way back to nothing.
        OctaviaService.stop(this)
        try { unregisterReceiver(releaseRequested) } catch (e: IllegalArgumentException) {
            Log.w("MainActivity", "receiver was already gone: ${e.message}")
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

                // The key has no way to show a refusal, so it says so in the log rather
                // than leaving a press that did nothing and explained nothing.
                val why = model.startTalking()
                if (why != null) { Log.w("MainActivity", "volume-up refused: $why"); return true }
                talking = true
            }

            KeyEvent.ACTION_UP -> {
                if (!talking) return true
                talking = false
                model.stopTalking()
            }
        }
        return true
    }

    /**
     * Ask for the camera, if it has not been granted already.
     *
     * **Asked on `look` rather than at startup**, the same as the microphone: a face that
     * opens with two permission dialogs in front of it has made a worse first impression
     * than one that asks when she reaches for something. The cost is that the prompt appears
     * unprompted by the *person* — she asked, not them — which is exactly why the live
     * marker goes up before this and not after.
     */
    /** The same, for the microphone. Both buttons ask when they are reached for. */
    private suspend fun askForMic(): Boolean {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO)
            == PackageManager.PERMISSION_GRANTED
        ) return true

        if (micAsked != null) return false

        return suspendCancellableCoroutine { cont ->
            micAsked = cont
            cont.invokeOnCancellation { micAsked = null }
            askMic.launch(Manifest.permission.RECORD_AUDIO)
        }
    }

    /**
     * Ask to post the notification that is the way back from the background.
     *
     * **A refusal is survivable and is not treated as one.** Android still runs the service
     * and still shows *some* indication that an app is running in the background, so she
     * stays connected either way — what is lost is the tap that brings her back and the
     * button that lets her go, and for that the app icon and the setting are both still
     * there. Nothing here blocks on the answer.
     */
    private fun askForNotifications() {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) return
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.POST_NOTIFICATIONS)
            == PackageManager.PERMISSION_GRANTED
        ) return

        askNotifications.launch(Manifest.permission.POST_NOTIFICATIONS)
    }

    private suspend fun askForCamera(): Boolean {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
            == PackageManager.PERMISSION_GRANTED
        ) return true

        // A second `look` arriving mid-prompt would strand the first. She only ever has one
        // outstanding, but this class should not depend on that staying true.
        if (cameraAsked != null) return false

        return suspendCancellableCoroutine { cont ->
            cameraAsked = cont
            cont.invokeOnCancellation { cameraAsked = null }
            askCamera.launch(Manifest.permission.CAMERA)
        }
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
