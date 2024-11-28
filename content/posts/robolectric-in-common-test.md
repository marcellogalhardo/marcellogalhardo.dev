---
title: "Robolectric in commonTest"
date: 2024-11-28T16:15:00+01:00
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

Sharing tests across Kotlin Multiplatform (KMP) projects can be tricky when dealing with platform-specific APIs like Android's `Bundle`. `commonTest` on Android relies on the `androidTest` source set, which uses an empty `android.jar`, leading to test failures.

### The Solution

The ideal solution is to avoid platform-specific APIs. However, if that’s not an option, you can use Robolectric in your `commonTest` source set to access functional Android classes. This approach:

- Allows testing of platform-specific code in `commonTest`.
- Eliminates the need to duplicate tests in `androidTest`.

### How It’s Done

1. Add the Robolectric dependency to your `androidTest` :

```groovy
implementation("org.robolectric:robolectric:4.14")
```

2. Use `expect`/`actual` to provide platform-specific `abstract class RobolectricTest`.

In `commonTest`, define the expect class:

```kotlin
expect abstract class RobolectricTest()
```

In `androidTest`, provide the actual implementation using `Robolectric`:

```kotlin
@RunWith(RobolectricTestRunner::class)
@Config(manifest = Config.NONE)
internal actual abstract class RobolectricTest actual constructor()
```

For other platforms, provide an empty actual implementation:

```kotlin
actual abstract class RobolectricTest actual constructor()
```

3. Write your tests in `commonTest` extending the expected declaration.

```kotlin
class SavedStateTest : RobolectricTest()
```

### That’s It

Robolectric will simulate the Android environment and allow a `commonTest` to run platform-specific code.

### References

- [savedstate/savedstate/src/commonTest/kotlin/androidx/savedstate/SavedStateTest.kt](https://cs.android.com/androidx/platform/frameworks/support/+/7dc24cd1eb1889c2b9234a32bb5852dc0d263995:savedstate/savedstate/src/commonTest/kotlin/androidx/savedstate/SavedStateTest.kt;l=23 "https://cs.android.com/androidx/platform/frameworks/support/+/7dc24cd1eb1889c2b9234a32bb5852dc0d263995:savedstate/savedstate/src/commonTest/kotlin/androidx/savedstate/SavedStateTest.kt;l=23")
- [savedstate/savedstate/src/commonTest/kotlin/androidx/savedstate/RobolectricTest.kt](https://cs.android.com/androidx/platform/frameworks/support/+/7dc24cd1eb1889c2b9234a32bb5852dc0d263995:savedstate/savedstate/src/commonTest/kotlin/androidx/savedstate/RobolectricTest.kt;l=19 "https://cs.android.com/androidx/platform/frameworks/support/+/7dc24cd1eb1889c2b9234a32bb5852dc0d263995:savedstate/savedstate/src/commonTest/kotlin/androidx/savedstate/RobolectricTest.kt;l=19")
- [savedstate/savedstate/src/androidUnitTest/kotlin/androidx/savedstate/RobolectricTest.android.kt](https://cs.android.com/androidx/platform/frameworks/support/+/7dc24cd1eb1889c2b9234a32bb5852dc0d263995:savedstate/savedstate/src/androidUnitTest/kotlin/androidx/savedstate/RobolectricTest.android.kt;l=25 "https://cs.android.com/androidx/platform/frameworks/support/+/7dc24cd1eb1889c2b9234a32bb5852dc0d263995:savedstate/savedstate/src/androidUnitTest/kotlin/androidx/savedstate/RobolectricTest.android.kt;l=25")
- [savedstate/savedstate/src/nonAndroidTest/kotlin/androidx/savedstate/RobolectricTest.nonAndroid.kt](https://cs.android.com/androidx/platform/frameworks/support/+/7dc24cd1eb1889c2b9234a32bb5852dc0d263995:savedstate/savedstate/src/nonAndroidTest/kotlin/androidx/savedstate/RobolectricTest.nonAndroid.kt "https://cs.android.com/androidx/platform/frameworks/support/+/7dc24cd1eb1889c2b9234a32bb5852dc0d263995:savedstate/savedstate/src/nonAndroidTest/kotlin/androidx/savedstate/RobolectricTest.nonAndroid.kt")

