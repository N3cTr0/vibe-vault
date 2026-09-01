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

    val configured: Boolean get() = host.isNotBlank() && credential.isNotBlank()

    private companion object {
        const val HOST = "host"
        const val PORT = "port"
        const val CREDENTIAL = "credential"
        const val PLAY_AUDIO = "playAudio"
        const val SHOW_FACE = "showFace"
    }
}
```
