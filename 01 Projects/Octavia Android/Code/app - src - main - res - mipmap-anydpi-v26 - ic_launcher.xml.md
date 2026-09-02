---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\res\mipmap-anydpi-v26\ic_launcher.xml
---

# app\src\main\res\mipmap-anydpi-v26\ic_launcher.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--
  Her icon, the one her desktop client already wears. It was the Android Studio template
  robot until 0.10.0 — which is what the notification put on screen, since MIUI draws the
  launcher icon there rather than the small one.

  Generated from `src/Octavia.App/Assets/octavia.ico` in her repo, at the 256 frame. The
  foreground sits inside the 72dp safe zone, because a launcher may mask this to any shape
  and her plate has its own frame that should not be clipped.
-->
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/ic_launcher_background" />
    <foreground android:drawable="@mipmap/ic_launcher_foreground" />
    <!-- Themed icons are drawn as a silhouette, so the artwork cannot be one. The status
         bar mark is already a flat shape and is the right thing here. -->
    <monochrome android:drawable="@drawable/ic_notification" />
</adaptive-icon>
```
