---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\ui\main\FaceScreen.kt
---

# app\src\main\java\com\n3ctr0\octavia\ui\main\FaceScreen.kt

```kotlin
package com.n3ctr0.octavia.ui.main

import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.gestures.detectTapGestures
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.BoxScope
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.RadioButton
import androidx.compose.material3.Switch
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.input.pointer.pointerInput
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.n3ctr0.octavia.camera.DeviceSenses
import com.n3ctr0.octavia.camera.Lens
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.net.FaceSocket

/**
 * Her face, and nothing else.
 *
 * **The app is a window onto her page, not a second interface to her.** Everything this
 * screen used to draw — a title, a connection line, a transcript, a text box and a Send
 * button — was a *reimplementation* of controls her own page already has, stacked above and
 * below it. The result was her rendered into the top third of the screen with a dead band
 * underneath where an empty transcript was holding `weight(1f)`.
 *
 * So the native chrome is gone and the page is given the whole display. What remains native
 * is what a browser cannot do on a plain `http://` origin: the socket, the microphone and
 * her voice.
 *
 * Push-to-talk moved to the volume key here for the same reason, and **moved back** in
 * v0.12.2: v0.9.0 returned her own buttons and v0.12.0 taught the on-screen one to tap and
 * hold, so the key was a duplicate that cost a volume control. The page's own button is the
 * only push-to-talk now.
 *
 * **Set up survives as a long press in the bottom-right corner**, because deleting it
 * outright would be a trap: a key that stops working would leave an app that cannot be
 * repaired from the device at all, which is exactly the state this handset was in on
 * 09/01/2026, and it took `adb` to get out of.
 */
@Composable
fun FaceScreen(
    state: FaceState,
    settings: Settings,
    viewModel: FaceViewModel,
    senses: DeviceSenses?,
    /** Her page's microphone button. Suspends because it may raise a permission prompt, and
     *  throws so a refusal reaches the page rather than being swallowed. */
    onTalking: suspend (Boolean) -> Unit,
    /** Asks for the notification permission. Only an activity can, and the notification is
     *  the only way back to her once she is in the background. */
    onWantNotifications: () -> Unit,
    modifier: Modifier = Modifier,
) {

    var settingsOpen by remember { mutableStateOf(!settings.configured) }
    var faceShown by remember { mutableStateOf(settings.showFace) }

    Box(modifier.fillMaxSize()) {

        if (faceShown && state.link == FaceSocket.Link.Up) {
            // No padding, no corner radius, no weight. The page is the screen.
            FacePanel(
                config = settings,
                senses = senses,
                /* Her microphone button, on this device's microphone. The floor is held by
                   a FaceId and audio is only accepted from the holder, so this drives the
                   *native* socket rather than the panel's — the two are different faces to
                   her, and a press in the panel would have her dropping our audio.

                   Supplied by the activity because it may have to raise a permission
                   prompt, and refusals are thrown rather than swallowed: the page is what
                   the person is looking at, and the only thing that can tell them why the
                   button they just pressed did nothing. */
                onTalking = onTalking,
                /* Her microphone button, tapped rather than held: leave this device
                   listening. The refusal travels, because a tap that silently does nothing
                   is the failure `micAccepted` exists to prevent, in a new place. */
                onListening = { on ->
                    viewModel.setListening(on)?.let { why -> throw IllegalStateException(why) }
                },
                /* Changed from inside her Settings panel. Only the four that describe the
                   connection reopen it; a camera choice is read at every use and a
                   background switch at every stop, so neither needs anything restarted. */
                onChanged = { reconnect -> if (reconnect) viewModel.reconnect() },
                modifier = Modifier.fillMaxSize(),
            )
        } else {
            // There is no page to show yet, and a black rectangle explains nothing.
            Disconnected(state, viewModel, onSetUp = { settingsOpen = true })
        }

        if (state.cameraLive) CameraMarker()

        /* The way back to Set up. Bottom-right because that corner of her page is empty —
           her own controls sit top-left, top-right and bottom-left — so this steals touches
           from nothing. It has no appearance on purpose: it is a recovery path, not a
           feature, and drawing it would put native chrome back on the screen. */
        Box(
            Modifier
                .align(Alignment.BottomEnd)
                .size(56.dp)
                .pointerInput(Unit) {
                    detectTapGestures(onLongPress = { settingsOpen = true })
                }
        )
    }

    if (settingsOpen) {
        SetupDialog(
            settings = settings,
            onWantNotifications = onWantNotifications,
            onDismiss = { settingsOpen = false },
            onSave = { settingsOpen = false; faceShown = settings.showFace; viewModel.reconnect() },
        )
    }
}

