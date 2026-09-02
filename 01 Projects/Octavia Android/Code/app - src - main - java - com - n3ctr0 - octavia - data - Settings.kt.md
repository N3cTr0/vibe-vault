---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\data\Settings.kt
---

# app\src\main\java\com\n3ctr0\octavia\data\Settings.kt

```kotlin
package com.n3ctr0.octavia.data

import android.content.Context

/**
 * Where she is, and what to present to get in.
 *
 * **On storing the credential in plain app-private preferences.** Her own `remote.key` is
 * kept in the clear on the host, deliberately and with a comment saying so: it has to be
 * *readable* to be shown in Settings and typed in here, which rules out sealing it. The key
 * is not the security boundary at either end — the socket is meant to be reachable only
 * over Wireguard, and remote access is off by default. Storing it here the same way is
 * consistent with that trade rather than a second, worse decision.
 */
class Settings(context: Context) {

    private val prefs = context.getSharedPreferences("octavia", Context.MODE_PRIVATE)

    /** `127.0.0.1` is right far more often than it looks during development: `adb reverse`
     *  points the handset's own loopback at the host's, over USB, so nothing needs to be
     *  exposed to the network to build against her. Over Wireguard this becomes her LAN
     *  address. */
    var host: String
        get() = prefs.getString(HOST, "127.0.0.1") ?: "127.0.0.1"
        set(value) = prefs.edit().putString(HOST, value.trim()).apply()

    var port: Int
        get() = prefs.getInt(PORT, 8848)
        set(value) = prefs.edit().putInt(PORT, value).apply()

    var credential: String
        get() = prefs.getString(CREDENTIAL, "") ?: ""
        set(value) = prefs.edit().putString(CREDENTIAL, value.trim()).apply()

    /**
     * Whether to ask the host for her voice.
     *
     * **Off by default, and that is the protocol's decision rather than caution.** Audio is
     * a physical output: a face that draws her mouth has not thereby claimed the right to
     * make noise, and every face on the host's own machine would otherwise play her over
     * the speakers she is already using. On a desk cable this handset is in that same room.
     *
     * It becomes true for the device that is *meant* to be heard — a tablet on a wall in
     * another room — and that is a choice a person makes once.
     */
    var playAudio: Boolean
        get() = prefs.getBoolean(PLAY_AUDIO, false)
        set(value) = prefs.edit().putBoolean(PLAY_AUDIO, value).apply()

    /**
     * Whether to render her face in a WebView.
     *
     * Off by default because it is the expensive thing on the page — a VRM, a room shader
     * and a WebGL context — and because not every device that should hear her can draw her.
     * The J7 Pro is the case in point: 192 MB of heap and a single-core Mali.
     */
    var showFace: Boolean
        get() = prefs.getBoolean(SHOW_FACE, false)
        set(value) = prefs.edit().putBoolean(SHOW_FACE, value).apply()

    /**
     * Which space this device is, as far as she is concerned.
     *
     * She is one being in several rooms: same brain, same avatar, same personality, separate
     * conversations. What is said here should not appear on the desktop, and her answer here
     * should not play through speakers in an empty house.
     *
     * **Both connections this app makes — the native socket and the WebView panel — must
     * name the same room**, or its own face would be in a different space from itself.
     *
     * Her side has understood this since host v0.24.0: a face names its room in `ready`,
     * absent means the host room, and her voice, conversation and captions follow the room
     * rather than every face. Empty here would therefore put this handset in the *desktop's*
     * space, which is why the default is a name rather than a blank.
     */
    var room: String
        get() = prefs.getString(ROOM, "phone") ?: "phone"
        set(value) = prefs.edit().putString(ROOM, value.trim()).apply()

    /**
     * Which camera this device lends her — see `Lens`.
     *
     * Stored as an id rather than an ordinal so that reordering the enum cannot silently
     * repoint somebody's camera.
     */
    var camera: String
        get() = prefs.getString(CAMERA, "front") ?: "front"
        set(value) = prefs.edit().putString(CAMERA, value).apply()

    /**
     * Whether she stays connected once you look away.
     *
     * **This is the Windows client's tray, and the reason it is a setting rather than the
     * behaviour is that the two clients pay different prices for it.** A tray icon on a
     * desktop costs nothing anybody notices. The same promise on a handset is a socket, a
     * ping every twenty seconds and a radio that cannot fully sleep, which is why leaving
     * the app has dropped her since 0.3.0 — *"nothing is spent on a conversation nobody is
     * reading"*.
     *
     * Off by default, so the device that was doing the frugal thing keeps doing it. On, it
     * behaves like the desktop: her voice keeps playing, she keeps her room and her face
     * identity, and there is a notification to bring her back or let her go.
     *
     * **Her camera stops either way.** The marker that promises the camera is live is drawn
     * by her page, so it leaves with the screen — and a face that could watch with its
     * marker off-screen would break a promise the protocol makes on every face's behalf.
     */
    var stayConnected: Boolean
        get() = prefs.getBoolean(STAY_CONNECTED, false)
        set(value) = prefs.edit().putBoolean(STAY_CONNECTED, value).apply()

    val configured: Boolean get() = host.isNotBlank() && credential.isNotBlank()

    private companion object {
        const val HOST = "host"
        const val PORT = "port"
        const val CREDENTIAL = "credential"
        const val PLAY_AUDIO = "playAudio"
        const val SHOW_FACE = "showFace"
        const val ROOM = "room"
        const val CAMERA = "camera"
        const val STAY_CONNECTED = "stayConnected"
    }
}
```
