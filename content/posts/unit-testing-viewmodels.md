---
title: "Unit Testing ViewModels"
date: 2025-02-22T22:22:22+22:22
draft: false
toc: false
images:
- /logo.png
categories:
- software development
tags:
  - android
  - kotlin
---

Lifecycle [2.9.0-alpha01](https://developer.android.com/jetpack/androidx/releases/lifecycle#2.9.0-alpha01) introduced [ViewModelScenario](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=128-131 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=128-131"), a helper that simplifies unit testing for [ViewModels](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModel.kt;l=99 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModel.kt;l=99").

## Why It Matters

You can test a [`ViewModel`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModel.kt;l=99) by simply creating an instance using its constructor in your test code. However, this approach has limitations — there is no straightforward way to:

- Trigger [`ViewModelStore.clear()`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModelStore.kt;l=56)/[`ViewModel.onCleared()`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModel.kt;l=167).
- Simulate system process death and state restoration.

With [`ViewModelScenario`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=128-131), these scenarios are now easy to test, helping you catch errors related to ViewModel clean-up and saved state.

## How to Use

First, add the dependency to your Gradle file:

```groovy
implementation("androidx.lifecycle:lifecycle-viewmodel-testing:2.9.0-alpha10")
```

Next, define your ViewModel:

```kotlin
class MyViewModel(
  scope: CoroutineScope,
  private val handle: SavedStateHandle,
) : ViewModel(scope) {
  // TODO("not implemented")
}
```

Now, write a unit test for your `ViewModel` using a `ViewModelScenario`:

```kotlin
class MyViewModelTest {

  @Test
  fun testStateRestoration() = runTest { // this: TestScope
    viewModelScenario { // this: CreationExtras
        MyViewModel(
          scope = this@runTest,
          handle = createSavedStateHandle(),
        )
    }.use { viewModel: MyViewModel ->
      // Set a ViewModel state.
      recreate()
      // Assert state is restored correctly.
    }
  }
}
```

## Test in Details

Here’s the cleaned-up version with fixed markdown links and no bold text:

- [`ViewModelScenario.recreate()`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=92) simulates Android’s state saving and restoring, including parceling (see [Save UI states](https://developer.android.com/topic/libraries/architecture/saving-states#onsaveinstancestate)).
  - It verifies that data in [`SavedStateHandle`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-savedstate/src/androidMain/kotlin/androidx/lifecycle/SavedStateHandle.android.kt;l=30) is correctly persisted.
- [`AutoCloseable.use {}`](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-auto-closeable.html) ensures [`ViewModelStore.clear()`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModelStore.kt;l=56)/[`ViewModel.onCleared()`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModel.kt;l=167) runs after the test.
  - You can manually call [`ViewModelScenario.close()`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=78-80) to manually trigger clearing.
- Also, [`ViewModelScenario`](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=128-131) is KMP compatible.

> ℹ️ If you enjoyed the article you might enjoy following me on [Bluesky](https://bsky.app/profile/marcellogalhardo.dev). ℹ️