/**
 * The camera is open.
 *
 * **This is a promise the protocol makes on a face's behalf**, in those words: a conforming
 * face *"shows that the camera is live, unmistakably, for as long as it is"*. So it is a
 * border around the entire screen and a label, not a discreet dot — the whole point is that
 * it cannot be mistaken for part of her page, and that nobody has to be looking at one
 * particular corner to catch it.
 *
 * It covers the permission prompt as well as the shutter, because being asked for the
 * camera is part of the camera being opened.
 */
@Composable
private fun BoxScope.CameraMarker() {
    Box(
        Modifier
            .matchParentSize()
            .border(6.dp, Color(0xFFDC2626))
    )
    Box(
        Modifier
            .align(Alignment.TopCenter)
            .padding(top = 28.dp)
            .clip(RoundedCornerShape(20.dp))
            .background(Color(0xFFDC2626))
            .padding(horizontal = 18.dp, vertical = 8.dp)
    ) {
        Text(
            "CAMERA ON",
            color = Color.White,
            fontSize = 13.sp,
            fontWeight = FontWeight.Bold,
            fontFamily = FontFamily.Monospace,
        )
    }
}

/**
 * What is shown when her page cannot be. "Nothing is happening" is the worst thing a face
 * can show, and that is truer now that there is no status line above her.
 */
@Composable
private fun Disconnected(state: FaceState, viewModel: FaceViewModel, onSetUp: () -> Unit) {
    val text = when (state.link) {
        FaceSocket.Link.Idle -> "Not connected. Set up her address and key."
        FaceSocket.Link.Connecting -> "Reaching her…"
        // Deliberately not "retrying": nothing is going to happen until someone acts.
        FaceSocket.Link.Refused -> state.linkDetail ?: "Refused. Set up again."
        FaceSocket.Link.Retrying -> state.linkDetail?.let { "$it Retrying…" } ?: "Retrying…"
        // Connected — so the only way to be here is that she was asked not to draw.
        FaceSocket.Link.Up -> "Her face is switched off in Set up."
    }

    Column(
        Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        Text(
            text,
            fontSize = 14.sp,
            textAlign = TextAlign.Center,
            color = if (state.link == FaceSocket.Link.Refused) MaterialTheme.colorScheme.error
                    else MaterialTheme.colorScheme.onSurface.copy(alpha = 0.7f),
        )

        if (state.needKey) {
            Spacer(Modifier.padding(vertical = 6.dp))
            Text(
                "She has no API key, so she cannot think. That is fixed on the host, not here.",
                fontSize = 12.sp,
                textAlign = TextAlign.Center,
                color = MaterialTheme.colorScheme.error,
            )
        }

        Spacer(Modifier.padding(vertical = 10.dp))

        Row(verticalAlignment = Alignment.CenterVertically) {
            Button(onClick = { viewModel.reconnect() }) { Text("Retry") }
            Spacer(Modifier.width(8.dp))
            TextButton(onClick = onSetUp) { Text("Set up") }
        }
    }
}

