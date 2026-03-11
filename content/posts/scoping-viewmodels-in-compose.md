---
title: "Scoping ViewModels in Compose"
date: 2026-03-11T18:51:00+01:00
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
  - compose
---

Lifecycle ViewModel [2.11.0-alpha02](https://developer.android.com/jetpack/androidx/releases/lifecycle#2.11.0-alpha02) introduces `rememberViewModelStoreOwner`, an API to scope `ViewModelStore` directly within the Compose hierarchy.

## Why It Matters

Until now, `ViewModelStore` scoping was tied to navigation destinations, activities or fragments. There was no clean way to scope a `ViewModel` to an arbitrary part of your UI (such as a `Pager` page, a `LazyList` item, or a custom layout) without building your own `ViewModelStoreOwner` from scratch.

These new APIs close that gap:
- [rememberViewModelStoreOwner](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-viewmodel-compose/src/commonMain/kotlin/androidx/lifecycle/viewmodel/compose/RememberViewModelStoreOwner.kt;l=63;drc=72a763a5476f8849620a613c95a98ef350f6f11f)
- [rememberViewModelStoreProvider](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-viewmodel-compose/src/commonMain/kotlin/androidx/lifecycle/viewmodel/compose/RememberViewModelStoreProvider.kt;l=58;drc=12c90beccf79459db81e733e08df5754ffa7c81c)

## Basic Usage

```kotlin
// Parent-scoped ViewModel — behaves as before.
val parentViewModel = viewModel<ParentViewModel>()

// Create a child-scoped owner.
val childOwner = rememberViewModelStoreOwner()

// ViewModel scoped to the child owner.
val childViewModel = viewModel<ChildViewModel>(childOwner)

// Optionally propagate the owner down the composition tree.
CompositionLocalProvider(LocalViewModelStoreOwner provides childOwner) {
    // Composables here resolve ViewModels from childOwner.
}
```

The child ViewModelStore is cleared when its owner exits the composition (unless triggered by a configuration change) and its lifespan mirrors its position in the UI tree.

## Hoisting State

For cases where you need per-page or per-item `ViewModel` scoping, use `rememberViewModelStoreProvider` to hoist the owner management:

```kotlin
val storeProvider = rememberViewModelStoreProvider()

HorizontalPager(pageCount = pages.size) { page ->
    val pageOwner = storeProvider.rememberViewModelStoreOwner(page)

    CompositionLocalProvider(LocalViewModelStoreOwner provides pageOwner) {
        val pageViewModel = viewModel<PageViewModel>()
        PageContent(pageViewModel)
    }
}
```

The same approach works for any composable that needs its own scoped `ViewModel` state, such as a `LazyList`.

## What Gets Propagated

When not explicitly provided, the child owner inherits the following from the parent scope:

* `savedStateRegistryOwner: SavedStateRegistryOwner`
* `defaultArgs: SavedState`
* `defaultFactory: ViewModelProvider.Factory`
* `defaultCreationExtras: CreationExtras`

This means Hilt-injected ViewModels, saved state, and custom factories work as expected in child-scoped owners without any additional wiring.

## Nested Saved State

For advanced use cases requiring nested `SavedStateRegistry` scopes, `rememberViewModelStoreOwner` can be combined with the Compose `SaveableStateHolder` API:

```kotlin
val saveableStateHolder = rememberSaveableStateHolder()
val storeProvider = rememberViewModelStoreProvider()

HorizontalPager(pageCount = pages.size) { page ->
    saveableStateHolder.SaveableStateProvider(page) {
        val pageOwner = storeProvider.rememberViewModelStoreOwner(page)
        CompositionLocalProvider(LocalViewModelStoreOwner provides pageOwner) {
            val pageViewModel = viewModel<PageViewModel>()
            PageContent(pageViewModel)
        }
    }
}

```

Each page receives its own `SavedStateRegistry` nested within the parent, fully integrated with the child `ViewModelStore` scope.

## Feedback Welcome

The API is still in alpha. If you have use cases that are not covered or encounter issues, the [issue tracker](https://issuetracker.google.com/issues/new?component=413132&template=1096619) is the best place to share feedback.

> ℹ️ If you enjoyed the article you might enjoy following me on [Bluesky](https://bsky.app/profile/marcellogalhardo.dev). ℹ️
