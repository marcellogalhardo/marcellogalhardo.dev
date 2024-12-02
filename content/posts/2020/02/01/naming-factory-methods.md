---
title: "Naming Factory Methods"
date: 2020-02-01T09:02:50+01:00
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

When discussing Factory Methods, extension functions are often preferred in Kotlin. However, naming these functions in a discoverable way without cluttering your project's namespace can be challenging. A great source of inspiration is Kotlin's Standard Library, which offers numerous examples to guide function design.

## Wrapping an Instance

If you want to take an existing instance and adapt it to a different object to meet another contract—such as creating a `ViewModelProvider.Factory` that internally uses a `javax.inject.Provider`—you’ll need a wrapper. An example from Kotlin's Standard Library is [asExecutor](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/as-executor.html), which performs a similar role.

Here’s an example of how such a wrapper could look:

```kotlin
fun <VM : ViewModel> Provider<out VM>.asViewModelProviderFactory(): ViewModelProvider.Factory {
    return object : ViewModelProvider.Factory {
        override fun <T : ViewModel?> create(modelClass: Class<T>): T {
            @Suppress("UNCHECKED_CAST")
            return get() as T
        }
    }
}
```

## Mapping Data

When you need to transform an instance by copying its data into a different format—like converting a `UserResponse` (DTO) into a `User` (domain entity) — a mapper is the right tool. Kotlin's Standard Library provides a great example with [toList](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/to-list.html).

```kotlin
fun UserDto.toUser(): User {
    return User(
        name = Name(
            first = this.firstName,
            last = this.lastName
        ),
        email = Email(this.email),
        age = this.age
    )
}
```

## Vararg Constructors

If you need a secondary constructor that accepts an unknown number of elements, use a collection wrapper. A `vararg` method allows you to create a new object without overloading your class definition. Kotlin's Standard Library provides the [listOf](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/list-of.html) function as an example.

```kotlin
fun controllerOf(vararg controllers: Controller): Controller {
    return CompositeController(controllers)
}
```

## Builders

Kotlin’s constructors combined with named parameters cover most use cases for the Builder Design Pattern. However, when creating instances with many optional parameters or highly configurable attributes, builders become necessary. The [buildString](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/build-string.html) function is a good reference.

```kotlin
inline fun buildImageLoader(
    builderAction: ImageLoaderBuilder.() -> Unit
): String

data class ImageLoaderBuilder(
    var uri: URI,
    @DrawableRes var loadingPlaceholder: Int?,
    @DrawableRes var errorPlaceholder: Int?
    // ...
)

// Usage example
fun main() {
    val imageLoader = buildImageLoader {
        uri = validUri
        loadingPlaceholder = R.drawable.loading_placeholder
        errorPlaceholder = R.drawable.error_placeholder
        // ...
    }
}
```

## Fake Constructors (Factory Methods)

Sometimes, you need a secondary constructor that leverages `reified` types or provides a concrete type while exposing an interface. In such cases, a function named after the class serves as a Factory Method. Consider this example from [MutableStateFlow](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/flow/StateFlow.kt#L187).

```kotlin
interface MutableStateFlow<T> {
    var value: T
    fun compareAndSet(expect: T, update: T): Boolean
}

fun <T> MutableStateFlow(value: T): MutableStateFlow<T> {
    return StateFlowImpl(value ?: NULL)
}
```

## Conclusion

As demonstrated, Kotlin’s Standard Library is an excellent resource for designing code that enhances discoverability for Kotlin developers. Kotlin’s consistency can be a powerful advantage — why not leverage it?

---

> ℹ️ If you enjoyed the article you might enjoy following me on [Bluesky](https://bsky.app/profile/marcellogalhardo.dev). ℹ️