@Composable
private fun SetupDialog(
    settings: Settings,
    onWantNotifications: () -> Unit,
    onDismiss: () -> Unit,
    onSave: () -> Unit,
) {
    var host by remember { mutableStateOf(settings.host) }
    var port by remember { mutableStateOf(settings.port.toString()) }
    var credential by remember { mutableStateOf(settings.credential) }
    var room by remember { mutableStateOf(settings.room) }
    var camera by remember { mutableStateOf(Lens.of(settings.camera)) }
    var playAudio by remember { mutableStateOf(settings.playAudio) }
    var showFace by remember { mutableStateOf(settings.showFace) }
    var stayConnected by remember { mutableStateOf(settings.stayConnected) }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("Where she is") },
        text = {
            /* Scrollable, because this dialog is taller than a phone on its side. Landscape
               gives it about 1080px and the content wants more, so without this the room
               field, both hints and both switches are simply **not drawn** — with one
               orphaned toggle floating over the credential box, which is what it looked
               like when the room field was added and the shot was taken. */
            Column(
                verticalArrangement = Arrangement.spacedBy(10.dp),
                modifier = Modifier.verticalScroll(rememberScrollState()),
            ) {
                OutlinedTextField(host, { host = it }, label = { Text("Host") }, singleLine = true)
                OutlinedTextField(port, { port = it.filter(Char::isDigit) }, label = { Text("Port") }, singleLine = true)
                OutlinedTextField(
                    credential, { credential = it },
                    label = { Text("Token or remote key") },
                    singleLine = true,
                )
                Text(
                    "Her remote key is in Settings on the host and survives a restart. The " +
                        "per-run token is in her log and only works over loopback — which " +
                        "includes an adb reverse tunnel.",
                    fontSize = 11.sp,
                    color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f),
                )

                OutlinedTextField(room, { room = it }, label = { Text("Room") }, singleLine = true)
                Text(
                    "Which space this device is. She is one being in several rooms — same " +
                        "brain, same face, separate conversations — so what is said here " +
                        "stays here. Leaving it empty puts this device in the desktop's room.",
                    fontSize = 11.sp,
                    color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f),
                )

                /* Her page cannot offer this. It was loaded over plain http, so a browser
                   will not enumerate cameras for it and her camera list reads "Not known
                   yet" — while the device it would be choosing between belongs to this app
                   anyway. Same conclusion her Stage 15 item 3 reaches for the microphone. */
                Text("Camera", fontSize = 14.sp)
                Row(verticalAlignment = Alignment.CenterVertically, modifier = Modifier.fillMaxWidth()) {
                    Lens.entries.forEach { option ->
                        Row(
                            verticalAlignment = Alignment.CenterVertically,
                            modifier = Modifier.weight(1f),
                        ) {
                            RadioButton(
                                selected = camera == option,
                                onClick = { camera = option },
                            )
                            Text(
                                if (option == Lens.Front) "Front" else "Back",
                                fontSize = 13.sp,
                            )
                        }
                    }
                }
                Text(
                    "Which one she looks through, both for a glance and for following you. " +
                        "A device with only one camera uses it either way.",
                    fontSize = 11.sp,
                    color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f),
                )

                Row(verticalAlignment = Alignment.CenterVertically, modifier = Modifier.fillMaxWidth()) {
                    Column(Modifier.weight(1f)) {
                        Text("Speak here", fontSize = 14.sp)
                        Text(
                            // She will not send audio unasked, and this is why: on a desk
                            // cable this handset is in the same room as her speakers.
                            "Off unless this is the device that should make noise. She plays " +
                                "through the host's speakers regardless.",
                            fontSize = 11.sp,
                            color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f),
                        )
                    }
                    Switch(checked = playAudio, onCheckedChange = { playAudio = it })
                }

                Row(verticalAlignment = Alignment.CenterVertically, modifier = Modifier.fillMaxWidth()) {
                    Column(Modifier.weight(1f)) {
                        Text("Stay in the background", fontSize = 14.sp)
                        Text(
                            "Her tray icon, as the desktop has. She keeps her room and her " +
                                "voice when you look away, and a notification brings her " +
                                "back or lets her go. Her camera stops either way.",
                            fontSize = 11.sp,
                            color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f),
                        )
                    }
                    Switch(
                        checked = stayConnected,
                        onCheckedChange = {
                            stayConnected = it
                            // Asked here rather than at launch: the notification only means
                            // anything once this is on, and it is the only way back to her.
                            if (it) onWantNotifications()
                        },
                    )
                }

                Row(verticalAlignment = Alignment.CenterVertically, modifier = Modifier.fillMaxWidth()) {
                    Column(Modifier.weight(1f)) {
                        Text("Show her face", fontSize = 14.sp)
                        Text(
                            "Her own renderer, served by her socket. The expensive part of " +
                                "the page — an older handset may not manage it.",
                            fontSize = 11.sp,
                            color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f),
                        )
                    }
                    Switch(checked = showFace, onCheckedChange = { showFace = it })
                }
            }
        },
        confirmButton = {
            TextButton(onClick = {
                settings.host = host
                settings.port = port.toIntOrNull() ?: 8848
                settings.credential = credential
                settings.room = room
                settings.camera = camera.id
                settings.playAudio = playAudio
                settings.showFace = showFace
                settings.stayConnected = stayConnected
                onSave()
            }) { Text("Connect") }
        },
        dismissButton = { TextButton(onClick = onDismiss) { Text("Cancel") } },
    )
}
```
