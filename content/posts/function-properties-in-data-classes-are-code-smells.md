---
title: "Function Properties in Data Classes are Code Smells"
date: 2024-11-29T16:09:00+01:00
draft: false
toc: false
images:
- /logo.png
categories:
- software development
tags:
  - android
  - kotlin
  - kmp
---

To me, using functions as properties in the primary constructor of a data class is a **code smell**. Here's why:

- **Data classes represent data.** Data is a value. Data is never executed.
- **Functions are not data.** They produce values when executed.

**Note:** By the book, a **function** returns a value, while a **procedure** executes commands. In both cases, **neither is data**.

### Why It Matters

Kotlin generates key methods for data classes based on the properties in the primary constructor, such as:

- `equals()` and `hashCode()` for comparing instances.
- `toString()` for readable output like `"Person(name=John Doe, age=30)"`.

When you add functions as properties, you breaks the `data class` contract of holding only data, and these generated methods will misbehave.

### Example: Data vs. Functions

#### Expected Behavior: Data Class with Data.

```kotlin
data class Person(
  val name: String,
  val surname: String,
  val age: Int,
)

val p1 = Person("John", "Doe", 22)
// Person(name=John, surname=Doe, age=22)
val p2 = Person("John", "Doe", 22)
// Person(name=John, surname=Doe, age=22)
val p3 = Person("John", "Doe", 44)
// Person(name=John, surname=Doe, age=44)

// equals
println(p1 == p2) // true
println(p2 == p3) // false

// hashCode
println(p1.hashCode() == p2.hashCode()) // true
println(p2.hashCode() == p3.hashCode()) // false

// toString
println(p1.toString() == p2.toString()) // true 
println(p2.toString() == p3.toString()) // false
```

#### Unexpected Behavior: Data Class with Functions

```kotlin
data class UiState(
  val people: List<Person>,
  val eventSink: (UiEvent) -> Unit,
)
sealed class UiEvent

val s1 = UiState(listOf(), eventSink = {})
// UiState(people=[], eventSink=Function1<UiEvent, kotlin.Unit>)
val s2 = UiState(listOf(), eventSink = {})
// UiState(people=[], eventSink=Function1<UiEvent, kotlin.Unit>)
val s3 = UiState(listOf(), eventSink = { println("Effect") })
// UiState(people=[], eventSink=Function1<UiEvent, kotlin.Unit>)

// equals
println(s1 == s2) // false
println(s2 == s3) // false

// hashCode
println(s1.hashCode() == s2.hashCode()) // false
println(s2.hashCode() == s3.hashCode()) // false

// toString
println(s1.toString() == s2.toString()) // true 
println(s2.toString() == s3.toString()) // true
```

You can find the example above at [this link](https://pl.kotl.in/49cN7r5Wu).

### Issues in Detail

1. **Equality** (`equals`): Functions are never equal, even if they are duplicates of each other. 
2. **Hash Codes** (`hashCode`): Different instances with same data generates different hashes.
3. **String Representation** (`toString`): Functions produce generic output, leading to misleading comparisons.

For example, using an assertion library? You might encounter:

> "(non-equal value with same string representation)"

This happens because function properties can't be meaningfully compared or represented, causing `equals` and `toString` to behave inconsistently.

### The Solution

**Use data classes only for data.** If you need to include functions or behaviors like callbacks:

- Use a regular `class`.
- Override `equals()`, `hashCode()`, and `toString()` manually.

By using a regular class, you clarify intent, avoid unintended behavior from generated methods, and uphold the principle that **data classes should only hold data**.

### F.A.Q.

#### 1. Functions are considered equal if they refer to the same instance. For example, you can assign them to a `val`.

Exactly. Data is compared based on their content, not their instance reference. This supports my argument.

#### 2. You can override `equals()`, `hashCode()`, and `toString()` in a `data class`.

Yes, but my point is about the semantic definition of a "data class" and how the compiler's assumptions about what is `data` can lead to issues with the generated methods.

#### 3. The example you put forth looks a lot like what we do in Circuit, was that what part of prompted this?

No. I'm not familiar with Circuit, so I don’t feel qualified to comment on it. My focus was to discuss `data class` as a Kotlin construct, separate from Compose or any specific library.

That said, Zac Sweers (Slack/Circuit) has reached out to confirm that Circuit leverages the Compose compiler [Strong Skipping](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping#lambda-memoization) for lambda memoization, preventing the issues in this article. You can find his comment [here](https://bsky.app/profile/zacsweers.dev/post/3lc42yw76cc2f).

### References

- [Kotlin Data Classes Documentation](https://kotlinlang.org/docs/data-classes.html)
- [A Curious Case of Mistaken Identity](https://blog.mmckenna.me/a-curious-case-of-mistaken-identity)
