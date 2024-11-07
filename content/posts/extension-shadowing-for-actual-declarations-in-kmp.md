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
---

**Heads-up:** this article assumes familiarity with Kotlin's [extension functions][extension-fun] and [expect and actual declarations][expect-actual] in [Kotlin Multiplatform (KMP)][kmp].

My work has recently focused on "commonizing"[^1] APIs, and I came across [KT-70012][KT-70012], which I believe merits attention.

In Kotlin JVM development, the `EXTENSION_SHADOWED_BY_MEMBER` warning indicates that an extension function is redundant, as it will always be overshadowed by a member function with the same name when invoked. However, in [KMP][kmp], this behaviour can be useful, as shadowing may occur on some platforms but not all.

Starting with Kotlin 2.1.0-Beta1, this warning will be removed in such cases. While this won't change any behaviour, it will validate shadowing as a legitimate use case for [KMP][kmp].

This allows for effective strategies when commonizing existing APIs that need to remain backwards compatible, especially those not initially developed with Kotlin in mind.

Let's explore a naive use case for Android: [Bundle.putBoolean(String?, Boolean)][bundle-putboolean]

```kotlin
// Common
expect class Bundle
expect fun Bundle.putBoolean(key: String, value: Boolean)

// Android
actual typealias Bundle = android.os.Bundle
actual inline fun Bundle.putBoolean(key: String, value: Boolean) {
  putBoolean(key, value)
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
3. On Android, the actual extension function is an inline function and will not exist at runtime.
4. On non-Android platforms, the actual extension function is a regular function.

With this setup:

- On Android, `putBoolean` won't exist at runtime and will always be superseded by the member function.
- On other platforms, the extension function will be utilized instead.
- It enables stricter types for shadowed members (e.g., changing `key: String?` to `key: String` since the types are compatible).
- The new type alias remains backwards compatible with existing APIs.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wmvc69m96nl1df0zcm1b.png)

The `EXTENSION_SHADOWED_BY_MEMBER` warning is expected to be removed in Kotlin version 2.1.0. I hope to see more documentation on extension shadowing for actual declarations and clarification on what we can expect from the final bytecode on platforms where methods are shadowed — will the method be fully removed, or will it simply adjust the namespace?

For now, however, it remains an interesting concept to explore in side projects.

**References**
- [KT-70012][KT-70012]
- [KT-69799][KT-69799]

[^1]: Commonization refers to making an API available in the Kotlin Multiplatform `common` source sets across multiple platforms.
[kmp]: https://kotlinlang.org/docs/multiplatform.html
[expect-actual]: https://kotlinlang.org/docs/multiplatform-expect-actual.html
[extension-fun]: https://kotlinlang.org/docs/extensions.html
[KT-70012]: https://youtrack.jetbrains.com/issue/KT-70012/EXTENSIONSHADOWEDBYMEMBER-shouldnt-be-reported-for-actual-declarations
[KT-69799]: https://youtrack.jetbrains.com/issue/KT-69799#focus=Comments-27-10153878.0-0
[bundle-putboolean]: https://developer.android.com/reference/android/os/BaseBundle#putBoolean(java.lang.String,%20boolean)
