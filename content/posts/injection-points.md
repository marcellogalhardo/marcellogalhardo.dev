---
title: "Injection Points"
date: 2023-06-02T09:12:00+01:00
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

Android has made significant progress in becoming a [DI-Friendly Framework](https://blog.ploeh.dk/2014/05/19/di-friendly-framework/). Throughout the years, new APIs like `AppComponentFactory` and `FragmentFactory` have been introduced, allowing apps to incorporate their own custom constructors and facilitating the development of testable code.

The purpose of this article is to highlight some of the Android APIs that are utilised behind the scenes. By exploring these APIs, developers can gain a better understanding of the underlying mechanisms employed by popular libraries (such as [Dagger Hilt](https://developer.android.com/training/dependency-injection/hilt-android) and [Koin Android](https://insert-koin.io/docs/quickstart/android/)).

Please keep in mind that I will not delve into the concept of DI, SL, specific libraries or its significance. Instead, the focus will be solely on the Android APIs that enable constructor injection. If you are interested in learning more about the concepts of DI, I recommend exploring resources on [Dependency Injection in Android](https://developer.android.com/training/dependency-injection).

## AppComponentFactory

Starting from [API Level 28](https://developer.android.com/reference/android/app/AppComponentFactory), you can use the `AppComponentFactory` abstract class to customise the instantiation of application component objects declared in your manifest, such as `Application`, `Activity`, `Service`, and `ContentProvider`.

Here's how you can use it.

1. Create a new class that extends `AppComponentFactory`, and override the appropriate methods.

```kotlin
class MyAppComponentFactory : AppComponentFactory() {

    // Customize the instantiation of Application class here.
    override fun instantiateApplication(
        classLoader: ClassLoader,
        className: String,
    ): Application = when (className) {
        MyApplication::class.java.name -> MyApplication()

        // Fallback to Android's default implementation.
        else -> super.instantiateApplication(cl, className)
    }

    // Customize the instantiation of Activity class here.
    override fun instantiateActivity(
        classLoader: ClassLoader,
        className: String,
        intent: Intent,
    ): Activity = when (className) {
        MyActivity::class.java.name -> MyActivity()

        // Fallback to Android's default implementation.
        else -> super.instantiateActivity(cl, className, intent)
    }

    // Override other `instantiate*` methods here.
}
```

2. Declare your custom `AppComponentFactory` in the `AndroidManifest.xml` file by adding the `android:appComponentFactory` attribute to the `<application>` tag.

```xml
<application android:appComponentFactory=".MyAppComponentFactory" />
```

**Disclaimer:** contrary to the belief of many, the latest Android API Version Distribution, last updated on January 6th, 2023, reveals that API 28 and above comprise a significant majority of the Android market, accounting for approximately 81%.

## LayoutInflater.Factory and Views

Starting from [API Level 11](https://developer.android.com/reference/android/view/LayoutInflater.Factory2), you can use the `LayoutInflater.Factory2` interface to customise the inflation of a `View` instance.

Here's how you can use it.

1. Create a new class that implements `LayoutInflater.Factory2`, and override the appropriate methods.

```kotlin
class MyLayoutInflaterFactory : LayoutInflater.Factory2 {

    // Create and return a custom View based on the parent, name, context, and attributes.
    override fun onCreateView(
        parent: View?,
        name: String,
        context: Context,
        attrs: AttributeSet,
    ): View? = when (name) {
        MyCustomView::class.java.name -> MyCustomView(context, attrs, /* defStyleAttr = */ null)

        // Return null to fallback to Android's default implementation.
        else -> null
    }

   override fun onCreateView(
      name: String,
      context: Context,
      attrs: AttributeSet?,
   ): View? = onCreateView(/* parent = */ null, name, context, attrs)
}
```

2. Set your custom `LayoutInflater.Factory2` on your `Activity` .

```kotlin
class MyActivity : Activity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        layoutInflater.factory2 = MyLayoutInflaterFactory()
        super.onCreate(savedInstanceState)

        // Your activity initialization and logic here.
    }

    // Other overridden methods and custom methods here.
}
```

### AppCompat Exception

Setting a custom `LayoutInflater.Factory2` will crash if you use `AppCompat` (for example, `AppCompatActivity`).

> java.lang.IllegalStateException: A factory has already been set on this LayoutInflater.
>

That is by design. `AppCompat` relies on a custom `LayoutInflater.Factory2` that must be set to work as expected.

Here's a workaround.

1. Remove the previous code from from your Activity's `onCreate`.
2. Create a new class that implements `LayoutInflater.Factory2`, and receives 2 `LayoutInflater.Factory2` as parameters:
   - `base` to handle your custom views with constructor injector.
   - `fallback` which will be used to handle all other views.

```kotlin
class LayoutInflaterFactory2Wrapper(
    val base: LayoutInflater.Factory2?,
    val fallback: LayoutInflater.Factory2?,
) : LayoutInflater.Factory2 {

    override fun onCreateView(
        parent: View?,
        name: String?,
        context: Context?,
        attrs: AttributeSet?,
    ): View? = base?.onCreateView(parent, name, context, attrs)
        ?: fallback?.onCreateView(parent, name, context, attrs)

    override fun onCreateView(
        name: String,
        context: Context,
        attrs: AttributeSet?,
    ): View? = onCreateView(/* parent = */ null, name, context, attrs)
}
```

3. Create a new class that extends `ContextWrapper`, and override the `getSystemService` method.

```kotlin
class MyContextWrapper(
    base: Context,
    private val baseFactory: LayoutInflater.Factory2,
) : ContextWrapper(base) {

    private val inflater: LayoutInflater by lazy {
        LayoutInflater
            .from(baseContext)
            .cloneInContext(/* newContext = */ this)
            .apply {
               factory = LayoutInflaterFactory2Wrapper(baseFactory, factory2)
            }
    }

    override fun getSystemService(name: String): Any? =
        if (LAYOUT_INFLATER_SERVICE == name) {
           inflater
        } else {
           super.getSystemService(name)
        }
}
```

4. Set your custom `ContextWrapper` on your `Activity` .

```kotlin
class MyActivity : AppCompatActivity() {

    override fun attachBaseContext(newBase: Context) {
        val baseFactory = MyLayoutInflaterFactory()
        val wrapper = MyContextWrapper(baseFactory, fallback = LayoutInflater.from(newBase))
        super.attachBaseContext(wrapper);
    }
}
```

# FragmentFactory

Starting from [androidx.fragment version 1.1.0-alpha01](https://developer.android.com/jetpack/androidx/releases/fragment#1.1.0-alpha01), you can use the `FragmentFactory` abstract class to customise the creation of a `Fragment` instance.

Here's how you can use it.

1. Create a new class that extends `FragmentFactory`, and override the appropriate methods.

```kotlin
class MyFragmentFactory : FragmentFactory() {

    override fun instantiate(
        classLoader: ClassLoader,
        className: String,
    ): Fragment = when (className) {
        MyFragment::class.java.name -> MyFragment()

        // Fallback to Android's default implementation.
        else -> super.instantiate(classLoader, className)
    }
}
```

2. Set your custom `FragmentFactory` instance on your `FragmentActivity` (or `Fragment`) before any fragment operations are performed.

```kotlin
class MyActivity : FragmentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        supportFragmentManager.fragmentFactory = MyFragmentFactory()
        super.onCreate(savedInstanceState)

        // Your activity initialization and logic here.
    }

    // Other overridden methods and custom methods here.
}
```

## ViewModel

Starting from [androidx.lifecycle version 2.5.0-alpha01](https://developer.android.com/jetpack/androidx/releases/lifecycle#2.5.0-alpha01), you can use the `ViewModelProvider.Factory` with the `CreationExtras` to simplify the `ViewModel` creation (for example, `SavedStateHandle`).

Here's how you can use it.

1. Create a new class that implements `ViewModelProvider.Factory`, and override the appropriate methods.

```kotlin
class MyViewModelFactory : ViewModelProvider.Factory {

    @Suppress("UNCHECKED_CAST")
    override fun <T : ViewModel> create(
        modelClass: Class<T>,
        extras: CreationExtras,
    ): T = when (modelClass) {
        MyViewModel::class.java -> MyViewModel(extras.createSavedStateHandle())
        else -> error("Unknown ViewModel class: $modelClass")
    } as T
}
```

2. When you need to create an instance of `MyViewModel`,  use the `MyViewModelFactory`.

```kotlin
val viewModel by viewModels { MyViewModelFactory() }
```

## WorkManager and WorkerFactory

Starting from [androidx.work version 1.0.0-alpha09](https://developer.android.com/jetpack/androidx/releases/work#1.0.0-alpha09), you can use the `WorkerFactory` interface to customise the creation of an `Worker` instance.

Here's how you can use it.

1. [Remove the default initialiser](https://developer.android.com/guide/background/persistent/configuration/custom-configuration#remove-default).
2. Create a custom `WorkerFactory` implementation.

```kotlin
class MyWorkerFactory : WorkerFactory() {

   // Customize the creation of your worker based on the workerClassName and workerParameters.
   override fun createWorker(
      appContext: Context,
      workerClassName: String,
      workerParameters: WorkerParameters,
   ): ListenableWorker? = when (workerClassName) {
      MyWorker::class.java.name -> MyWorker(appContext, workerParameters)

      // Return null to fallback to Android's default implementation.
      else -> null
   }
}
```

In the `createWorker` method, you can inspect the `workerClassName` and `workerParameters` to determine the appropriate worker instance to create. Return `null` if you don't want to handle the given `workerClassName` and let the default factory handle it.

3. Set the custom `WorkerFactory` to WorkManager: In your application's initialisation code, before using WorkManager, set your custom `WorkerFactory` using `Configuration`:

```kotlin
val configuration = Configuration.Builder()
    .setWorkerFactory(MyWorkerFactory())
    .build()

WorkManager.initialize(context, configuration)
```

# Wrapping Up

In conclusion, Android has made notable strides in becoming a DI-friendly framework by introducing various APIs that enable constructor injection.

The article covered several key APIs used by the two most popular DI frameworks for Android.

- `AppComponentFactory` for customising the instantiation of application components.
- `LayoutInflater.Factory2` and `View` for customising view inflation.
- `FragmentFactory` for customising fragment creation.
- `ViewModelProvider.Factory` with `CreationExtras` for simplifying ViewModel creation.
- `WorkerFactory` for customising Worker creation in WorkManager.

# Credits

Special thanks to [Maria Chietera](https://twitter.com/mchietera), and [Fred Porciúncula](https://twitter.com/tfcporciuncula) proofread review! 🔍

---

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions or I missed something, feel free to reach out to me! ℹ️
