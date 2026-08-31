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

    val configured: Boolean get() = host.isNotBlank() && credential.isNotBlank()

    private companion object {
        const val HOST = "host"
        const val PORT = "port"
        const val CREDENTIAL = "credential"
    }
}
```
