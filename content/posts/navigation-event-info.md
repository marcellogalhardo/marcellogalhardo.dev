---
title: "NavigationEvent Info"
date: 2025-11-01T22:22:22+22:22
draft: false
toc: false
images:
- /logo.png
categories:
- software development
tags:
  - android
  - kotlin
  - kmp
---

[NavigationEvent](https://developer.android.com/jetpack/androidx/releases/navigationevent) is a Kotlin Multiplatform library for handling system gestures - back and forward - on all platforms. It works on Android, iOS and Desktop with Web support coming next.

Android used to rely on `OnBackPressedDispatcher` from `androidx.activity`. That handled back presses, and that was enough. But once you target more than Android, you also need forward navigation - like on the web.

That's why `NavigationEvent` exists: one event system that works everywhere.

It's in [beta01](https://developer.android.com/jetpack/androidx/releases/navigationevent#1.0.0-beta01) on Android, and [OnBackPressedDispatcher](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:activity/activity/src/main/java/androidx/activity/OnBackPressedDispatcher.kt;l=70;drc=b36ffe4ae230e4bb1845223bcbb74e4849952144) already delegates to [NavigationEventDispatcher](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:activity/activity/src/main/java/androidx/activity/OnBackPressedDispatcher.kt;l=84;drc=b36ffe4ae230e4bb1845223bcbb74e4849952144) under the hood.

## Debugging Back State

Back handling gets messy fast. Multiple handlers register, each hoping to own the gesture. Where do you end up once back completes?

That's what the [NavigationEventInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:navigationevent/navigationevent/src/commonMain/kotlin/androidx/navigationevent/NavigationEventInfo.kt;l=38;drc=dc470c5b7ac70789fd921279f9f7f804046f3826) API answers.

Every `NavigationEventDispatcher` keeps a list of [handlers](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:navigationevent/navigationevent/src/commonMain/kotlin/androidx/navigationevent/NavigationEventDispatcher.kt;l=181;drc=6341049232684807b94d9d9030e9eef6ced75a38). Each handler exposes:
- [currentInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:navigationevent/navigationevent/src/commonMain/kotlin/androidx/navigationevent/NavigationEventHandler.kt;l=72;drc=c71c22d74fceb646111a069ef577022b155a14b6): what's active now
- [backInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:navigationevent/navigationevent/src/commonMain/kotlin/androidx/navigationevent/NavigationEventHandler.kt;l=82;drc=c71c22d74fceb646111a069ef577022b155a14b6): where “back” would go
- [forwardInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:navigationevent/navigationevent/src/commonMain/kotlin/androidx/navigationevent/NavigationEventHandler.kt;l=92;drc=c71c22d74fceb646111a069ef577022b155a14b6): where “forward” would go

Inspect these to see who's active and what happens next.

In Navigation3, [SceneInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:navigation3/navigation3-ui/src/commonMain/kotlin/androidx/navigation3/scene/SceneInfo.kt;l=32;drc=2dc165ff203eb8e5c9dce39f8ea69bf0d9ec19be) describes your current scene state. Similar support will come to `Navigation2` once it updates to a newer `compileSdk`.

You'll also find [OnBackPressedCallbackInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:activity/activity/src/main/java/androidx/activity/OnBackPressedDispatcher.kt;l=334;drc=b36ffe4ae230e4bb1845223bcbb74e4849952144), [BackHandlerInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:activity/activity-compose/src/main/java/androidx/activity/compose/BackHandler.kt;l=172;drc=13392d2a60ff98309b2e0e73780234f80ef4e0f5), [PredictiveBackHandlerInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:activity/activity-compose/src/main/java/androidx/activity/compose/PredictiveBackHandler.kt;l=275;drc=13392d2a60ff98309b2e0e73780234f80ef4e0f5), [ThreePaneScaffoldSceneInfo](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:compose/material3/adaptive/adaptive-navigation3/src/commonMain/kotlin/androidx/compose/material3/adaptive/navigation3/ThreePaneScaffoldScene.kt;l=322;drc=53b2d4525c319cbc5ebd9cb65f6a85d4fbe5bbfb) , and others. More are coming as libraries migrate.

## Custom Info

You can define your own `Info` to make debugging clearer in your app or library. In Compose:

```kotlin
@Composable
fun Sample() {
  NavigationBackHandler(
    state = rememberNavigationEventState(info = YourCustomInfo()),
    // ...
  )
}
```

## The Bigger Picture

NavigationEvent isn't just a new dispatcher. It's a step toward making back and forward navigation consistent across Android, iOS, Web and Desktop.

The Info API gives you a window into that system: what's happening, who's in control, and where your app is heading next.

> ℹ️ If you enjoyed the article you might enjoy following me on [Bluesky](https://bsky.app/profile/marcellogalhardo.dev). ℹ️
