---
title: "No Mocks Allowed"
date: 2023-06-28T09:02:50+01:00
draft: true
toc: false
images:
- /logo.png
categories:
- software development
tags:
- kotlin
---

[Testable code](http://xunitpatterns.com/design%20for%20testability.html) plays a crucial role in app development. When we neglect designing code for testability, we often resort to using a [mock](http://xunitpatterns.com/Mock%20Object.html) library as a means to achieve test coverage. Mocks have become a dominant presence in the Android testing ecosystem today. However, the practical implications of overusing mocks reveal several drawbacks:

1. Brittle Tests. Mocks tend to create tests that are fragile and easily break, hampering our ability to deliver quickly.
2. Change-Detector Tests. Tests built with mocks often emphasise implementation details, causing them to break even when the system's behaviour remains unchanged.
3. Slow Test Execution[^1]. Reflection and proxies used by mock libraries can lead to sluggish test execution, slowing down the overall testing process when they scale.
4. False Sense of Security. High test coverage achieved through extensive mocking, can create a false sense of security. When all dependencies are mocked, we may overlook the fact that no real behavior is being tested.

The primary purpose of a mock library is to aid developers in their work. If a codebase relies solely on the presence of mocks for testability, it indicates potential design flaws[^2].

To address this issue, we need to focus on making our System Under Test (SUT) genuinely testable. The key is to minimize dependencies, ideally reducing them to zero[^3]. We should aim to eliminate dependencies on components like a class that is used throughout the application, or a data model that the SUT doesn't own.

Now, let's explore a practical example to illustrate these concepts.

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
	val birthday: LocalDateTime,
	
	// 20+ unrelated properties...
)
```

The view is already implemented, our task is to develop the `ViewModel` that acts as the bridge between the `View` and the `Repository`.

A naive implementation of the `BirthdayViewModel` could be as follows:

```kotlin
class BirthdayViewModel(
	repository: EmployeeRepository,
) : ViewModel() {

	private val _employeeBirthdays = MutableStateFlow(emptyListOf<Employee>())
	val employeeBirthdays = _birthdays.asStateFlow()

	init {
		viewModelScope.launch {
			val today = LocalDateTime.now()
			_employeeBirthdays.value = repository
				.findEmployeesBornOn(today.month, day = today.day)
		}
	}
}
```

However, this approach has a few issues:
- `BirthdayViewModel` depends on a `Repository` that is used everywhere. If the `Repository` changes, you need to modify the `ViewModel`. Except by `findEmployeesBornOn`, the `ViewModel` does not use any other method.
- `BirthdayViewModel` depends on `Employee`, a data class it does not own. Except by `birthday`, the `ViewModel` does not use any other property.
- `EmployeeBirthdayViewModel` is unstable. It depends on both `EmployeeRepository` or `Employee`, and any change will directly affect it.
- The usage of `LocalDateTime.now()` as a static function makes it challenging to replace during testing.

Rather than coupling our feature with the `EmployeeRepository` and `Employee` data model, let's aim for loose coupling and invert the control.

### Function as an Interface

Revisiting our requirements, we realize that we only need two things:

1. Retrieve the names of employees whose birthday is today.
2. Obtain the current day.

Kotlin's support for  [high-order function](https://kotlinlang.org/docs/lambdas.html) allows us to treat any [function as an interface](https://fsharpforfunandprofit.com/posts/convenience-functions-as-interfaces/). This concept enables us to achieve loose coupling[^4].

Here's an improved version of `BirthdayViewModel` leveraging a function as an interface:

```kotlin
class BirthdayViewModel(
	findEmployeeNamesBornToday: suspend () -> List<String>,
) : ViewModel() {

	private val _employeeBirthdays = MutableStateFlow(emptyListOf<String>())
	val employeeBirthdays = _birthdays.asStateFlow()

	init {
		viewModelScope.launch {
			_employeeBirthdays.value = findEmployeeNamesBornToday()
		}
	}
}



