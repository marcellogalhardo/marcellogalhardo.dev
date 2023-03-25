---
title: "Using Compose Beta on AS 4.1"
date: 2021-03-30T17:37:41+02:00
draft: false
toc: false
images:
categories:
  - software development
tags:
  - android
  - compose
  - gradle
---

Jetpack Compose hit Beta! Many teams are excited to experiment with Compose, but as you might know, since [1.0.0-alpha04](https://developer.android.com/jetpack/androidx/releases/compose-compiler#compiler-1.0.0-alpha04), the compiler has been refactored to a new group and became incompatible with the current Android Studio (AS) 4.1 stable:

> Compose Version `1.0.0-alpha04` is only compatible with Android Studio 4.2 Canary 13 and later.
 
Been forced to use a Canary version of AS is a real bummer. There are cases in which you want to explore Compose in a real-world application (e.g., converting a Design System to Compose) while letting developers work in parallel using the stable version 4.1 to ship production code. Happily, Compose is a standard Kotlin Compiler Plugin, and it is pretty straightforward to apply it directly to your project:
1. Select the module you want to use Compose.
2. Remove the [default configuration](https://developer.android.com/jetpack/compose/setup#add-compose) from Compose docs, as we will set it up manually.
3. Apply the compiler plugin and include the runtime to your module.

**Heads-up**: the following code snippets assume you are using Groovy, but you can also do it with KTS. For more details, see [Gradle KTS Differences](#gradle-kts-differences).

As an example, let's configure Gradle with the latest Compose (`1.0.0-beta03`):

```groovy
import org.jetbrains.kotlin.gradle.plugin.KotlinPluginKt

android {
    defaultConfig {
        minSdkVersion 21
    }

    // Set both the Java and Kotlin compilers to target Java 8.

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = "1.8"
        useIR = true
    }
}

dependencies {
    def composeVersion = "1.0.0-beta03"

    add(KotlinPluginKt.PLUGIN_CLASSPATH_CONFIGURATION_NAME, "androidx.compose.compiler:compiler:${composeVersion}")
    implementation "androidx.compose.runtime:runtime:${composeVersion}"
}
```

Now you can [add Jetpack Compose toolkit dependencies](https://developer.android.com/jetpack/compose/setup#compose-compiler) as established in the official docs:

```groovy
dependencies {
    def composeVersion = "1.0.0-beta03"
    
    // ...compiler and runtime from previous code snippet

    implementation "androidx.compose.ui:ui:${composeVersion}"
    // Tooling support (Previews, etc.)
    implementation "androidx.compose.ui:ui-tooling:${composeVersion}"
    // Foundation (Border, Background, Box, Image, Scroll, shapes, animations, etc.)
    implementation "androidx.compose.foundation:foundation:${composeVersion}"
    // Material Design
    implementation "androidx.compose.material:material:${composeVersion}"
    // Material design icons
    implementation "androidx.compose.material:material-icons-core:${composeVersion}"
    implementation "androidx.compose.material:material-icons-extended:${composeVersion}"
    // Integration with activities
    implementation "androidx.activity:activity-compose:1.3.0-alpha05"
    // Integration with ViewModels
    implementation "androidx.lifecycle:lifecycle-viewmodel-compose:1.0.0-alpha03"

    // UI Tests
    androidTestImplementation "androidx.compose.ui:ui-test-junit4:${composeVersion}"
}
``` 

Once it is in place, you can build Compose Apps in your AS 4.1 stable. Note that you will not be able to use the basic IDE tooling (e.g., Preview) without opening your project in a higher version of Android Studio. Nevertheless, if you do not upgrade Android Gradle Plugin, this set-up enables you to switch between AS 4.1 and Arctic Fox and build the project with success. Keep in mind you should remove those manual configurations once you migrate to AS Arctic Fox or later.

## Gradle KTS Differences {#gradle-kts-differences}

As pointed out by [John O.Reilly](https://twitter.com/joreilly?s=20), if you use `gradle.kts` the `KotlinPluginKt` import will fail, and instead, you must import directly `PLUGIN_CLASSPATH_CONFIGURATION_NAME` as following:

```kotlin
import org.jetbrains.kotlin.gradle.plugin.PLUGIN_CLASSPATH_CONFIGURATION_NAME

//...

dependencies {
    val composeVersion = "1.0.0-beta03"

    add(PLUGIN_CLASSPATH_CONFIGURATION_NAME, "androidx.compose.compiler:compiler:${composeVersion}")

    //...
}
```

All other aspects remain the same.

# Update 2021.04.01

Simplified the code snippet on how to add a Kotlin Compiler Plugins with tips provided by [Jake Wharton](https://twitter.com/JakeWharton). For a matter of history, here is the previous solution:

```groovy
configurations {
    kotlinPlugin
}

dependencies {
    kotlinPlugin "androidx.compose.compiler:compiler:${composeVersion}"
}

tasks.withType(org.jetbrains.kotlin.gradle.tasks.KotlinCompile).configureEach {
    def pluginConfiguration = configurations.kotlinPlugin
    dependsOn(pluginConfiguration)
    doFirst {
       if (!pluginConfiguration.isEmpty()) {
            def composePlugin = pluginConfiguration.files.find { File file ->
                file.path.contains("/androidx.compose.compiler/compiler/${composeVersion}/")
            }
            if (composePlugin != null) {
                kotlinOptions.freeCompilerArgs += "-Xplugin=${composePlugin}"
            }
        }
    }
}
```

# Credits

Special thanks to [Jake Wharton](https://twitter.com/JakeWharton) for answering my question about the subject with the idea that originated this post, and [Colton Idle](https://twitter.com/ColtonIdle) for informing me about the simpler way to include a Kotlin Compiler Plugin.

> ℹ️ To stay up to date with my writing, follow me on [Mastodon](http://androiddev.social/@mg) or [Twitter](https://twitter.com/marcellogalhard). If you have any questions, feel free to reach me out! ℹ️
