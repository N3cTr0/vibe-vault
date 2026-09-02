---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\service\OctaviaService.kt
---

# app\src\main\java\com\n3ctr0\octavia\service\OctaviaService.kt

```kotlin
package com.n3ctr0.octavia.service

import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.app.Service
import android.content.Context
import android.content.Intent
import android.os.Build
import android.os.IBinder
import android.util.Log
import com.n3ctr0.octavia.MainActivity
import com.n3ctr0.octavia.R

/**
 * Her tray icon, in the only form Android has for one.
 *
 * **This service holds no socket and no state.** It exists so that the process it is part of
 * is not killed while the activity is in the background, which is what lets the connection in
 * `FaceViewModel` survive being looked away from. Putting the socket in here as well was the
 * obvious design and the wrong one: it would mean two owners of one connection, and the
 * reconnect and floor logic — the part that has actually been debugged — would have to move
 * with it.
 *
 * **The notification is not decoration; it is the affordance.** A background service with no
 * way back to it is how an app becomes something you have to force-stop. Tapping it brings
 * her window back, exactly as clicking a tray icon does, and *Let her go* is the tray's
 * Quit — without it, "stay connected" would be a state with no exit that did not involve
 * Android's own settings.
 *
 * Started only when *Stay in the background* is on. With it off nothing here ever runs, and
 * leaving the app drops her socket the way it always has.
 */
class OctaviaService : Service() {

    override fun onBind(intent: Intent?): IBinder? = null

    /**
     * **The promise is kept here, not in `onStartCommand`, and that is the whole point.**
     *
     * `startForegroundService` opens a short window in which this service *must* call
     * `startForeground` or the system kills the app —
     * `ForegroundServiceDidNotStartInTimeException`. Doing it in `onStartCommand` looks
     * equivalent and is not: a `stopService` arriving in between means `onStartCommand` never
     * runs, the promise is never kept, and the app dies for a service it had already been told
     * to abandon. Rotating the handset did exactly that.
     *
     * `onCreate` always runs, and runs first. The caller's race can no longer reach it.
     */
    override fun onCreate() {
        super.onCreate()
        startForeground(ID, notification())
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        if (intent?.action == ACTION_RELEASE) {
            // The person asked her to go. Stopping the service is only half of it — the
            // activity is still sitting in the background holding the socket, so it is told
            // to finish and takes the connection down with it on the way out.
            Log.i(TAG, "let go from the notification")
            sendBroadcast(Intent(ACTION_RELEASE).setPackage(packageName))
            stopSelf()
            return START_NOT_STICKY
        }

        /* Already foreground from `onCreate`; nothing to do but stay running.

           NOT sticky. A restart by the system would recreate this service with no activity
           behind it — a notification saying she is here, attached to nothing, and no way to
           tell the difference from the outside. */
        return START_NOT_STICKY
    }

    private fun notification(): Notification {
        val manager = getSystemService(NotificationManager::class.java)

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            manager.createNotificationChannel(
                NotificationChannel(CHANNEL, "Octavia is connected", NotificationManager.IMPORTANCE_LOW)
                    .apply {
                        description = "Shown while she stays connected in the background."
                        setShowBadge(false)
                    }
            )
        }

        val surface = PendingIntent.getActivity(
            this,
            0,
            Intent(this, MainActivity::class.java)
                .setFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP or Intent.FLAG_ACTIVITY_REORDER_TO_FRONT),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE,
        )

        val release = PendingIntent.getService(
            this,
            1,
            Intent(this, OctaviaService::class.java).setAction(ACTION_RELEASE),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE,
        )

        return Notification.Builder(this, CHANNEL)
            .setContentTitle("Octavia is here")
            // Says what is and is not true: the connection is up, and she is not watching —
            // the camera stops when her face leaves the screen, because the marker that
            // promises it is live goes with it.
            .setContentText("Connected. Her camera is off while you are away.")
            .setSmallIcon(R.drawable.ic_notification)
            .setContentIntent(surface)
            .addAction(Notification.Action.Builder(null, "Let her go", release).build())
            .setOngoing(true)
            .setShowWhen(false)
            .build()
    }

    companion object {
        private const val TAG = "OctaviaService"
        private const val CHANNEL = "octavia.presence"
        private const val ID = 1

        /** Broadcast to the activity when *Let her go* is pressed. */
        const val ACTION_RELEASE = "com.n3ctr0.octavia.RELEASE"

        fun start(context: Context) {
            context.startForegroundService(Intent(context, OctaviaService::class.java))
        }

        fun stop(context: Context) {
            context.stopService(Intent(context, OctaviaService::class.java))
        }
    }
}
```
