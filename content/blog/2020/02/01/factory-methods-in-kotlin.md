---
title: "Factory Methods in Kotlin"
date: 2020-02-01T09:02:50+01:00
draft: false
---

When talking about Factory Methods extension functions tends to be favored in Kotlin - but it might be a challenge to name these functions in a discoverable way without polluting your classpath. Therefore, we can get inspiration on Kotlin's Standard library to improve our naming.

## Wrapping an instance

If your intention is to get a given instance, for example `javax.inject.Provider`, create an instance of `ViewModelProvider.Factory` which internally uses it: you want a wrapper. Looking inside Kotlin's Standard library we can find [asExecutor](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/as-executor.html) which does exactly the same.

That said, a good way of doing it is:
```kotlin
fun <VM : ViewModel> Provider<out VM>.asViewModelFactory(): ViewModelProvider.Factory {
    return object : ViewModelProvider.Factory {
        override fun <T : ViewModel?> create(modelClass: Class<T>): T {
            @Suppress("UNCHECKED_CAST")
            return get() as T
        }
    }
}
```

## Mapping data

When your intention is to copy the data of a given instance, for example `UserDto`, into a new instance like `User` (domain model): you want a mapper. Again, Kotlin's Standard library give us a good example with [toList](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/to-list.html?_ga=2.130355144.672183661.1577982073-802284527.1577800392).

```kotlin
fun UserDto.toUser(): User {
    return User(
        name = Name(
            first = this.firstName,
            last = this.lastName
        ),
        email = Email(this.email)
        age = this.age
    )
}
```

## Vararg constructors

Maybe you want to provide a secondary constructor which accept a vararg of that list of parameters but you don't want to pollute your class code which such a small and optional thing. Kotlin's Standard Library offer us the convenient [listOf](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/list-of.html?_ga=2.139095028.672183661.1577982073-802284527.1577800392) as an example.

```kotlin
fun <T : Controller> compositeControllerOf(vararg providers: Provider<T>) = 
    CompositeControllerFactory(providers.toList())
```

## Builders

Last but not least, Kotlin's constructor combined with named parameter covers most use cases of the Builder Design Pattern. However, sometimes you might want to create an instance which constructors are not suitable - for example, with a lot of optionals and configurable attributes where you want to provide alternative combinations (maybe accepting URL or URI). For that cases, we can use [buildString](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/build-string.html) as an example.

```kotlin
inline fun buildImageLoader(
    builderAction: ImageLoaderBuilder.() -> Unit
): String

// usage example
fun main() {
    val validUri = // receives a valid URI object
    buildImageLoader {
        uri = validUri
        loadingPlaceholder = R.drawable.loading_placeholder
        errorPlaceholder = R.drawable.error_placeholder
    }
}
```

# Conclusion

As you can see, the Kotlin Standard library can be a good source of inspiration when designing code aiming for better discoverability between Kotlin Developers through the consistent pattern the language follows. Why not take advantage of that?
