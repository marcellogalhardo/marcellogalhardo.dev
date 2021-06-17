---
title: "Build your App on Android, not with"
date: 2021-06-17T17:37:41+02:00
draft: true
toc: false
images:
categories:
  - software development
tags:
  - android
---

As Android Developers we take much as granted. We have an `Application`, that is automatically created by the framework. We have an `Activity`, and an `Intent` system that we can use to navigate within activities - and I'm not even talking about `Fragment`, `Service` and others.

All of us are familiar with all the documentation teaching how we should do Android, what our classes should look like, and how they should communicate.

What if we are building in the wrong abstraction for so long that we are limited to repeat our steps again and again?

Heads-up: the following text assumes you are familiar with Dagger.

## It is not about what we do

We know how hard is to isolate the Android Framework: we need a `Context`, or in worse cases, an Activity, for everything. Sometimes it feels like Android does it best to be hard. We master all the concepts, we do all the tutorials, we read books and still the framework keep getting in our way and how we build our Android application.

## It is about how we do

We master how to build Android applications, except by we do not build Android applications. We build Applications, and those applications will run on the Android platform. That might sound like a small thing, but once we understand it well it open a sea of possibilities. Once this is clear to use, we unlock a design potential where the Android gets completely out of the way.

## Designing applications

The first thing we do when building the `Application` is to create entry point for our code. For the Java nostalgic, let's call an object called `Main` with a single method called `main` - for now it is empty but we will get back here.

```kotlin
object Main {

    fun main(application: Application) {
        TODO("Not implemented")
    }
}
```

As the Android Framework already uses the word `Application`, let's create a class called `Program` that will be our application root.

```kotlin
class Program @Inject constructor(
    private val application: Application,
)
```

Here, we will be using Dagger to take care of our object graph but the idea can be reproduced by any other Dependency Injection framework:

```kotlin
@Component
interface ApplicationComponent {

    fun getProgram(): Program

    @Component.Factory
    interface Factory {

        fun create(@BindsInstance application: Application): ApplicationComponent
    }
}
```

Nothing special so far, right? Now, let's start to connect our application with the Android Framework. First we will start our main method when the Android creates the `Application` class.

```kotlin
class MainApplication : Application() {

	override fun onCreate() {
		Main.main(application = this)
	}
}
```

Now, let's revisit our `Main` class to connect the dots.

```kotlin
object Main {

    private lateinit var program: Program

    fun main(application: Application) {
        program = ApplicationComponent.factory()
            .create(aplication)
            .getProgram()

        // Connect any callback you might need...
        ProcessLifecycleOwner.get()
            .lifecycle
            .addObserver(program.processLifecycleObserver)
        application.registerActivityLifecycleCallbacks(program.activityLifecycleCallbacks)
        application.registerComponentCallbacks(program.componentCallbacks)
        // And there you go...
        // Pro-tip: If are familiar Dagger multi-binding, imagine all of those as multi-bindings.
    }
}
```

Right, I completely decoupled my application from the `Application` class, I can use dependency injection in my `Program` as much as I want... but I still can't show anything to the user. A little useless, right? Let's fix that by connecting our `Activity` to the `Program` and using Jetpack libraries, as they are very popular in the community.

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        // Connect to the activity FragmentFactory.
        supportFragmentManager.fragmentFactory = Main.program.fragmentFactory
        super.onCreate(savedInstanceState)
        // Inflate your layout, and use Navigation Component with Fragments.
        setContentView(R.layout.activity_main)
    }
}
```

That is it. By now I imagine that you got the idea.

Now you can create any `Fragment` using dependency injection, and those fragments can inject a `ViewModelFactoryProvider` which can retrieve view models connected to your saved state handle (e.g., `vmFactoryProvider.getViewModelFactory(activity|fragment)`.

That is even more powerful if you add Kotlin Multiplatform Mobile and/or Compose to the mix.

# Conclusion

TODO("")

If you like my posts, follow me on Twitter: [@marcellogalhard](https://twitter.com/marcellogalhard)