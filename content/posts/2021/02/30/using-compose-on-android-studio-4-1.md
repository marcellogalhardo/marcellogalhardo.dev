---
title: "Using Compose on Android Studio 4.1"
date: 2021-02-30T09:02:50+01:00
draft: false
---

Jetpack Compose hit Beta! Many teams are excited to experiment with Compose, but as you might know, since [1.0.0-alpha04](https://developer.android.com/jetpack/androidx/releases/compose-compiler#compiler-1.0.0-alpha04
), the compiler has been refactored to a new group and became incompatible with the current Android Studio (AS) 4.1 stable:

> Compose Version `1.0.0-alpha04` is only compatible with Android Studio 4.2 Canary 13 and later.
 
Been forced a Beta version of AS is a bummer. Happily, Compose is a standard Kotlin Compiler Plugin, and it is pretty straightforward to apply it directly to your project:
1. Select the module you want to use Compose.
2. Remove the default configuration from Compose docs, as we will set it up manually.
3. Apply the compiler plugin and include the runtime to your module.

As an example, I will set up the latest Compose (`1.0.0-beta03`) with Material and ViewModel artefacts:

```groovy
configurations {
    kotlinPlugin
}

dependencies {
    def composeVersion = "1.0.0-beta03"
    
    kotlinPlugin "androidx.compose.compiler:compiler:${composeVersion}"

    // Compose
    implementation "androidx.compose.runtime:runtime:${composeVersion}"
    implementation "androidx.compose.ui:ui:${composeVersion}"
    implementation "androidx.compose.ui:ui-tooling:${composeVersion}"
    implementation "androidx.compose.foundation:foundation:${composeVersion}"
    implementation "androidx.compose.material:material:${composeVersion}"
    implementation "androidx.compose.material:material-icons-core:${composeVersion}"
    implementation "androidx.compose.material:material-icons-extended:${composeVersion}"
    androidTestImplementation "androidx.compose.ui:ui-test:${composeVersion}"

    // Jetpack Libs
    implementation "androidx.activity:activity-compose:1.3.0-alpha05"
    implementation "androidx.lifecycle:lifecycle-viewmodel-compose:1.0.0-alpha03"
}

tasks.withType(org.jetbrains.kotlin.gradle.tasks.KotlinCompile).configureEach {
    kotlinOptions {
        useIR = true
    }
    def pluginConfiguration = configurations.kotlinPlugin
    dependsOn(pluginConfiguration)
    doFirst {
        if (!pluginConfiguration.isEmpty()) {
            kotlinOptions.freeCompilerArgs += "-Xplugin=${pluginConfiguration.files.first()}"
        }
    }
}
``` 

Once it is in place, you can build Compose Apps in your AS 4.1 stable. Note that you will not be able to use the basic IDE tooling (e.g., Preview) without opening your project in a higher version of Android Studio. Nevertheless, if you do not upgrade Android Gradle Plugin, this set-up enables you to switch between AS 4.1 and Arctic Fox and build the project with success. Keep in mind that you should remove those manual configurations once you migrate to AS 4.2 or above.
 
Credits to [Jake Wharton](https://twitter.com/JakeWharton) which replied [my question on Android Study Group](https://androidstudygroup.slack.com
/archives/CJH03QASH/p1603978103094800) explaining [multiple optios to set-up Compose on AS 4.1](https://androidstudygroup.slack.com/archives/CJH03QASH/p1603979177097400).
