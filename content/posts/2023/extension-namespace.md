---
title: "Extension Namespace"
date: 2023-03-25T09:02:50+01:00
draft: true
toc: false
images:
categories:
  - software development
tags:
  - kotlin
---

A few days ago, I had to create an extension function - a prevalent task for any Kotlin developer. But there were a few problems:
* The receiver was a common type, polluted with methods.
* The extension function was only relevant to my feature package.
* Creating a ~Gradle~ module was out of scope.
* Introducing a new type to hold the function felt too much.

Trying to be a good citizen, I asked myself (not literally): how can I have the advantages of using extension functions but avoid these issues?

One way is to use objects to namespace extension functions.

```kotlin
// MyFeatureActivityManagers.kt
// Namespace object.
object MyFeatureActivityManagers {

  // Extension functions that are only interested to my feature.
  fun ActivityManager.doSomethingThatOnlyYouCare() = TODO("")
  fun ActivityManager.doSomethingThatOnlyYouCareToo() = TODO("")
}
```

Now to use:

```kotlin
import MyFeatureActivityManagers.doSomethingThatOnlyYouCare

fun myFeature(activityManager: ActivityManager) {
    activityManager.doSomethingThatOnlyYouCare()
}
```

The approach has the following advantages:
* It ensures that code in the same package can only access the extension function with a direct import;
* It helps humans (and tools) to identify where the extension is coming from;
* But most important, it has the right level of discoverability*.

Discoverability? What do you mean?

It seems IDEA has a clear hierarchy for suggesting auto-complete in the following order:
- Member functions
- Global extensions
- Object extensions

Hence, IDEA will only suggest to other developers your shiny function if they are actively looking for it.

---

**What about the naming of your namespace object?**

Ideally, I want to indicate why these particular functions are related, given the infinite number of possible extensions I could write.

A pattern I find helpful is: {Context}{Receiver}s, where:

- `Context` groups the functions based on their relevance.
- `Receiver` refers to the type of the receiver for these functions.
- `s` represents the collection of extensions.

---

If you find yourself creating extensions that should be limited in access, consider creating a namespace object for them.

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions, feel free to reach out to me! ℹ️
