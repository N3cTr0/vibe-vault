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
 * her voice. Push-to-talk moved to the volume key for the same reason — see
 * `MainActivity.dispatchKeyEvent`.
 *
 * **Set up survives as a long press in the bottom-right corner**, because deleting it
 * outright would be a trap: a key that stops working would leave an app that cannot be
 * repaired from the device at all, which is exactly the state this handset was in on
 * 09/01/2026, and it took `adb` to get out of.
 */
@Composable
fun FaceScreen(state: FaceState, settings: Settings, viewModel: FaceViewModel, modifier: Modifier = Modifier) {

    var settingsOpen by remember { mutableStateOf(!settings.configured) }
    var faceShown by remember { mutableStateOf(settings.showFace) }

    Box(modifier.fillMaxSize()) {

        if (faceShown && state.link == FaceSocket.Link.Up) {
            // No padding, no corner radius, no weight. The page is the screen.
            FacePanel(config = settings, modifier = Modifier.fillMaxSize())
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
private fun SetupDialog(settings: Settings, onDismiss: () -> Unit, onSave: () -> Unit) {
    var host by remember { mutableStateOf(settings.host) }
    var port by remember { mutableStateOf(settings.port.toString()) }
    var credential by remember { mutableStateOf(settings.credential) }
    var room by remember { mutableStateOf(settings.room) }
    var playAudio by remember { mutableStateOf(settings.playAudio) }
    var showFace by remember { mutableStateOf(settings.showFace) }

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
                        "stays here. Her side does not understand rooms yet.",
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
                settings.playAudio = playAudio
                settings.showFace = showFace
                onSave()
            }) { Text("Connect") }
        },
        dismissButton = { TextButton(onClick = onDismiss) { Text("Cancel") } },
    )
}
```
