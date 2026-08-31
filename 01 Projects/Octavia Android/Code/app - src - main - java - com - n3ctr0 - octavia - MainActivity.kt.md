---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\src\main\java\com\n3ctr0\octavia\MainActivity.kt
---

# app\src\main\java\com\n3ctr0\octavia\MainActivity.kt

```kotlin
package com.n3ctr0.octavia

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.safeDrawingPadding
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Surface
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.n3ctr0.octavia.data.Settings
import com.n3ctr0.octavia.theme.OctaviaTheme
import com.n3ctr0.octavia.ui.main.FaceScreen
import com.n3ctr0.octavia.ui.main.FaceViewModel

class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val settings = Settings(applicationContext)

        enableEdgeToEdge()
        setContent {
            OctaviaTheme {
                Surface(Modifier.fillMaxSize(), color = MaterialTheme.colorScheme.background) {

                    /* The socket outlives a rotation, which is the whole reason it lives in a
                       ViewModel rather than in a composable: reconnecting every time the
                       screen turns would be visible, slow, and wrong. */
                    val model: FaceViewModel = viewModel(factory = object : ViewModelProvider.Factory {
                        @Suppress("UNCHECKED_CAST")
                        override fun <T : ViewModel> create(modelClass: Class<T>): T =
                            FaceViewModel(settings).also { it.connect() } as T
                    })

                    val state by model.state.collectAsStateWithLifecycle()
                    FaceScreen(state, settings, model, Modifier.safeDrawingPadding())
                }
            }
        }
    }
}
```
