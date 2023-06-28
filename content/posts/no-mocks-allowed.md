---
title: "No Mocks Allowed"
date: 2023-05-01T09:02:50+01:00
draft: true
toc: false
images:
- /logo.png
categories:
- software development
tags:
- kotlin
---

---
title: "No Mocks Allowed"
date: 2023-05-01T09:02:50+01:00
draft: true
toc: false
images:
- /logo.png
categories:
- software development
tags:
- kotlin
---

Testable code plays a crucial role in app development. When we neglect designing code for testability, we often resort to using Mocks as a means to achieve test coverage. Mocks have become a dominant presence in the Android testing ecosystem today. However, the practical implications of relying heavily on Mocks reveal several drawbacks:

1. Brittle Tests: Mocks tend to create tests that are fragile and easily break, hampering our ability to deliver quickly.
2. Implementation Focus: Tests built with Mocks often emphasise implementation details, causing them to break even when the system's behaviour remains unchanged.
3. Slow Test Execution: Reflection and proxies used by Mock libraries can lead to sluggish test execution, slowing down the overall testing process when they scale.
4. False Sense of Security: High test coverage achieved through extensive mocking can create a false sense of security. When all dependencies are mocked, we may overlook the fact that no real behavior is being tested.

The primary purpose of a Mock library is to aid developers in their work. If a codebase relies solely on the presence of Mocks for testability, it indicates potential design flaws.

To address this issue, we need to focus on making our System Under Test (SUT) genuinely testable. The key is to minimize dependencies, ideally reducing them to zero. We should aim to eliminate dependencies on components like a class used throughout the application, or data model that the SUT doesn't own.

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

Kotlin's support for higher-order functions allows us to treat any [function as an interface](https://fsharpforfunandprofit.com/posts/convenience-functions-as-interfaces/).. This concept enables us to achieve loose coupling.

Here's an improved version of `BirthdayViewModel` leveraging a function as an interface:

```kotlin
class BirthdayViewModel(
	findEmployeeNamesBornToday: suspend () -> List<String>,
) : ViewModel() {

	private val _employeeBirthdays = MutableStateFlow(emptyListOf<String>())
	val employeeBirthdays = _birthdays.asStateFlow()

	init {
		viewModelScope.launch {
			_birthdays.value = repository.findEmployeeNamesBornToday()
		}
	}
}


// `findEmployeeNamesBornToday` implementation
fun createFindEmployeeNamesBornToday(repository: EmployeeRepository): suspend () -> List<String> {
	val today = LocalDateTime.now()
	return suspend {
		_employeeBirthdays.value = repository
			.findEmployeesBornOn(today.month, day = today.day)
			.map { it. }
	}
}

```

This approach offers several advantages over the previous implementation:

- During testing, it becomes straightforward to provide a trivial fake implementation of the `findEmployeeNamesBornToday` method.
- `BirthdayViewModel` has access only to the specific function or property it requires, promoting better encapsulation.
- `BirthdayViewModel` achieves stability since changes to `EmployeeRepository` or `Employee` no longer directly impact it.

## Wrap Up

In conclusion, relying excessively on them can lead to various pitfalls. By minimizing dependencies, we can achieve more robust and maintainable code. This architectural approach, known as "Ports & Adapters," allows for interchangeable adapters, enabling different implementations for production and testing scenarios. Embracing testable design principles ensures that our tests accurately reflect the desired behavior of the system, fostering a more reliable and efficient software development process.

## References: 

- [Mockito-Kotlin's original author: Niek Haarman](https://twitter.com/n_haarman/status/1610569197112770561?s=20)
- [Ports and Adapters Architecture](http://wiki.c2.com/?PortsAndAdaptersArchitecture)
- [Ports & Adapters](https://www.dossier-andreas.net/software_architecture/ports_and_adapters.html)
- [Do Not Mock](https://joeblu.com/blog/2023_06_mocks/)

---

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions or I missed something, feel free to reach out to me! ℹ️
