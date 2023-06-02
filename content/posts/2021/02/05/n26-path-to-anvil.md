---
title: "N26 Path to Anvil"
date: 2021-02-05T09:02:50+01:00
draft: false
toc: false
images:
- /logo.png
categories:
- software development
tags:
- android
- dagger
- anvil
---

This post represents my personal experience while working at N26. I do not speak for the company nor by other employees.

N26 Android App current codebase has a million lines of code, 280+ modules, and 30+ engineers working in 4 different countries and different timezones in a mono repository. Our modules are divided into features and libraries, and we have been using "[Sample Apps](https://cashapp.github.io/2020-08-25/attacking-build-times-with-sample-apps)" for years now as our full app build time might take up to 20 minutes.

Today I will share a little about my story with Dagger inside the company, why we adopt Anvil, some of the challenges we faced, and how are the results so far.

# Dagger, (oh, my)

N26 has a long history with Dagger. We have been using it for years, we developed our customizations and libraries on top of it, and many of our architectural decisions were made relying on Dagger features.

Dagger is excellent but comes with a price when used with Kotlin. The more we grew, the more we were charged. Stub generation issues, slow build time, increasing boilerplate to wire `@dagger.Component`, new joiners avoided touching the graphs due to complexity...

Looking for solutions, we found [Hilt by Google](https://dagger.dev/hilt/). We enjoyed many of the ideas they proposed as the [Monolithic Component](https://dagger.dev/hilt/monolithic.html) and the [testing philosophy](https://dagger.dev/hilt/testing.html). But the trade-off was not that good for us: a migration would be painful and would take months to years, to not say we did not appreciate the byte code manipulation and the extra KAPT processor.

Simultaneously, another solution appeared, which solved some of the same problems differently: [Anvil](https://github.com/square/anvil).

> Anvil is a Kotlin compiler plugin to make dependency injection with Dagger easier by automatically merging Dagger modules and component interfaces.

# Sharpening your blade

To get an idea of how Anvil would affect us, we started with a set of experiments. The first one would be to wire a few modules while keeping backward compatibility without impacting any feature developer. In a matter of hours, we managed to conclude it. In a week, we had many modules using Anvil. We did not identify any expressive build time impact.

We were happy with the result. We decided to be more ambitious: we selected our code base's prominent monolith module to fully migrated to Anvil, and afterward, we broke this monolith apart into small libraries. It took us a few months to complete the goal, but we succeeded and did not identify any expressive build time impact, again.

To be sure we were on the right path, we mapped the relationship of a few of our most complex modules before and after adopting Anvil. For a matter of company's privacy, all text is blurred, but you can still see the positive impact by noting the arrows:

![Before and After](/images/2021/02/05/before-and-after.png)

Finally, we decided to go full Anvil: we turned on [Anvil's Dagger Factory generation](https://github.com/square/anvil#dagger-factory-generation) in all modules that we could (20+ at the time). We identified build times improvements of ~50% for individual modules build times, ~10% for Sample Apps, and because Anvil does not rely on KAPT, we never saw any KAPT issue again on those modules.

![Benchmark](/images/2021/02/05/benchmark.png)

Many improvements, but we believed we could take it further: let's take the good things from Hilt and bring it to Anvil.

# Hilt to Anvil

We decided to adopt what we like from Hilt while using Anvil.

The first point was to provide a single monolith component, and for that, we started to merge the components and use Anvil's `@ContributesTo` to binding the modules instead. It took quite some time, it was challenging but it worked well.

The second one was to support some Jetpack Libraries. We started with `FragmentFactory` to leverage the constructor injector as much as possible. We created a `@FragmentKey` and used Dagger's multi-binding to wire everything. Here is how the code might look like:

```kotlin
@ContributesBinding(Singleton::class, FragmentFactory::class)
class MultibindingFragmentFactory @Inject constructor(
    private val map: Map<Class<out Fragment>, @JvmSuppressWildcards Provider<Fragment>>
) : FragmentFactory() {

    override fun instantiate(classLoader: ClassLoader, className: String): Fragment {
        val fragmentClass = loadFragmentClass(classLoader, className)
        return map[fragmentClass]?.get() ?: super.instantiate(classLoader, className)
    }
}

@Target(
    AnnotationTarget.CLASS,
    AnnotationTarget.FUNCTION
)
@Retention(value = AnnotationRetention.RUNTIME)
@MapKey
annotation class FragmentKey(val value: KClass<out Fragment>)

class Activity : AppCompatActivity() {

    private val component: MainComponent
        get() = TODO("Retrieve your Monolith Component")

    override fun onCreate(savedInstanceState: Bundle?) {
        supportFragmentManager.fragmentFactory = component.getFragmentFactory()
        super.onCreate(savedInstanceState)
    }
}
```

And now to wire your `Fragment`, you can simple:

```kotlin
@ContributesMultibinding(Singleton::class)
@FragmentKey(HomeFragment::class)
class HomeFragment @Inject constructor(
    // Dependencies goes here. :)
) : Fragment()
```

For more details around Fragments, check the [official guide](https://developer.android.com/guide/fragments).

Having a Monolith Component and relying on Fragment's constructor injector means we can invoke any fragment from any place of our application, and things "will work". Scoping becomes intuitive for those classes. If you inject an object in the `Fragment` is a fragment scope, if you inject into the `ViewModel` is a view model scope. We also offer a `SessionScope` and `Singleton` scope.

For `ViewModels` a simple `Provider<ViewModel>` or `AssistedInject`, if you need an instance of `SavedStateHandle`, will do the trick.

And finally, testing: Anvil offers a [replace module feature](https://github.com/square/anvil#exclusions) that is handful to provide new dependencies during tests. For that, we create helper modules called `testing` and we provide fake dependencies of those replacing the production modules. Developers that include the `testing` in their test classpath can automatically interact with our testing utilities (or create their own, if required). To be completely honest here, it is more of an ongoing process.

![Module Structure](/images/2021/02/05/module-structure.png)

# Conclusion

Anvil is a robust and straightforward solution. It does what it suppose to do and does it well. It benefits from a seamless synergy with Dagger, while not being opinionated and letting you decide how you integrate with other libraries (or not integrating it at all).

Also, the fact it does not rely on KAPT is a **tremendous advantage** for large projects and should be kept in mind while deciding between Anvil or Hilt. I'm delighted with the overall experience, and I enjoy seeing how many feature developers started to migrate away from KAPT to Anvil proactively:

![Kill KAPT](/images/2021/02/05/no-more-kapt.png)

Finally, as you can see above, many of the Hilt's features can be implemented in Anvil. However, it is vital to keep in mind Anvil is not a silver bullet. It is essential to have people in your team that understand Dagger and Dependency Injection to build the integrations you might need.

# Update 2021.02.25

As many people reached out asking advice on how to implement some Hilt features (e.g., ViewModelScope, SavedStateHandle, and others), I created a [small showcase project](https://github.com/marcellogalhardo/hilt-to-anvil).

# Update 2021.03.19

Updated `MultibindingFragmentFactory` example and [showcase project](https://github.com/marcellogalhardo/hilt-to-anvil)  to use new `@ContributesMultibinding` from [Anvil 2.2.0](https://github.com/square/anvil/releases/tag/v2.2.0).

# Credits

Thanks to [Maria Chietera](https://twitter.com/mchietera), [Rafael Araujo](https://twitter.com/orafaaraujo), [Tiago Cunha](https://twitter.com/laggedHero), [Fabio Carballo](https://twitter.com/fabiocarballo), and [Stojan Anastasov](https://twitter.com/s_anastasov) proofread review! 🔍

And a special thank you to [Ralf Wondratschek](https://twitter.com/vRallev) for early feedback and creating Anvil! :knife:

---

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions or I missed something, feel free to reach out to me! ℹ️
