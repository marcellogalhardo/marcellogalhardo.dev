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

You can test a [ViewModel](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModel.kt;l=99) by creating an instance with its constructor in your test code. But there wasn’t an easy way to:

- Trigger `ViewModelStore.clear()` / `ViewModel.onCleared()`.
- Simulate a system process death state restoration.

With [ViewModelScenario](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=128-131 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=128-131"), you can now catch errors related to these cases.

## How to Use

```kotlin
class MyViewModel(
  scope: CoroutineScope,
  private val handle: SavedStateHandle,
) : ViewModel(scope) {
  // TODO("not implemented")
}

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

- [ViewModelScenario.recreate()](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=92 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=92") simulates Android’s state saving and restoring, including parceling (see [Save UI states](https://developer.android.com/topic/libraries/architecture/saving-states#onsaveinstancestate).
    - It verifies that data in [SavedStateHandle](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-savedstate/src/androidMain/kotlin/androidx/lifecycle/SavedStateHandle.android.kt;l=30 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-savedstate/src/androidMain/kotlin/androidx/lifecycle/SavedStateHandle.android.kt;l=30") is correctly persisted.
- `[AutoCloseable.use {}](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-auto-closeable.html "https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/-auto-closeable.html")` ensures [ViewModelStore.clear()](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModelStore.kt;l=56 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModelStore.kt;l=56") / [ViewModel.onCleared()](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModel.kt;l=167 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel/src/commonMain/kotlin/androidx/lifecycle/ViewModel.kt;l=167") runs after the test.
    - You can manually call `[ViewModelScenario.close()](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=78-80 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=78-80")` to manually trigger clearing.
- Also, [ViewModelScenario](https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=128-131 "https://cs.android.com/androidx/platform/frameworks/support/+/a775989d0657e5fcbd86bf7949d95a190deb2334:lifecycle/lifecycle-viewmodel-testing/src/commonMain/kotlin/androidx/lifecycle/viewmodel/testing/ViewModelScenario.kt;l=128-131") is KMP compatible.
