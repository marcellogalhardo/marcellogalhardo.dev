---
title: "Humble Views, Proud ViewModels"
date: 2021-02-01T09:02:50+01:00
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

The Android Community has long advocated that Activities and Fragments were views  -  but this perception has changed over time. For good. Let's dive deep into how to design views and view models, how they wire to a LifecycleOwner, and how this can positively impact your's app testability.

![Sign-Up Form](/images/2021/02/01/sign-up-form.png)

To better describe how to build humble views we will be developing an elementary Sign-Up form with an email, a password text field and two buttons: a cancel that pops the user's back stack and a sign up that creates an account and moves the user to the home screen.

> ["Fragment is not your View" - Juliano Moraes](https://medium.com/r?url=https%3A%2F%2Fnetflixtechblog.com%2Fmaking-our-android-studio-apps-reactive-with-ui-components-redux-5e37aac3b244)

**Heads-up:** this article expects you to be familiar with Dependency Injection (but no particular framework), Coroutines, Fragment, and View Binding. I won't use Jetpack's ViewModel, but the code here is entirely compatible. I will not cover any other aspects outside humble Views and ViewModels.

# Views

`View`s represent your UI. They can be written in code or, the most common way, using XML. Android views can not have a custom constructor as they must be created by the OS using reflection when inflating it from XML, during a configuration change, or process death.

`View`s should not be aware of your architecture decisions. If you use MVP, MVC, MVVM, or MVI is irrelevant for a well-designed view.

`View`s are hard to test and require an Instrumentation or Robolectric environment, which makes them slow. For that reason, we might want our views to be a [Humble Object](https://martinfowler.com/bliki/HumbleObject.html).

```kotlin
class SignUpView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : ConstraintLayout(context, attrs) {

    private val binding = SignUpViewBinding.inflate(LayoutInflater.from(context), this)

    var email: String
        get() = binding.email.text
        set(value) { binding.email.setText(value) }

    var password: String
        get() = binding.email.text
        set(value) { binding.email.setText(value) }

    var isSignUpEnabled: Boolean
        get() = binding.signUpButton.isEnabled
        set(value) { binding.signUpButton.isEnabled = value }

    var onSignUpClicked: () -> Unit = {}
    var onCancelClicked: () -> Unit = {}
    var onEmailChanged: (String) -> Unit = {}
    var onPasswordChanged: (String) -> Unit = {}

    init {
        binding.signUpButton.setOnClickListener { onSignUpClicked() }
        binding.onCancelClicked.setOnClickListener { onCancelClicked() }
        binding.email.doOnTextChanged { text, _, _, _ ->
            onEmailChanged(text)
        }
        binding.password.doOnTextChanged { text, _, _, _ ->
            onPasswordChanged(text)
        }
    }
}
```

#### What is going on with this ViewBinding?

In my `View`'s XML root, I use a `<merge>` tag, and `ViewBinding` understands that: it provides me with an inflate function that attaches the view automatically to its parent.

# ViewModels

`ViewModel`'s are a lightweight representation of your UI. The `ViewModel` is responsible for coordinating any user interaction to any business model required. It may offer a one or two-way binding to let those business models interact with the GUI.

```kotlin
class SignUpViewModel(
  private val savedStateHandle: SavedStateHandle,
  private val scope: CoroutineScope,
  private val repository: SignUpRepository,
) {

  private val _navigation = Channel<Navigation>()
  val navigation = _navigation.receiveAsFlow()

  private val _toastMessage = Channel<ToastMessage>()
  val toastMessage = _toastMessage.receiveAsFlow()

  val email = savedStateHandle.getStateFlow<String>(scope, "email", "")

  val password = savedStateHandle.getStateFlow<String>(scope, "password", "")

  private val _isSignUpEnabled = savedStateHandle.getStateFlow<Boolean>(scope, "isSignUpEnabled", false)
  val isSignUpEnabled = _isSignUpEnabled.asStateFlow()

  init {
    email.collectIn(scope) { updateSignUpEnabled() }
    password.collectIn(scope) { updateSignUpEnabled() }
  }

  fun onSignUpClicked() {
    runCatching {
      // Try to parse e-mail and validate password. If yes, save.
      val email: String = // parse e-mail.
      val password: String = // verify if password is acceptable.
      repository.signUp(email, password)
    }.fold(
      onSuccess = { _navigation.sendIn(scope, Navigation.Push(SignUpRoutes.Home)) },
      onFailure = { _toastMessage.sendIn(scope, ToastMessage(R.string.generic_error)) },
    )
  }

  fun onCancelClicked() {
    _navigation.sendIn(scope, Navigation.Pop)
  }

  private fun updateSignUpEnabled() {
    // Naive logic to enable sign up.
    _isSignUpEnabled = email.isNotBlank() && password.isNotBlank()
  }
}
```

#### Why expose some flows as MutableStateFlow and others no?

Each field has a different requirement: two-way or one-way binding.

TextFields requires a two-way binding as we want to get updates when the user types a new content while preserving the ability to update its content whenever necessary. Exposing it as a `MutableStateFlow` allows us to move all sanitization logic to the view model (see `onSignUpClicked`) and ensure the View is as humble as possible.

The sign-up button state requires a one-way binding (see `isSignUpEnabled`)  -  only the `ViewModel` can change its state based in a validation logic.

#### Why not modeling my UI State as a single flow in the ViewModel?

The concepts here do not exclude a single view state, I believe they complement each other.

You can easily keep your view humble while modeling UI State as a single object on top of the ViewModel. In this case, the `ViewModel` would be responsible for handling the unidirectional data flow and doing the required diffs between each state emission within your bindings to enforce your UI is not being updated without demand. Therefore, let your view humble allow you to keep it agnostic of how you handle state, architecture decisions, and it enable you to test your `ViewModel` and classes above it as a unit very closely of how it would behave in the real world with a GUI: remember, the `View` is humble!

I will not cover these concepts here but for a better understanding of why you would like to model your UI State as a single sealed hierarchy and how to do it well, I highly recommend my friend's [Stojan Anastasov](https://twitter.com/s_anastasov) post about the subject: [Modeling UI State](https://lordraydenmk.github.io/2021/modelling-ui-state/).

# LifecycleOwner

The UI Controller. Usually, a `Fragment` responsible for a section of the screen. A `LifecycleOwner` is responsible to wire up a `View` to a `ViewModel`, delegate callbacks from the Android OS, handle configuration changes and more.

```kotlin
class SignUpFragment : Fragment() {

    // Retain the instance across configuration changes.
    private val viewModel by retain { entry ->
        val repository: SignUpRepository = // creates the repository
            SignUpViewModel(entry.savedStateHandle, entry.scope, repository)
    }

    // Creates your View and connects to the ViewModel.
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        val view = SignUpView(requireContext())

        // Bind the View and ViewModel
        view.onSignUpClicked = viewModel::onSignUpClicked
        view.onCancelClicked = viewModel::onCancelClicked
        viewModel.email.observeIn(viewLifecycleOwner, view::email::set)
        viewModel.password.observeIn(viewLifecycleOwner, view::password::set)
        viewModel.isSignUpEnabled.observeIn(viewLifecycleOwner, view::isSignUpEnabled::set)
        // Other binds you might need with the view...
        viewModel.navigation.observeIn(viewLifecycleOwner) { navigation ->
            // Handle navigation.
        }
        viewModel.toastMessage.observeIn(viewLifecycleOwner) { message ->
            // Handle toast message.
        }

        return view
    }
}
```

# Testing

We designed our View as a Humble Object to let us quickly test both `View` and `ViewModel`. As you might have perceived, the `ViewModel` is straightforward to write tests as we do not depend on any Android related class.

```kotlin
class SignUpViewModelTest {

  private val scope = TestCoroutineScope()

  // A fake version of the data source to not cross boundaries like network and database.
  private val dataSource = TestDataSource()

  private fun createSut(): SignUpViewModel {
    return SignUpViewModel(SavedStateHandle(), scope, SignUpRepository(dataSource))
  }

  // Set up, tear down, other tests, etc...

  @Test
  fun `given email and password field is not blank when signing up then navigate to the home` {
    // Arrange Phase
    val sut = createSut().apply {
      email = "my.fake.person@gmail.com"
      password = "8S#@2LAaB_.NJh(Y"
    }
    var actualNavigation: Navigation? = null
    scope.launch { sut.navigation.collect { actualNavigation = it } }

    // Act Phase
    sut.onSignUpClicked()

    // Assert Phase
    val expectedNavigation = Navigation.Push(SignUpRoutes.Home)
    assertThat(actualNavigation).isEqualTo(expectedNavigation)
  }
}
```

As the view is a humble object, there is nothing much to test on it. However, we might want to ensure that the UI behaves as expected when the `ViewModel` changes its attributes. For example:

```kotlin
class SignUpViewTest {

  @Test
  fun `given sign up is disabled then the button should not be clickable`() {
    launchViewInFragment { SignUpView(requireContext()) }
      .onView { it.isSignUpEnabled = false }
    
    onView(withId(R.id.signUpButton))
      .check(matches(not(isEnabled())))
  }
}
```

# Conclusion

It is relatively easy to see that `View`'s, in Android, are strongly tied to the concept of their functionality. Humble View's is a way to create a clear separation between them and the Android Framework aiming for scalability, readability, and maybe the most essential point, testing.

In this article, we applied Humble Object pattern and MVVM concepts to create the circumstances that would allow developers to test `View`'s and `ViewModel`'s with ease even when they are strongly tied conceptually. Additionally, understanding that the `LifecycleOwner` is not your `View` will help you to ensure your code is highly testable.

Other than testing, projects that use this approach can easily compose more complex `View`'s, and `ViewModel`'s, from smaller objects. Lastly, this approach fit with Jetpack's Compose: replace the `View` with a `@Composable` and reuse the `ViewModel`!

**Happy Coding!** 😎

# Helper functions & credits!

I used [Retained](http://github.com/marcellogalhardo/retained) library to keep the state of the `ViewModel` in configuration changes. I also published all the helper functions as [Github GISTs](https://gist.github.com/marcellogalhardo/a9985f7b3875fa41c379a2ba65d8ac9c), so you can use them as you please.

Special thanks to [Maria Chietera](https://twitter.com/mchietera), [Tiago Dávila](https://twitter.com/TiagoDvl), [Rafael Araujo](https://twitter.com/orafaaraujo), [Tiago Cunha](https://twitter.com/laggedHero), [Felipe Pedroso](https://twitter.com/felipeapedroso), and [Stojan Anastasov](https://twitter.com/s_anastasov) proofread review! 🔍

---

> ℹ️ To stay up to date with my writing, follow me on [Twitter](https://twitter.com/marcellogalhard) or [Mastodon](http://androiddev.social/@mg). If you have any questions or I missed something, feel free to reach out to me! ℹ️
