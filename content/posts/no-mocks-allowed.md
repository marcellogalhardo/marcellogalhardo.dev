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

There is a simple rule I follow to improve my code design: any "new" code I should be unit testable without requiring any additional library.

In other words:

1. You must be able to test a piece of code without using any Mocking or Assertion library.
2. You may use an xUnit Library (i.e., JUnit).

That may sound intimidating at first, but bear with me while I guide you in a step by step example.

Heads-up: I will be using source code I got from [Now In Android](https://github.com/android/nowinandroid).

# Settings Sample

Let's imagine we have a simple settings screen. After a user performs a change, it will be persisted in disk using a repository.

The repository may look like the following [UserDataRepository](https://github.com/android/nowinandroid/blob/d9057010282c48d257cd742701350108c27daa5d/core/data/src/main/java/com/google/samples/apps/nowinandroid/core/data/repository/UserDataRepository.kt#L24):

```kotlin
interface UserDataRepository {
    val userData: Flow<UserData>
    suspend fun setFollowedTopicIds(followedTopicIds: Set<String>)
    suspend fun toggleFollowedTopicId(followedTopicId: String, followed: Boolean)
    suspend fun updateNewsResourceBookmark(newsResourceId: String, bookmarked: Boolean)
    suspend fun setNewsResourceViewed(newsResourceId: String, viewed: Boolean)
    suspend fun setThemeBrand(themeBrand: ThemeBrand)
    suspend fun setDarkThemeConfig(darkThemeConfig: DarkThemeConfig)
    suspend fun setDynamicColorPreference(useDynamicColor: Boolean)
    suspend fun setShouldHideOnboarding(shouldHideOnboarding: Boolean)
}
```

And the [SettingsViewModel](https://github.com/android/nowinandroid/blob/819dd494ad3d95f9dc742947500ee2f7b461360b/feature/settings/src/main/java/com/google/samples/apps/nowinandroid/feature/settings/SettingsViewModel.kt) may look like this one:

```kotlin
class SettingsViewModel @Inject constructor(
    private val userDataRepository: UserDataRepository,
) : ViewModel() {
    
    val settingsUiState: StateFlow<SettingsUiState> =
        userDataRepository.userData
            .map { userData ->
                Success(
                    settings = UserEditableSettings(
                        brand = userData.themeBrand,
                        useDynamicColor = userData.useDynamicColor,
                        darkThemeConfig = userData.darkThemeConfig,
                    ),
                )
            }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.Eagerly,
                initialValue = Loading,
            )

    fun updateThemeBrand(themeBrand: ThemeBrand) {
        viewModelScope.launch {
            userDataRepository.setThemeBrand(themeBrand)
        }
    }

    fun updateDarkThemeConfig(darkThemeConfig: DarkThemeConfig) {
        viewModelScope.launch {
            userDataRepository.setDarkThemeConfig(darkThemeConfig)
        }
    }

    fun updateDynamicColorPreference(useDynamicColor: Boolean) {
        viewModelScope.launch {
            userDataRepository.setDynamicColorPreference(useDynamicColor)
        }
    }
}
```

TODO: Sample relying on mocks, direct dependency.

TODO: Sample inverting the dependency, Ports & Adapters.

TODO: Functional Interfaces, Typealiases.

# Good practices

1. Refuse depending on large contracts (i.e., an interface declaring multiple functions). They are painful to mock and will force you back to mocks.
2. Avoid depending on any class your SUT does not own (i.e., a shared repository or use case).
3. 


The main point is that these libraries are there to support you in your job. If your code is only testable with their presence, that is a smell something went wrong in your design.

Disclaimer: there are people with strong opinions in favor or against mocks. I stand in the midpoint. Mocks are great as an intermediate step when refactoring Legacy code, but the goal should be to remove it eventually. No testing strategy should have a mock library as their final solution for quality. That is why I intentionally suggest that you "must be able" to do it. You don't have to if the library brings value for you.

If you don't trust me, you may want to trust [Mockito-Kotlin's original author: Niek Haarman](https://twitter.com/n_haarman/status/1610569197112770561?s=20).

References:
- a
- b
- c
