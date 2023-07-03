---
title: "No Mocks Allowed"
date: 2023-06-28T18:34:00+01:00
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

[Testable code](http://xunitpatterns.com/design%20for%20testability.html) plays a crucial role in app development. When we neglect designing code for testability, we often resort to using a [mock](http://xunitpatterns.com/Mock%20Object.html) library as a mean to achieve test coverage. Mocks have become a dominant presence in the Android testing ecosystem today. However, there are drawbacks:

1. Brittle & [Change-Detector Tests](https://testing.googleblog.com/2015/01/testing-on-toilet-change-detector-tests.html). Tests built with mocks often emphasise implementation details, causing them to break even when the system's behaviour remains unchanged.
2. False Sense of Security. High test coverage achieved through extensive mocking, can create a false sense of security. When all dependencies are mocked, we may overlook the fact that no real behavior is being tested.
3. Slow Test Execution[^1]. Reflection and proxies used by mock libraries can lead to sluggish test execution, slowing down the test suite when they increase in quantity.

The primary purpose of a mock library is to aid developers in their work. If a codebase relies solely on the presence of mocks for testability, it indicates potential design flaws[^2].

To address this issue, we need to focus on making our [System Under Test (SUT)](http://xunitpatterns.com/SUT.html) genuinely testable. The key is to minimize dependencies, ideally reducing them to zero[^3]. We should aim to eliminate dependencies on components like a class that is used throughout the application, or a data model that the SUT doesn't own.

Now, let's look an example[^4].

### The Birthday Feature

Suppose we want to introduce a new feature in our app: a listing of employees' birthdays. The feature has two requirements:
1. Retrieve a set of employee records from a database.
2. Display a list of employee records whose birthday falls on the current day.

To achieve this, we notice that we can reuse the existing `EmployeeRepository` shared across the application. We only need to add a new method that filters employees based on their birthdate.

Here's an example of the modified `EmployeeRepository`:

```kotlin
class EmployeeRepository {

	// New method!
	suspend fun findEmployeesBornOn(month: Int, day: Int): List<Employee> =
		findEmployees()
			.filter { it.month == month && it.day == day }

	// Old method, reused.
	suspend fun findEmployees(): List<Employee> =
		TODO("Load employees from a DB")
		
	// 20+ unrelated methods...
}
```

And here is an example of its model:

```kotlin
data class Employee(
	val id: EmployeeId,
	val name: String,
	val birthday: LocalDate,
	
	// 20+ unrelated properties...
)
```

The view is already implemented, our task is to develop the `ViewModel` that acts as the bridge between the `View` and the `Repository`.

A **naive** implementation of the `BirthdayViewModel` could be as follows:

```kotlin
class BirthdayViewModel(
	repository: EmployeeRepository,
) : ViewModel() {

	private val _employeeBirthdays = MutableStateFlow(emptyListOf<Employee>())
	val employeeBirthdays = _employeeBirthdays.asStateFlow()

	init {
		viewModelScope.launch {
			val today = LocalDate.now()
			_employeeBirthdays.value = repository
				.findEmployeesBornOn(today.month, dayOfMonth = today.dayOfMonth)
		}
	}
}
```

However, this approach has a few issues:
- `BirthdayViewModel` depends on a `Repository` that is used everywhere. If the `Repository` changes, you need to modify the `ViewModel`. Except by `findEmployeesBornOn`, the `ViewModel` does not use any other method.
- `BirthdayViewModel` depends on `Employee`, a data class it does not own. Except by `employeeBirthdays`, the `ViewModel` does not use any other property.
- `EmployeeBirthdayViewModel` is unstable. It depends on both `EmployeeRepository` or `Employee`, and any change will directly affect it.
- The usage of `LocalDate.now()` as a static function makes it challenging to replace during testing.

Rather than coupling our feature with the `EmployeeRepository` and `Employee` data model, let's aim for loose coupling and [inversion of control](https://en.wikipedia.org/wiki/Inversion_of_control).

### Function as an Interface

Revisiting our requirements, we realize that we only need two things:

1. Retrieve the names of employees whose birthday is today.
2. Obtain the current day.

Kotlin's support for  [high-order function](https://kotlinlang.org/docs/lambdas.html) allows us to treat any [function as an interface](https://fsharpforfunandprofit.com/posts/convenience-functions-as-interfaces/). This concept enables us to achieve loose coupling[^5].

Here's an improved version of `BirthdayViewModel` leveraging a function as an interface[^6]:

```kotlin
class BirthdayViewModel(
	findEmployeeNamesBornToday: suspend () -> List<Employee>,
) : ViewModel() {

	private val _employeeBirthdays = MutableStateFlow(emptyListOf<String>())
	val employeeBirthdays = _employeeBirthdays.asStateFlow()

	init {
		viewModelScope.launch {
			_employeeBirthdays.value = findEmployeeNamesBornToday()
		}
	}

    data class Employee(val name: String, val birthday: LocalDate)
}

// For simplicity, will do all wiring in the factory:
val BirthdayViewModelFactory: ViewModelProvider.Factory = viewModelFactory {
	initializer {
		val application = (this[APPLICATION_KEY] as MyApplication)
		val repository = application.employeeRepository
		val findEmployeeNamesBornToday = createFindEmployeeNamesBornToday(
            findEmployees = repository::findEmployees,
            now = LocalDate::now,
        )
		BirthdayViewModel(findEmployeeNamesBornToday)
	}
}

// Creates an implementation of `findEmployeeNamesBornToday`. Could be a class or whatever.
fun createFindEmployeeNamesBornToday(
    findEmployees: () -> List<Employee>,
    now: () -> LocalDate,
): suspend () -> BirthdayViewModel.Employee = suspend {
    val today = now()
    findEmployees()
        .filter { it -> it.birthday.month == month && it.birthday.dayOfMonth == today.dayOfMonth }
        .map { it -> BirthdayViewModel.Employee(it.name, it.birthday) }
}
```

This approach offers several advantages over the previous implementation:

- During testing, it becomes straightforward to provide a trivial fake implementation of the `findEmployeeNamesBornToday` method.
- `BirthdayViewModel` has access only to the specific function or property it requires, promoting better encapsulation.
- `BirthdayViewModel` achieves stability since changes to `EmployeeRepository` or `Employee` no longer directly impact it.
- Both `BirthdayViewModel` and `createFindEmployeeBirthday` can be tested in isolation without mocks or any complicated architecture, as easy as passing custom functions during your test set-up.

### Wrapping Up

In conclusion, relying excessively on mocks can lead to various pitfalls. By minimizing dependencies, we can achieve more robust and maintainable code. This architectural approach, known as "Ports & Adapters,"[^7] allows for interchangeable adapters, enabling different implementations for production and testing scenarios. Embracing testable design principles ensures that our tests accurately reflect the desired behaviour of the system, fostering a more reliable and efficient software development process.

If you want to learn more testing without mocks, here are a few links that can help you in your journey:

- [Ports and Adapters Architecture](http://wiki.c2.com/?PortsAndAdaptersArchitecture)
- [Ports And Adapters / Hexagonal Architecture](https://www.dossier-andreas.net/software_architecture/ports_and_adapters.html)
- [Testing on the Toilet: Do Not Overuse Mocks](https://testing.googleblog.com/2013/05/testing-on-toilet-dont-overuse-mocks.html)
- [Testing on the Toilet: Test Behaviour, Not Implementation](https://testing.googleblog.com/2013/08/testing-on-toilet-test-behavior-not.html)
- [Testing on the Toilet: Change-Detector Tests Considered Harmful](https://testing.googleblog.com/2015/01/testing-on-toilet-change-detector-tests.html)
- [Jetpack: Do Not Mock](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-core-core-role-release/docs/do_not_mock.md)
- [Joe Blubaugh: Do Not Mock](https://joeblu.com/blog/2023_06_mocks/)
- [Fakes Are Great, But Mocks I Hate](https://www.billjings.com/posts/title/fakes-are-great-but-mocks-i-hate/)
- [Testing Without Mocks](https://www.jamesshore.com/v2/projects/nullables/testing-without-mocks)
- [How to Write Good Tests](https://github.com/mockito/mockito/wiki/How-to-write-good-tests)

### Frequently Asked Questions

1. That is an article about tests. Why is there, not a single test?

Good question. I did plan to write a before and after with tests, but the draft of my article got featured on [Android Weekly #557](https://androidweekly.net/issues/issue-577) (?), and I _really-really want to play Final Fantasy 16_ so I guess this is now the final version.

2. That is interesting but how does it look in practice?

One of the article's reviewer has been kind, and shared a real use case from their production project:

![Sign-Up Form](/images/no-mocks-allowed-example.jpeg)

3. Should I do that for everything?

Software engineering is about trade-offs. I recommend you to follow the [Principle of Least Power: Dependency Injection](https://www.lihaoyi.com/post/StrategicScalaStylePrincipleofLeastPower.html#dependency-injection) and build up as your requirements force you to introduce more complexity.

4. You could achieve Ports & Adapters by introducing an interface to the repository. What is the advantage?

True, but repositories are often shared between multiple features creating a coupling between the components of our system. Alternatively, you can create a repository interface in your module but again: [function as an interface](https://fsharpforfunandprofit.com/posts/convenience-functions-as-interfaces/) solves the problem with ease.

### Credits

Special thanks to [Jacob Rein](https://twitter.com/deathssouls), [Stojan Anastasov](https://twitter.com/s_anastasov), [Fabricio Vergara](https://www.linkedin.com/in/fabriciovergal), [Thiago Souto](https://twitter.com/othiagosouto) and [Guilherme Baptista](https://github.com/guilhermesgb) proofread review! 🔍

And a special thank you to [Niek Haarman](https://twitter.com/n_haarman) for the [Twitter thread](https://twitter.com/n_haarman/status/1610908251553501184) that motivated me to write the blog post.

---

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions or I missed something, feel free to reach out to me! ℹ️

[^1]: That is a stretch, I know. Any testing framework or library can introduce overhead. It's not exclusive to mocks.
[^2]: See [How to Write Good Tests](https://github.com/mockito/mockito/wiki/How-to-write-good-tests) for more examples.
[^3]: While it is important to minimize unnecessary dependencies, in practical scenarios, it is not always possible or even desirable to have zero dependencies.
[^4]: Birthday example is inspired by [Birthday Greetings Kata](http://matteo.vaccari.name/blog/archives/154).
[^5]: Alternatively, you can use a nested [Functional Interface](https://kotlinlang.org/docs/fun-interfaces.html) instead of a [high-order function](https://kotlinlang.org/docs/lambdas.html). That is useful when you need distinguished types, such as when using libraries such as [Dagger](https://dagger.dev/) or [Koin](https://insert-koin.io/).
[^6]: I know the example is a too simple, there isn't much to test. But I hope you get the point.
[^7]: `BirthdayViewModel` is the system, `findEmployeeNamesBornToday` is a port, and `createFindEmployeeNamesBornToday` creates the adapter. Mastering Ports & Adapters is a powerful skill which can be applied to *any level of abstraction* (functions, classes, or entire modules!) as you see fit.
