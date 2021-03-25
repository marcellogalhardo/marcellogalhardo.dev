---
title: "Naming Factory Methods"
date: 2020-02-01T09:02:50+01:00
draft: false
---

When talking about Factory Methods, extension functions tend to be favored in Kotlin - but it might be a challenge to name these functions in a discoverable way without polluting your project's namespace. A good source of inspiration is Kotlin's Standard library: it contains many examples we can use as a base when deciding how to design a function.

## Wrapping an instance

If you intend to get a given instance and adapt to one different object to follow another contract, for example, creating a `ViewModelProvider.Factory` that internally uses a `javax.inject.Provider`: you want a wrapper. Looking inside Kotlin's Standard library, we can find [asExecutor](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/as-executor.html) which does the same.

Here is an example on how it could look like:
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

## Mapping data

If you intend to get a given instance and copy its data to a different format, for example, a `UserResponse` (DTO) to a `User` (domain entity): you want a mapper. Again, Kotlin's Standard library gives us a good example with [toList](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/to-list.html?_ga=2.130355144.672183661.1577982073-802284527.1577800392).

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

## Vararg constructors

If you need to provide a secondary constructor which accepts an unknown number of elements: you want a collection wrapper. For that, you can use a `vararg` method that returns the new composed object without polluting your class definition with an optional requirement. Kotlin's Standard Library offers us the convenient [listOf](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/list-of.html?_ga=2.139095028.672183661.1577982073-802284527.1577800392) as an example.

```kotlin
fun controllerOf(vararg controllers: Controller): Controller {
    return CompositeController(controllers)
}
```

## Builders

Kotlin's constructor combined with the named parameter covers most use cases of the Builder Design Pattern. However, sometimes you might want to create an instance where constructors are not suitable, with many optional parameters and configurable attributes where developers should be able to create alternative combinations highly configurable. Then, we can look to [buildString](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/build-string.html).

```kotlin
inline fun buildImageLoader(
    builderAction: ImageLoaderBuilder.() -> Unit
): String

data class ImageLoaderBuilder(
    var uri: URI,
    @DrawableRes var loadingPlaceholder: Int?,
    @DrawableRes var loadingPlaceholder: Int?,
    // ...
)

class ImageLoader

// usage example
fun main() {
    val imageLoader = buildImageLoader {
        uri = validUri
        loadingPlaceholder = R.drawable.loading_placeholder
        errorPlaceholder = R.drawable.error_placeholder
        // ...
    }
}
```

## Fake Constructors (aka, Factory Methods)

Sometimes you need to provide a secondary constructor that takes advantage of `reified` or exposes an interface as a concrete type and hides the implementation: we can use a function named a class. Let's have a look into [MutableStateFlow](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/flow/StateFlow.kt#L187).

```kotlin
interface MutableStateFlow<T> {
    var value: T
    fun compareAndSet(expect: T, update: T): Boolean
}

fun <T> MutableStateFlow(value: T): MutableStateFlow<T> {
    return StateFlowImpl(value ?: NULL)
}
```

# Conclusion

As you can see, the Kotlin Standard library can be a good source of inspiration when designing code aiming for better discoverability between Kotlin Developers. Kotlin is a consistent language, and we can use it in our favor. Why not take advantage of that?
