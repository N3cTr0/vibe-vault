---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\AndroidManifest.xml
---

# app\src\main\AndroidManifest.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <!-- Held only while the talk button is down. The microphone belongs to the app rather
         than to a WebView because getUserMedia will not run on a plain http LAN origin —
         that is not a secure context. The camera arrived the same way. -->
    <uses-permission android:name="android.permission.RECORD_AUDIO" />

    <!-- Opened only on `look`, for one frame, never on a timer and never speculatively.
         PROTOCOL.md makes all four of those promises on a face's behalf. -->
    <uses-permission android:name="android.permission.CAMERA" />

    <!-- Not required: a device with no camera is still a perfectly good face. It answers
         `look` with an error, which the protocol asks for and which is not a failure. -->
    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <!-- Her tray icon. Only ever started when "Stay in the background" is switched on;
         with it off, no service runs and leaving the app drops her socket as before.
         `specialUse` rather than `dataSync` because this is not a sync and the six-hour
         daily cap on that type would end her presence mid-afternoon with no explanation. -->
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />

    <!-- The notification *is* the way back to her, so a refusal costs the way back rather
         than an alert. Asked for at the moment the setting is switched on, not at launch. -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application
        android:allowBackup="true"
        android:hardwareAccelerated="true"
        android:largeHeap="true"
        android:networkSecurityConfig="@xml/network_security_config"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:windowSoftInputMode="adjustResize"
        android:theme="@style/Theme.Octavia">
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service
            android:name=".service.OctaviaService"
            android:exported="false"
            android:foregroundServiceType="specialUse">
            <property
                android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                android:value="Keeps the WebSocket connection to the user's own
                    self-hosted Octavia server open while the app is in the background,
                    so her voice keeps playing and she stays in her room. No socket is
                    forwarded and no third-party service is contacted." />
        </service>
    </application>

</manifest>
```
