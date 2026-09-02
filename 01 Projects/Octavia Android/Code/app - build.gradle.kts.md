---
project: Octavia Android
tags: [octavia, octavia-android, code]
source-path: app\build.gradle.kts
---

# app\build.gradle.kts

```kotlin
plugins {
  alias(libs.plugins.android.application)
  alias(libs.plugins.compose.compiler)
}

android {
    namespace = "com.n3ctr0.octavia"
    compileSdk = 36
    defaultConfig {
        applicationId = "com.n3ctr0.octavia"
        minSdk = 28
        targetSdk = 36
        /* Kept in step with `versions.md`, which is the record. It said 0.9.1 while every
           APK ever built reported 1.0 — so a handset could not be asked what it was running,
           and "is that the build with the fix?" had no answer but a reinstall.

           The code is `major*10000 + minor*100 + patch`: monotonic, and readable back. */
        versionCode = 1000
        versionName = "0.10.0"
        // The camera can only be proven on a real camera, so its check is an instrumented
        // test rather than a unit test. Nothing ran on device before this.
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    buildFeatures {
      compose = true
      aidl = false
      buildConfig = false
      shaders = false
    }

    packaging {
      resources {
        excludes += "/META-INF/{AL2.0,LGPL2.1}"
      }
    }
}

kotlin {
    jvmToolchain(17)
}

dependencies {
  val composeBom = platform(libs.androidx.compose.bom)
  implementation(composeBom)
  androidTestImplementation(composeBom)

  // Core Android dependencies
  implementation(libs.androidx.core.ktx)
  implementation(libs.androidx.lifecycle.runtime.ktx)
  implementation(libs.androidx.activity.compose)

  // Arch Components
  implementation(libs.androidx.lifecycle.runtime.compose)
  implementation(libs.androidx.lifecycle.viewmodel.compose)
  // viewModelScope, for the one thing this client does that is genuinely async: `look`.
  implementation(libs.androidx.lifecycle.viewmodel.ktx)

  // Compose
  implementation(libs.androidx.compose.ui)
  implementation(libs.androidx.compose.ui.tooling.preview)
  implementation(libs.androidx.compose.material3)
  // Tooling
  debugImplementation(libs.androidx.compose.ui.tooling)
  // Instrumented tests
  androidTestImplementation(libs.androidx.compose.ui.test.junit4)
  debugImplementation(libs.androidx.compose.ui.test.manifest)

  // Local tests: jUnit, coroutines, Android runner
  testImplementation(libs.junit)
  testImplementation(libs.kotlinx.coroutines.test)

  // Instrumented tests: jUnit rules and runners
  androidTestImplementation(libs.androidx.test.core)
  androidTestImplementation(libs.androidx.test.ext.junit)
  androidTestImplementation(libs.androidx.test.runner)
  androidTestImplementation(libs.androidx.test.espresso.core)

  // Her eyes, on this device. See PROTOCOL.md: a face on a plain http origin cannot use
  // getUserMedia, so the camera is native and the WebView only draws.
  implementation(libs.androidx.camera.core)
  implementation(libs.androidx.camera.camera2)
  implementation(libs.androidx.camera.lifecycle)

  // The embedder seam. Needed for addDocumentStartJavaScript and addWebMessageListener,
  // both of which take an origin allow-list - which addJavascriptInterface does not.
  implementation(libs.androidx.webkit)

  // Her socket. See the note in libs.versions.toml for why this is the only one.
  implementation(libs.okhttp)
}
```
