---
title: "Dagger, ViewModels and Fragments"
date: 2020-01-01T09:02:50+01:00
draft: false
---

Dagger is a powerful DI framework but when combined with Architecture Components from Android it might cause some boilerplate: mainly when using ViewModels and Fragments with constructor injection.

Kotlin offers many options to deal with this boilerplate in an elegante way for simple use cases.

#### ViewModelProvider.Factory

Dagger allows you to inject the [Provider](https://docs.oracle.com/javaee/6/api/javax/inject/Provider.html) of any type available on the dependency graph and this provider can be easily mapped to ViewModelProvider.Factory. 

```kotlin
// ViewModelFactory.kt
/**
 * Creates a [ViewModelProvider.Factory] that wraps the original ViewModel [Provider].
 */
fun <VM : ViewModel> Provider<out VM>.asViewModelFactory(): ViewModelProvider.Factory {
    return object : ViewModelProvider.Factory {
        override fun <T : ViewModel?> create(modelClass: Class<T>): T {
            @Suppress("UNCHECKED_CAST")
            return get() as T
        }
    }
}

// MyActivity.kt
class MyActivity : AppCompatActivity() {

    @Inject
    lateinit var viewModelProvider: Provider<MyViewModel>

    private val viewModel by viewModels<MyViewModel> { viewModelProvider.asViewModelFactory() }
}
```

##### Composable ViewModelProvider.Factory

However, sometimes a single ViewModelProvider.Factory is not enough and a composable approach might be needed. Luckly, Dagger also allows you to easily accomplish that.

```kotlin
// ViewModelFactory.kt
internal typealias ViewModelProviderMap = Map<Class<out ViewModel>, @JvmSuppressWildcards Provider<ViewModel>>

/**
 * A [ViewModelProvider.Factory] that can hold onto multiple other ViewModel [Provider]'s.
 *
 * Note this was designed to be used with [ViewModelKey].
 */
class CompositeViewModelFactory @Inject constructor(
    private val map: ViewModelProviderMap
) : ViewModelProvider.Factory {
    override fun <T : ViewModel?> create(modelClass: Class<T>): T {
        @Suppress("UNCHECKED_CAST")
        return map[modelClass]?.get() as T
            ?: throw IllegalStateException("${modelClass.name} cannot be provided without an @Inject constructor or from an @Provides-annotated method. Check your @IntoMap and @ClassKey provider.")
    }
}

/**
 * A [MapKey] annotation for maps with [KClass] of [ViewModel] keys.
 *
 * Note this was designed to be used only with [CompositeViewModelFactory].
 */
@Target(
    AnnotationTarget.FUNCTION,
    AnnotationTarget.PROPERTY_GETTER,
    AnnotationTarget.PROPERTY_SETTER
)
@Retention(value = AnnotationRetention.RUNTIME)
@MapKey
annotation class ViewModelKey(val value: KClass<out ViewModel>)
```

Now, you must do some extra setup to use the CompositeViewModelFactory in your components:

```kotlin
// ViewModelFactoryModule.kt
@Module
interface ViewModelFactoryModule {

    @Binds
    fun provideViewModelFactory(impl: CompositeViewModelFactory): ViewModelProvider.Factory

    @[Binds @IntoMap @ViewModelKey(MyViewModel1::class)]
    fun provideViewModel(impl: MyViewModel1): MyViewModel1

    @[Binds @IntoMap @ViewModelKey(MyViewModel2::class)]
    fun provideViewModel(impl: MyViewModel2): MyViewModel2
}

// MyActivity.kt
class MyActivity : AppCompatActivity() {

    @Inject
    lateinit var viewModelFactory: ViewModelProvider.Factory // no need for a specific VM.Factory

    private val viewModel1 by viewModels<MyViewModel1> { viewModelFactory }
    private val viewModel2 by viewModels<MyViewModel2> { viewModelFactory }
}
```

### FragmentFactory

The same idea can be applied to FragmentFactories for both single and composable versions.

```kotlin
// FragmentFactory.kt
/**
 * Creates a [ViewModelProvider.Factory] that wraps the original ViewModel [Provider].
 */
inline fun <reified F : Fragment> Provider<out F>.asFragmentFactory(): FragmentFactory {
    return object : FragmentFactory() {
        override fun instantiate(classLoader: ClassLoader, className: String): Fragment {
            val fragmentClass = loadFragmentClass(classLoader, className)
            return if (F::class == fragmentClass) {
                get()
            } else {
                super.instantiate(classLoader, className)
            }
        }
    }
}

// MyActivity.kt
class MyActivity : AppCompatActivity() {

    @Inject
    internal lateinit var fragmentProvider: Provider<MyFragment>

    override fun onCreate(savedInstanceState: Bundle?) {
        supportFragmentManager.fragmentFactory = fragmentProvider.asFragmentFactory()
        super.onCreate(savedInstanceState)
    }
}
```

##### Composable FragmentFactory

```kotlin
// FragmentFactory.kt
typealias FragmentProviderMap = Map<Class<out Fragment>, @JvmSuppressWildcards Provider<Fragment>>

/**
 * A [FragmentFactory] that can hold onto multiple other FragmentFactory [Provider]'s.
 *
 * Note this was designed to be used with [FragmentKey].
 */
class CompositeFragmentFactory @Inject constructor(
    private val map: FragmentProviderMap
) : FragmentFactory() {

    override fun instantiate(classLoader: ClassLoader, className: String): Fragment {
        val fragmentClass = loadFragmentClass(classLoader, className)
        return map[fragmentClass]?.get() ?: super.instantiate(classLoader, className)
    }
}

/**
 * A [MapKey] annotation for maps with [KClass] of [Fragment] keys.
 *
 * Note this was designed to be used only with [CompositeFragmentFactory].
 */
@Target(
    AnnotationTarget.FUNCTION,
    AnnotationTarget.PROPERTY_GETTER,
    AnnotationTarget.PROPERTY_SETTER
)
@Retention(value = AnnotationRetention.RUNTIME)
@MapKey
annotation class FragmentKey(val value: KClass<out Fragment>)
```

Similar to the `CompositeViewModelFactory` you must do some additional setup.

```kotlin
// FragmentFactoryModule.kt
@Module
interface FragmentFactoryModule {

    @Binds
    fun provideFragmentFactory(impl: CompositeFragmentFactory): FragmentFactory

    @[Binds @IntoMap @FragmentKey(MyFragment1::class)]
    fun provideFragment(impl: MyFragment1): MyFragment1

    @[Binds @IntoMap @FragmentKey(MyFragment2::class)]
    fun provideFragment(impl: MyFragment2): MyFragment2
}

// MyActivity.kt
class MyActivity : AppCompatActivity() {

    @Inject
    internal lateinit var fragmentFactory: FragmentFactory // no need for a specific Factory

    override fun onCreate(savedInstanceState: Bundle?) {
        supportFragmentManager.fragmentFactory = fragmentFactory
        super.onCreate(savedInstanceState)
    }
}
```

#### Conclusion

As you can see it is easy to support the Architecture Components combining Dagger with Kotlin features. I personally prefer the composite approach where we can fully hide the implementation details from the production code - Activities and Fragments aren't aware of the `javax.inject.Provider` interface - and the "glue" code is done inside Dagger modules with @ClassKey annotation style. Therefore, the additional might be an overkill for use cases where you want only one ViewModel or Fragment and these extension functions can be very handy if you don't care to use the Provider directly.

**Disclaimer:** Note that you must connected the `CompositeViewModelFactoryModule` and `CompositeFragmentFactoryModule` to your Component. I propositally ommited this part to keep this tutorial short - if you need more information, check the [Dagger documentation](https://dagger.dev/).