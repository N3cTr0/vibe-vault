---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\ui\main\FaceScreen.kt
---

# app\src\main\java\com\n3ctr0\octavia\ui\main\FaceScreen.kt

```kotlin
package com.n3ctr0.octavia.ui.main

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.text.KeyboardActions
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.net.FaceSocket

/**
 * Stage 1: connect, see what she says, type at her.
 *
 * No avatar and no microphone, and that is the stage rather than a shortcut — the J7 Pro
 * cannot render a VRM and the protocol cannot yet carry audio upstream. What this screen is
 * for is proving the parts that are genuinely risky: pairing, the socket, reconnection, and
 * whether any of it survives a locked screen and a roaming radio.
 */
@Composable
fun FaceScreen(state: FaceState, settings: Settings, viewModel: FaceViewModel, modifier: Modifier = Modifier) {

    var draft by remember { mutableStateOf("") }
    var settingsOpen by remember { mutableStateOf(!settings.configured) }
    val listState = rememberLazyListState()

    // Follow the tail. A transcript that has to be dragged to the bottom after every answer
    // is one nobody reads.
    LaunchedEffect(state.lines.size) {
        if (state.lines.isNotEmpty()) listState.animateScrollToItem(state.lines.lastIndex)
    }

    Column(modifier.fillMaxSize().padding(16.dp)) {

        Row(verticalAlignment = Alignment.CenterVertically, modifier = Modifier.fillMaxWidth()) {
            Text("Octavia.", fontWeight = FontWeight.SemiBold, fontSize = 20.sp)
            Spacer(Modifier.weight(1f))
            StatePill(state)
            Spacer(Modifier.width(8.dp))
            TextButton(onClick = { settingsOpen = true }) { Text("Set up") }
        }

        LinkLine(state, viewModel)

        LazyColumn(
            state = listState,
            modifier = Modifier.weight(1f).fillMaxWidth(),
            verticalArrangement = Arrangement.spacedBy(10.dp),
        ) {
            items(state.lines) { line -> TranscriptLine(line) }
        }

        if (state.caption.isNotBlank()) {
            Text(
                state.caption,
                fontSize = 14.sp,
                // A partial transcript is shown, but never as though it were settled.
                color = MaterialTheme.colorScheme.onSurface.copy(alpha = if (state.captionTentative) 0.45f else 0.75f),
                modifier = Modifier.fillMaxWidth().padding(vertical = 6.dp),
            )
        }

        Row(verticalAlignment = Alignment.CenterVertically, modifier = Modifier.fillMaxWidth()) {
            OutlinedTextField(
                value = draft,
                onValueChange = { draft = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("Say something") },
                enabled = state.link == FaceSocket.Link.Up,
                singleLine = true,
                keyboardOptions = KeyboardOptions(imeAction = ImeAction.Send),
                keyboardActions = KeyboardActions(onSend = {
                    viewModel.say(draft)
                    draft = ""
                }),
            )
            Spacer(Modifier.width(8.dp))
            Button(
                onClick = { viewModel.say(draft); draft = "" },
                enabled = state.link == FaceSocket.Link.Up && draft.isNotBlank(),
            ) { Text("Send") }
        }
    }

    if (settingsOpen) {
        SetupDialog(
            settings = settings,
            onDismiss = { settingsOpen = false },
            onSave = { settingsOpen = false; viewModel.reconnect() },
        )
    }
}

@Composable
private fun StatePill(state: FaceState) {
    val colour = when (state.her) {
        "listening" -> Color(0xFF3B82F6)
        "thinking" -> Color(0xFFF59E0B)
        "speaking" -> Color(0xFF10B981)
        else -> MaterialTheme.colorScheme.outline
    }
    Row(verticalAlignment = Alignment.CenterVertically) {
        Box(Modifier.size(8.dp).clip(CircleShape).background(colour))
        Spacer(Modifier.width(6.dp))
        Text(state.her.uppercase(), fontSize = 11.sp, fontFamily = FontFamily.Monospace)
    }
}

/** The connection, said plainly. "Nothing is happening" is the worst thing a face can show. */
@Composable
private fun LinkLine(state: FaceState, viewModel: FaceViewModel) {
    val text = when (state.link) {
        FaceSocket.Link.Idle -> "Not connected. Set up her address and key."
        FaceSocket.Link.Connecting -> "Reaching her…"
        FaceSocket.Link.Retrying -> state.linkDetail?.let { "$it Retrying…" } ?: "Retrying…"
        FaceSocket.Link.Up ->
            buildString {
                append("Connected")
                if (state.protocol > 0) append(" · protocol ${state.protocol}")
                if (state.model.isNotBlank()) append(" · ${state.model}")
                if (state.profile.isNotBlank()) append(" · ${state.profile}")
            }
    }

    Row(verticalAlignment = Alignment.CenterVertically, modifier = Modifier.fillMaxWidth().padding(vertical = 8.dp)) {
        Text(
            text,
            fontSize = 12.sp,
            color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.6f),
            modifier = Modifier.weight(1f),
        )
        if (state.link != FaceSocket.Link.Up) {
            TextButton(onClick = { viewModel.reconnect() }) { Text("Retry") }
        }
    }

    if (state.needKey) {
        Text(
            "She has no API key, so she cannot think. That is fixed on the host, not here.",
            fontSize = 12.sp,
            color = MaterialTheme.colorScheme.error,
            modifier = Modifier.fillMaxWidth().padding(bottom = 8.dp),
        )
    }
}

@Composable
private fun TranscriptLine(line: Line) {
    when (line.kind) {
        // Shown, faintly, and never swallowed — the protocol is explicit about this one.
        Line.Kind.Overheard -> Text(
            line.text,
            fontSize = 12.sp,
            color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.35f),
            modifier = Modifier.fillMaxWidth(),
        )

        Line.Kind.Notice -> Text(
            line.text,
            fontSize = 12.sp,
            textAlign = TextAlign.Center,
            color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.5f),
            modifier = Modifier.fillMaxWidth(),
        )

        else -> Column(Modifier.fillMaxWidth()) {
            Text(
                line.who,
                fontSize = 11.sp,
                fontFamily = FontFamily.Monospace,
                color = MaterialTheme.colorScheme.onSurface.copy(alpha = 0.45f),
            )
            Text(line.text, fontSize = 15.sp)
        }
    }
}

@Composable
private fun SetupDialog(settings: Settings, onDismiss: () -> Unit, onSave: () -> Unit) {
    var host by remember { mutableStateOf(settings.host) }
    var port by remember { mutableStateOf(settings.port.toString()) }
    var credential by remember { mutableStateOf(settings.credential) }

    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("Where she is") },
        text = {
            Column(verticalArrangement = Arrangement.spacedBy(10.dp)) {
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
            }
        },
        confirmButton = {
            TextButton(onClick = {
                settings.host = host
                settings.port = port.toIntOrNull() ?: 8848
                settings.credential = credential
                onSave()
            }) { Text("Connect") }
        },
        dismissButton = { TextButton(onClick = onDismiss) { Text("Cancel") } },
    )
}
```
