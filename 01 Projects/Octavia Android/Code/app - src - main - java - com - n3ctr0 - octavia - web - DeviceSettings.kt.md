---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\web\DeviceSettings.kt
---

# app\src\main\java\com\n3ctr0\octavia\web\DeviceSettings.kt

```kotlin
package com.n3ctr0.octavia.web

import com.n3ctr0.octavia.camera.Lens
import com.n3ctr0.octavia.data.Settings
import org.json.JSONArray
import org.json.JSONObject

/**
 * This app's own settings, described so that her page can draw them.
 *
 * **They used to live behind a long press on an invisible corner.** That began as a recovery
 * path — a wrong key leaves an app that cannot be repaired from the device, which is exactly
 * where this handset was on 09/01/2026 and it took `adb` to get out of — and a recovery path
 * is a poor place to keep a setting somebody wants to change. Choosing which camera she looks
 * through is not an emergency.
 *
 * So they are rendered where every other setting is: in her Settings panel, above hers. The
 * long press stays, because the reason it existed has not gone away — if the address is wrong
 * the page will not load, and then there is no panel to open.
 *
 * **The page is handed fields, never told what it is embedded in.** A Windows client would
 * describe its hotkey through the same call and get the same rows; nothing in `bridge.js`
 * mentions Android, which is the rule the embedder seam was written under.
 */
object DeviceSettings {

    fun describe(settings: Settings): JSONArray {
        val fields = JSONArray()

        fields.put(field(
            key = "host", label = "Her address", type = "text", value = settings.host,
            hint = "The machine her server runs on. Changing this reconnects."
        ))

        fields.put(field(
            key = "port", label = "Port", type = "number", value = settings.port,
            hint = "8848 unless she was told otherwise."
        ))

        /* A password field. The remote key is not a secret from the person holding the
           phone — they typed it in — but it is one from whoever is standing behind them,
           and it was previously drawn in clear text at full width. */
        fields.put(field(
            key = "credential", label = "Token or remote key", type = "password",
            value = settings.credential,
            hint = "Her remote key survives a restart. The per-run token is in her log and " +
                "only works over loopback."
        ))

        fields.put(field(
            key = "room", label = "Room", type = "text", value = settings.room,
            hint = "Which space this device is. Same brain, separate conversation — so what " +
                "is said here stays here. Empty puts it in the desktop's room."
        ))

        fields.put(field(
            key = "camera", label = "Camera", type = "choice", value = settings.camera,
            hint = "Which one she looks through, for a glance and for following you.",
            options = Lens.entries.map { it.id to it.label }
        ))

        fields.put(field(
            key = "playAudio", label = "Speak here", type = "switch", value = settings.playAudio,
            hint = "Off unless this is the device that should make noise."
        ))

        fields.put(field(
            key = "stayConnected", label = "Stay in the background", type = "switch",
            value = settings.stayConnected,
            hint = "She keeps her room and her voice when you look away. Her camera stops " +
                "either way."
        ))

        return fields
    }

    /**
     * Apply one field.
     *
     * `onChanged` is told whether the connection itself moved, because those four fields are
     * the ones that need the socket dropped and reopened — and doing that for a camera choice
     * would take a new `FaceId` and re-enter the room for no reason anybody could see.
     */
    fun apply(settings: Settings, key: String, value: Any?, onChanged: (reconnect: Boolean) -> Unit) {
        when (key) {
            "host" -> { settings.host = text(value); onChanged(true) }
            "port" -> { settings.port = number(value) ?: 8848; onChanged(true) }
            "credential" -> { settings.credential = text(value); onChanged(true) }
            "room" -> { settings.room = text(value); onChanged(true) }
            "playAudio" -> { settings.playAudio = flag(value); onChanged(true) }

            // Read at every use, so it takes effect on the next look with nothing to restart.
            "camera" -> { settings.camera = text(value); onChanged(false) }

            // Read from the activity's lifecycle each time, so likewise.
            "stayConnected" -> { settings.stayConnected = flag(value); onChanged(false) }

            else -> throw IllegalArgumentException("this device has no setting called '$key'")
        }
    }

    private fun field(
        key: String,
        label: String,
        type: String,
        value: Any?,
        hint: String,
        options: List<Pair<String, String>>? = null,
    ): JSONObject {
        val out = JSONObject()
            .put("key", key)
            .put("label", label)
            .put("type", type)
            .put("value", value)
            .put("hint", hint)

        if (options != null) {
            val list = JSONArray()
            for ((id, text) in options) {
                list.put(JSONObject().put("value", id).put("label", text))
            }
            out.put("options", list)
        }

        return out
    }

    /** `org.json` hands back whatever the page sent, so every read is defensive — the same
     *  rule this app reads *her* messages by. */
    private fun text(value: Any?): String = value?.toString()?.trim() ?: ""

    private fun number(value: Any?): Int? = when (value) {
        is Number -> value.toInt()
        else -> value?.toString()?.trim()?.toIntOrNull()
    }

    private fun flag(value: Any?): Boolean = when (value) {
        is Boolean -> value
        else -> value?.toString()?.equals("true", ignoreCase = true) == true
    }
}
```
