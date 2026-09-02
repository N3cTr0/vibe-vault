---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\res\drawable\ic_notification.xml
---

# app\src\main\res\drawable\ic_notification.xml

```xml
<!--
  Her mark in the status bar.

  Monochrome and flat on purpose: Android draws a small icon as a silhouette, tinting every
  non-transparent pixel one colour. The launcher icon was used here at first and arrived as
  the template robot, which is both wrong and the giveaway that nobody had looked at it.

  **Each circle is two half arcs, not one full one.** A single arc that ends where it starts
  is degenerate — the renderer has no way to know which way round to sweep — and draws
  nothing at all, which presents as a blank icon and then as the system substituting
  something of its own.
-->
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">

    <!-- A ring: outer circle with the inner one cut out of it. -->
    <path
        android:fillColor="#FFFFFF"
        android:fillType="evenOdd"
        android:pathData="M12,2.5 A9.5,9.5 0 1,0 12,21.5 A9.5,9.5 0 1,0 12,2.5 Z
                          M12,6 A6,6 0 1,0 12,18 A6,6 0 1,0 12,6 Z" />

    <!-- and a centre, so it reads as a presence rather than a letter O. -->
    <path
        android:fillColor="#FFFFFF"
        android:pathData="M12,9 A3,3 0 1,0 12,15 A3,3 0 1,0 12,9 Z" />

</vector>
```
