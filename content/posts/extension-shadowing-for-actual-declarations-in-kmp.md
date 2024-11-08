---
title: "Extension Shadowing for Actual Declarations in KMP"
date: 2024-11-07T15:49:00+01:00
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

**Heads-up:** this article assumes familiarity with Kotlin's [extension functions](https://kotlinlang.org/docs/extensions.html) and [expect and actual declarations](https://kotlinlang.org/docs/multiplatform-expect-actual.html) in [Kotlin Multiplatform (KMP)](https://kotlinlang.org/docs/multiplatform.html).

My work has recently focused on "commonizing"[^1] APIs, and I came across [KT-70012](https://youtrack.jetbrains.com/issue/KT-70012/EXTENSIONSHADOWEDBYMEMBER-shouldnt-be-reported-for-actual-declarations), which I believe merits attention.

In Kotlin JVM development, the `EXTENSION_SHADOWED_BY_MEMBER` warning indicates that an extension function is redundant, as it will always be overshadowed by a member function with the same name when invoked. However, in [KMP](https://kotlinlang.org/docs/multiplatform.html), this behaviour can be useful, as shadowing may occur on some platforms but not all.

Starting with Kotlin 2.1.0-Beta1, this warning will be removed in such cases. While this won't change any behaviour, it will validate shadowing as a legitimate use case for [KMP](https://kotlinlang.org/docs/multiplatform.html).

This allows for effective strategies when commonizing existing APIs that need to remain backwards compatible, especially those not initially developed with Kotlin in mind.

Let's explore a naive use case for Android: [Bundle.putBoolean(String?, Boolean)](https://developer.android.com/reference/android/os/BaseBundle#putBoolean(java.lang.String,%20boolean))

```kotlin
// Common
expect class Bundle
expect fun Bundle.putBoolean(key: String, value: Boolean)

// Android
actual typealias Bundle = android.os.Bundle
actual inline fun Bundle.putBoolean(key: String, value: Boolean) {
  error("Extension shadowed by member")
}

// Non-Android
actual class Bundle(map: Map<String, Any?>)
actual fun Bundle.putBoolean(key: String, value: Boolean) {
    map[key] = value
}
```

Let’s break this down:

1. `expect class Bundle` is an opaque type with no members in `commonMain`.
2. `expect fun Bundle.putBoolean(key, value)` matches the signature of the original `putBoolean` method, but changes the first parameter from `String?` to `String`.

With this setup:

- On Android, `putBoolean` won't exist at runtime and will always be superseded by the original member function.
- On other platforms, the extension function will be utilized instead.
- It enables stricter types for shadowed members (e.g., changing `key: String?` to `key: String` since the types are compatible).
- The new type alias remains backward compatible with existing APIs.

![extension-shadowing-for-actual-declarations-in-kmp](/images/extension-shadowing-for-actual-declarations-in-kmp.png)

The `EXTENSION_SHADOWED_BY_MEMBER` warning is expected to be removed in Kotlin version 2.1.0. I hope to see more documentation on extension shadowing for actual declarations and clarification on what we can expect from the final bytecode on platforms where methods are shadowed — will the method be fully removed, or will it simply adjust the namespace?

For now, however, it remains an interesting concept to explore in side projects.

[^1]: Commonization refers to making an API available in the Kotlin Multiplatform `common` source sets across multiple platforms.
