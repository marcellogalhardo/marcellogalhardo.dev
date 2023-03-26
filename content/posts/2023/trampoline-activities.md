---
title: "Trampoline Activities"
date: 2023-03-06T09:02:50+01:00
draft: false
toc: false
images:
categories:
  - software development
tags:
  - android
---

Today, I came across the term "Trampoline Activities". Although it's not an official name ([or is it](https://developer.android.com/about/versions/12/behavior-changes-12#notification-trampolines)?), I've noticed it is a typical pattern and have decided to make a "How To" guide.

## What are Trampoline Activities?

A Trampoline Activity is an `Activity` that launches another activity and finishes itself. It may include conditional logic to determine which activity to launch or transforming the parameters before sending it to the next `Activity`.

Here is an example: [Trampoline Activity](https://android.googlesource.com/platform/packages/apps/ManagedProvisioning/+/b949b1f/src/com/android/managedprovisioning/TrampolineActivity.java).

## How to create a Trampoline Activity?

To create a Trampoline Activity, you need an activity without a UI that closes itself and moves on to the next activity. The key is to use the `NoDisplay` theme to remove the UI:

```xml
<activity
    android:name=".MyTrampolineActivity"
    android:theme="@android:style/Theme.NoDisplay" />
```

You can write your logic in the `onCreate` method of `MyTrampolineActivity` and launch the next activity as soon as possible. Remember to finish the `MyTrampolineActivity` at the end of the `onCreate` method:

```kotlin
class MyTrampolineActivity : Activity() {
  fun onCreate() {
    super.onCreate()
    // Start your next activity.
    finish()
  }
}
```

Note that the `NoDisplay` theme means no UI, including toasts or dialogs. If you don't call `finish()` before `onResume()` completes, the activity will crash on any Android 23+ device. The behaviour is intentional, as you can see [here](https://web.archive.org/web/20151116170752/https://code.google.com/p/android-developer-preview/issues/detail?id=2353) and here:

> PSA: The new requirement to immediately finish an activity if using Theme.NoDisplay is not a regression, this has always been a requirement of it (see https://developer.android.com/reference/android/R.style.html#Theme_NoDisplay for example).
>
> The reason the platform in M is now crashing the app if it doesn't use this is because not using it would previously break in very subtle and mysterious ways. For example, you would sometimes end up with your app ANRing for no reason.

If you want a transparent `Activity`, consider using the `Translucent` theme instead.

---

## When should you use a Trampoline Activity?

Avoid using Trampoline Activities in every situation (or even better, use a [Single Activity](https://www.youtube.com/watch?v=2k8x8V77CrU)) and always consider alternative solutions before relying on them. However, there are some scenarios in which it can be helpful, including:
* When you need to access Activity context APIs, from a different context.
* When you want to transform or sanitize parameters, before passing them forward to the next Activity.
* When you need to perform conditional routing logic between multiple activities.

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions, feel free to reach out to me! ℹ️