val BirthdayViewModelFactory: ViewModelProvider.Factory = viewModelFactory {
	initializer {
		val application = (this[APPLICATION_KEY] as MyApplication)
		val repository = application.employeeRepository
		// `findEmployeeNamesBornToday` implementation.
		//   can be a class, a function, or whatever.
		val findEmployeeNamesBornToday = suspend {
			val today = LocalDateTime.now()
			repository
				.findEmployeesBornOn(today.month, day = today.day)
				.map { it.name }
		}
		BirthdayViewModel(findEmployeeNamesBornToday)
	}
}
```

This approach offers several advantages over the previous implementation:

- During testing, it becomes straightforward to provide a trivial fake implementation of the `findEmployeeNamesBornToday` method.
- `BirthdayViewModel` has access only to the specific function or property it requires, promoting better encapsulation.
- `BirthdayViewModel` achieves stability since changes to `EmployeeRepository` or `Employee` no longer directly impact it.

### Wrapping Up

In conclusion, relying excessively on them can lead to various pitfalls. By minimizing dependencies, we can achieve more robust and maintainable code. This architectural approach, known as "Ports & Adapters," allows for interchangeable adapters, enabling different implementations for production and testing scenarios. Embracing testable design principles ensures that our tests accurately reflect the desired behaviour of the system, fostering a more reliable and efficient software development process.

#### Foot Notes

[^1]: Any testing framework or library can introduce overhead. It's not exclusive to Mocks.
[^2]: See [How to Write Good Tests]([https://github.com/mockito/mockito/wiki/How-to-write-good-tests](https://github.com/mockito/mockito/wiki/How-to-write-good-tests "https://github.com/mockito/mockito/wiki/How-to-write-good-tests")) for more examples.
[^3]: While it is important to minimize unnecessary dependencies, in practical scenarios, it is not always possible or even desirable to have zero dependencies.
[^4]: alternatively, you can use a nested [Functional Interface](https://kotlinlang.org/docs/fun-interfaces.html) instead of a [high-order function](https://kotlinlang.org/docs/lambdas.html). That is useful when you need to distinguished types, such as when using libraries such as [Dagger](https://dagger.dev/) or [Koin](https://insert-koin.io/).

#### References: 

- [Mockito-Kotlin's original author: Niek Haarman](https://twitter.com/n_haarman/status/1610569197112770561?s=20)
- [Ports and Adapters Architecture](http://wiki.c2.com/?PortsAndAdaptersArchitecture)
- [Ports & Adapters](https://www.dossier-andreas.net/software_architecture/ports_and_adapters.html)
- [Do Not Mock](https://joeblu.com/blog/2023_06_mocks/)
- [Do Not Overuse Mocks](https://testing.googleblog.com/2013/05/testing-on-toilet-dont-overuse-mocks.html)
- [Test Behaviour, Not Implementation](https://testing.googleblog.com/2013/08/testing-on-toilet-test-behavior-not.html)
- [Change Detector Tests](https://testing.googleblog.com/2015/01/testing-on-toilet-change-detector-tests.html)
- [Jetpack Team: Do Not Mock](https://android.googlesource.com/platform/frameworks/support/+/refs/heads/androidx-core-core-role-release/docs/do_not_mock.md)
- [Fakes Are Great, But Mocks I Hate]([https://www.billjings.com/posts/title/fakes-are-great-but-mocks-i-hate/](https://www.billjings.com/posts/title/fakes-are-great-but-mocks-i-hate/ "https://www.billjings.com/posts/title/fakes-are-great-but-mocks-i-hate/"))
- [Testing Without Mocks]([https://www.jamesshore.com/v2/projects/nullables/testing-without-mocks](https://www.jamesshore.com/v2/projects/nullables/testing-without-mocks "https://www.jamesshore.com/v2/projects/nullables/testing-without-mocks")
- [How to Write Good Tests]([https://github.com/mockito/mockito/wiki/How-to-write-good-tests](https://github.com/mockito/mockito/wiki/How-to-write-good-tests "https://github.com/mockito/mockito/wiki/How-to-write-good-tests"))

### Credits

Special thanks to [Maria Chietera](https://twitter.com/mchietera), and [Jacob Rein](https://twitter.com/deathssouls) proofread review! 🔍

---

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions or I missed something, feel free to reach out to me! ℹ️
