---
title: "Namespace for Extension Functions"
date: 2023-03-25T09:02:50+01:00
draft: false
toc: false
images:
categories:
  - software development
tags:
  - kotlin
---

A few weeks ago, I had to create an extension function - a usual task for any Kotlin developer. But there were a few limitations:
* The receiver was a common type, polluted with methods.
* The extension function was only relevant to my feature package.
* Creating a ~~Gradle~~ module was out of scope.
* Introducing a new type to hold the function felt like too much.

So how can I have the advantages of using extension functions but avoid these issues?

One way is to use objects as namespace for extension functions.

Here is an example:

```kotlin
// feature/MyFeatureActivityManagers.kt

// Namespace object.
object MyFeatureActivityManagers {

  // Extension functions that are only interested to my feature.
  fun ActivityManager.doSomethingThatOnlyYouCare() = TODO("")
  fun ActivityManager.doSomethingThatOnlyYouCareToo() = TODO("")
}
```

And here is the usage:

```kotlin
// feature/MyFeature.kt

import MyFeatureActivityManagers.doSomethingThatOnlyYouCare

fun myFeature(activityManager: ActivityManager) {
    activityManager.doSomethingThatOnlyYouCare()
}
```

It didn't change much but the approach has a few advantages:
* It ensures that code in the same package can only access the extension function with a direct import;
* It helps humans (and tools) to identify what file the extension is coming from;
* But most important, it has the right level of discoverability*.

*: IntelliJ IDEA has a hierarchy for suggesting auto-complete in the following order:
- Member functions
- Global extensions
- Object extensions

Hence, IDEA will only suggest to other developers your shiny function if they are actively looking for it.

---

**What's up with the naming?**

I want to indicate why these particular functions are related, given the infinite number of possible extensions I could write.

A pattern I find useful is: {Context}{Receiver}s, where:

- `Context` groups the functions based on their relevance.
- `Receiver` refers to the type of the receiver for these functions.
- `s` represents the collection of extensions.

---

If you find yourself creating extensions that should be limited in access, consider creating a namespace object for them.

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions or I missed something, feel free to reach out to me! ℹ️
