# Introduction
This document serves as an overview of Thunderbird Android's code base as of commit [`045181a`](https://github.com/thunderbird/thunderbird-android/tree/045181ab5486297882b4b53e9f1a0a2a9a89fa62). It will analyze the project architecture and evaluate technical decisions and points of interest.

# Project Overview
[Thunderbird Android](https://github.com/thunderbird/thunderbird-android) is an Android mail client that allows users to manage multiple email accounts at once, including a unified inbox to collate all emails into one list. It is the successor to K-9 mail.

<img src="assets/thunderbird_sidebar.png" width=300>

# Prerequisites
Working with this project requires prerequisite knowledge and installed software. Ensure having prior knowledge of the following:

- Kotlin and Java
- Android Activities and Fragments
- Jetpack Compose
- LiveData
- Kotlin Coroutines and Flow

The following software must also be available:

- IDE ([Android Studio](https://developer.android.com/studio) preferred)
- Git

# Building the Project

## Checkout the project
The project may be checked out via GitHub CLI or any Git GUI

### GitHub CLI
`$ gh repo clone thunderbird/thunderbird-android`

### Android Studio
<img src="assets/android_studio_vcs.png">

# Project Structure

## Architecture
The project makes use of a very modular architecture that seems to be evolving over time. There are multiple module groups, many of which have their own heirarchy of submodules, and the details of which shall be discussed in more detail below. An overview of the project architecture showcasing the highest level modules is as follows:

```mermaid
---
config:
  layout: elk
---
graph LR
    AppModules["App Modules"] 
        AppModules --> AppCommon
        AppModules --> Core
        AppModules --> Legacy
        AppModules --> Feature
        AppModules --> Backend
    AppCommon["App Common"]
        AppCommon --> Legacy
        AppCommon --> Feature
        AppCommon --> Core
        AppCommon --> Mail
    Feature["Feature Modules"]
        Feature <--> Legacy
        Feature --> Core
        Feature --> Mail
        Feature --> Backend
        Feature --> UIUtils
    Core["Core Modules"]
        Core --> Mail
    Legacy["Legacy Modules"]
        Legacy --> Core
        Legacy --> Backend
        Legacy --> Mail
        Legacy --> UIUtils
        Legacy --> Library
        Legacy --> Plugins
    Backend["Backend Modules"]
        Backend --> Mail
    CLI["CLI Modules"]
        CLI --> Feature
        CLI --> Library
    Mail["Mail Modules"]
    UIUtils["UI Utils"]
    Library["Library Modules"]
    Plugins["Plugin Modules"] 
```

The most major modules to note are the App, Legacy, Feature, Backend and Mail modules.

## App modules
The App modules, which serve as the entry points or highest level of the app architecture, depend on all other modules both directly and indirectly. They are responsible for defining which features are presented to the users and how the dependency tree is built.

```mermaid
---
config:
  layout: elk
---
graph
    app-thunderbird
    app-k9mail
    app-catalog
    app-common
        app-k9mail --> app-common
        app-thunderbird --> app-common
```

`app-thunderbird` is the primary module for running the app, and contains all the features of the app alongside the Thunderbird theme. The `ThunderbirdApp` application class overrides from `CommonApp` defined in the `legacy.common` module and has additional telemetry initialization.

`app-k9mail` is a secondary module for running the app. It is the original K-9 Mail app, and has most of the features of the Thunderbird app with the main difference being the theme and lack of telemetry.

`app-ui-catalog` is a separate app purely to showcase the UI components used by both `app-thunderbird` and `app-k9mail`.

### App Common module
The purpose of this module according to the supplied README is to provide common functionality between the `app-thunderbird` and `app-k9mail` modules. However, the content of `app-common` is simply a DI setup of Account-related classes. It is possible this module may be expanded in the future, but given the current setup with the new Feature modules it may make more sense to move this DI setup to the `feature/account` module instead.

### Dependency Injection
The project uses [Koin](https://insert-koin.io/) to handle dependency injection, and takes a modular approach to creating the dependencies. The `app-thunderbird` module defines all the dependencies of the app, referencing Koin modules from other app modules such as `appCommonModule`, `featureModule`, etc., each of which define and supply their own abstractions/implementations.

## Legacy modules
The Legacy modules are the original implementation of the app, with structure going as far back as using the previous Activity and Fragment with XML layouts (For example, `MessageList` with `MessageListFragment` and `message_list_fragment.xml`). Each feature is separated into its own submodule (e.g., `account`, `search`, etc.), but many of the modules are interdependent. This can result in changes to one deeply nested module having a cascading effect, requiring updates in multiple other modules. For example, an update to the `account` module may require `core` to update, leading to `testing`, `storage`, `ui base` and `common` all needing updates too.

```mermaid
---
config:
  layout: elk
---
graph
    Account["account"]
    Core["core"]
        Core --> Account
        Core --> DI
        Core --> Mailstore
        Core --> Message
        Core --> Search
    Common["common"]
        Common --> Core
        Common --> UILegacy
        Common --> Storage
        Common --> CryptoOpenPGP
    CryptoOpenPGP["crypto-openpgp"]
    DI["di"]
    Mailstore["mailstore"]
    Message["message"]
        Message --> Mailstore
    Search["search"]
    Storage["storage"]
        Storage --> Core
    Testing["testing"]
        Testing --> Core

    subgraph ui["UI"]
        UIAccount["ui account"]
        UIBase["ui base"]
        UILegacy["ui legacy"]
        UIFolder["ui folder"]
    end
        UIBase --> Core
        UIAccount --> UIBase
        UILegacy --> UIBase
        UILegacy --> UIAccount
        UILegacy --> UIFolder
        UIFolder --> UIBase
```

## Feature modules
The Feature modules are an updated implementation of the apps features with focus on more modern code usage (e.g., Jetpack Compose instead of XML layouts), as well as more separation between features. The majority of the feature modules do not depend on other features and thus can be worked on independently.

```mermaid
---
config:
  layout: elk
---
graph 
    account["account"]
        account --> autodiscovery
    autodiscovery["autodiscovery"]
    folder["folder"]
    funding["funding"]
    launcher["launcher"]
    migration["migration"]
    navigation["navigation"]
    notification["notification"]
    onboarding["onboarding"]
        onboarding --> account
        onboarding --> settings
    settings["settings"]
        settings --> account
        settings --> migration
    telemetry["telemetry"]
    widget["widget"]
```

Each feature makes use of the MVI architecture, which is deeper dived in the [Points of Interest](#mvi-architecture) section below

## Mail modules
The Mail modules are the most foundational modules of the app, responsible for handling mail messages and interacting with different email protocols

```mermaid
---
config:
  layout: elk
---
graph
    common
    subgraph protocols
        imap
        pop3
        smtp
    end
    testing

    imap --> common
    pop3 --> common
    smtp --> common
    testing --> common
```

### Common
`mail.common` may be considered the single most foundational module of the app, providing a base on which to build higher level functionality. This module provides the following features:

#### Message Structure and Representation
Messages are built upon the abstract `Message` class whose structure is defined by `Part` (Representing different parts of a message such as headers and content type) and `Body` (Representing the message body). The foundational implementation of this abstraction is the `MimeMessage` class. This allows the project to support multiple message types such as `ImapMessage`, `Pop3Message`, `TestMessage`, etc. 

```mermaid
graph RL
    Part
    Body
    Message
        Message --> Part
        Message --> Body
    MimeMessage
        MimeMessage --> Message
    ImapMessage
        ImapMessage --> MimeMessage
    Pop3Message
        Pop3Message --> MimeMessage
    TestMessage
        TestMessage --> MimeMessage
```

Addresses are also supported by the concrete `Address` class that allows defining an email address and a personal identifier or contact name.

Different states associated with messages are defined as well, such as `Flags` (e.g., Seen, Deleted, Draft), `FolderType` (e.g., Inbox, Outbox, Spam) and `MessageDownloadState` (e.g., Partial, Full).

#### MIME Processing
The app supports the MIME standard and provides many helper classes for handling this format.

#### Logging
Despite not being related to mail, this module also provides the `Logger` abstraction, to allow using different logger implementations in different environments (e.g., Timber and `System.out`).

### Protocols
The three sub-packages here provide protocol-specific implementations for three major email protocols: `IMAP`, `POP3` and `SMTP`.

- `IMAP` is used for synchronization-based email access, meaning that all messages and their associated states are kept on the server. This module add support for synchronization via `RealImapFolder` and push notifications via `RealImapFolderIdler`.
- `POP3` is used for download-based email access, meaning that all messages will be downloaded to the device.
- `SMTP` is used for sending messages, and thus handles message transmission as well as formatting and authentication via `SmtpTransport`.

### Testing
This module provides utilities and helper classes to assist with testing mail related components. Among the many classes, we can note the following key functionalities:

#### Test Message Creation
The module provides both a mock message implementation `TestMessage` as well as builders and DSLs (`TestMessageBuilder` and `buildMessage()`) for creating test messages.

#### Mail-Specific Testing Tools
Custom assertions for email objects are provided in `MessageExtensions` with the aim of simplifying the tests. String helper utilities are also provided in the base package to handle linebreaks in messages.

#### Security Testing Support
Mock implementations of security components are provided to test different security scenarios, such as `VeryTrustingTrustManager` which allows all certificates to pass through and `FakeTrustManager` which conditionally allows a certificate based on a flag. 

## Backend modules
The Backend Modules implement protocol-specific functionality, relying only on Mail modules. Three protocols (JMAP, IMAP, and POP3) are supported in their own module, with each module having the same overall structure. Also included are a demo module for non-production demos, as well as a testing module that provides test implementations of `BackendFolder` and `BackendStorage` to reuse across each protocol modules tests.

```mermaid
graph BT
    api
        api --> mail.common
    demo
        demo --> api
    testing
        testing --> api
    imap
        imap --> api
        imap --> mail.protocols
    jmap
        jmap --> api
    pop3
        pop3 --> api
        pop3 --> mail.protocols

    mail.protocols
    mail.common
```

The `api` module defines common interfaces for each protocol module to implement, with the main focus being the `Backend` interface. This interface defines methods for syncing, downloading, uploading and deleting messages, among other common functions. It also allows the implementing class to define flags showcasing the protocols support for certain functions, such as uploading data.

The three protocol modules all follow a similar structure:
- `Command` classes act as use cases, with each command having a single purpose. For example, both `imap` and `pop3` declare a `CommandDownloadMessage` class but with different implementations.
- `Backend` classes implement the `Backend` interface from the `api` module, with the appropriate flags set based on the protocols supported features and implementation for said features. For example, `Pop3Backend` does not support moving messages, and so marks `supportsMove` as false and `moveMessages()` throws `UnsupportedOperationException`. The majority of the supported features rely on the `Command` classes.
- `Sync` classes provides the concrete implementation of each features synchronization feature. It does not implement any interface, but instead provides methods relevant to the protocols functionality. For example, `ImapSync` provides `sync()` and `downloadMessage()` functions.

## Core modules
```mermaid
---
config:
  layout: elk
---
graph
    android
        android --> common
    mail
        mail --> common
    common
    featureFlags
    outcome
    preferences
    testing
    ui
        ui --> testing
```

### Android
This module provides Android platform specific functionality that relies on Android APIs. The following features are implemented:

- `common` provides several Android-related utlities such as Context and Cursor extensions, as well as a repository to read contacts from the Android system.
- `logging` allows writing logs to local files.
- `network` provides a simplified wrapper over Android's `ConnectivityManager` class to handle network state monitoring, with handling for multiple different versions of Android.
- `permissions` provides a simplified abstraction for checking the state of Android permissions. 
- `testing` provides a `Robolectric` abstraction to support tests that rely on Android functionality, such as testing URIs. 

### Common
This module provides commonly used structures and utilities to be used throughout the project:

- `cache` provides a generic caching mechanism along with three implementations:
  - `ExpiringCache` clears its data after a set period.
  - `InMemoryCache` only holds data for the lifecycle of the app
  - `SynchronizedCache` is a wrapper over other caches to implement thread safe operations.
- `validation` provides abstractions for validation results, similar to Kotlin's `Result` class.
- `mail` provides e-mail address and domain data classes, as well as constants for e-mail protocols and common tokens. These can all be migrated to the `mail` module to organize related components together.
- `net` provides value classes to wrap values into validated Domain, Host and Port classes.
- `oauth` provides classes to represent OAuth configurations as well as relevant factory and provider interfaces.
- `provider` provides interfaces to allow the apps to provide basic information such as app name and brand name.

### Feature Flags
This module defines a structure for defining feature flags in the app. The feature flags are declared per module by implementing the `FeatureFlagFactory` interface. One thing to note is that keys for the flags are not explicitly defined, meaning the implementing class can create as many keys as they want with whatever name they desire. This allows for more flexibility from the factory side, but also means that when checking the flags for a specified feature, the caller does not have any reference for if a key exists or not.

```kotlin
// Feature Flag provider
class TbFeatureFlagFactory : FeatureFlagFactory {
    override fun createFeatureCatalog(): List<FeatureFlag> {
        return listOf(
            FeatureFlag("archive_marks_as_read".toFeatureFlagKey(), enabled = true),
            // ...
        )
    }
}

// Feature Flag caller
// ArchiveOperations.kt
private fun archiveMessages() {
    val operation = featureFlagProvider
        // How can we verify that this is a valid feature flag?
        .provide("archive_marks_as_read".toFeatureFlagKey())
        .whenEnabledOrNot(
            onEnabled = { MoveOrCopyFlavor.MOVE_AND_MARK_AS_READ },
            onDisabledOrUnavailable = { MoveOrCopyFlavor.MOVE },
        )
    // ...
}
```

### Mail
This module contains classes that hold folder information, as well as an enum defining a mail server direction. However, there are two major issues with this module:

1. The `MailServerDirection` enum is only used in one place ( `LocalKeyStoreManager`) and even then the related functions are never called anywhere. This enum can thus be removed without any issue, along with the unused keystore manager functions.
2. The folder classes may be moved to the `feature/folder` module to consolidate related features in one place.

### UI
This module provides UI for both legacy XML approaches and the modern Jetpack Compose approach. 

#### Legacy

For Legacy UI, the module provides an `Icons` object to wrap the XML icons into a more readable format, as well as XML files containing the theme data for both k9mail and thunderbird.

#### Jetpack Compose

For Jetpack Compose, the module provides much more utility. Compared to the relatively barebones Legacy UI module, this module provides:

- A deep theming scheme following Material Design principles (e.g., shapes and elevation)
- Composable components in multiple tiers:
  - **Atoms** are composables at the smallest level: Buttons, Text, Images, etc.
  - **Molecules** are composables that are slightly more complex than atoms: They generally combine multple atoms into a single reusable component such as `CheckboxInput`.
  - **Organisms** are generally more complex than molecules and typically override from Material components, such as `AlertDialog` or `NavigationDrawerItem`
- Extensions and utility methods to assist with composable styling, previews, preferences and navigation

The module also defines `BaseViewModel` and `UnidirectionalViewModel`, which the Feature ViewModels implement. This will be deeper dived in the [Points of Interest](#mvi-architecture) section below

### Others
There are several other minor though equally important submodules in the `core` module. The `preferences` module is concerned with user settings storage. The `outcome` module provides an `Outcome` wrapper class to identify success and failure states (Similar to Kotlin's `Result` class). The `testing` module provides test utilities similar to the other test modules.

## Other Modules 
The following module groups below each contain their own submodules, and each submodule is completely isolated from all other modules in the project with the exception of `cli/html-cleaner`, which relies on `library/html-cleaner`

```mermaid
---
config:
  layout: elk
---
graph
    subgraph cli
        direction TB
        autodiscovery
        cli-html-cleaner[html-cleaner]
        resource-mover
        translation
    end
    subgraph ui-utils
        direction TB
        ItemTouchHelper
        LinearLayoutManager
        ToolbarBottomSheet
    end
    subgraph library
        direction TB
        lib-html-cleaner[html-cleaner]
        TokenAutoComplete

        cli-html-cleaner --> lib-html-cleaner
    end
    subgraph plugins
        direction TB
        openpgp-api-lib
    end
```

### CLI
This module provides multiple command-line utilities, most likely to assist with development tasks. Each module makes use of the [Clikt](https://ajalt.github.io/clikt/) library to simplify command-line development.

- `autodiscovery`: Fetches mail server settings for a given email address.
- `html-cleaner`: Sanitizes HTML input via the `library/html-cleaner` module.
- `resource-mover`: Move string resources from one module to another.
- `translation`: Track translation completion progress and supported languages. The translation is handled by [Weblate](https://weblate.org/en/).

### UI Utils
This module provides customized versions of specific AndroidX components to support Thunderbirds specific requirements.

- `ItemTouchHelper` is a fork of RecyclerView's `ItemTouchHelper` to support swipe gestures that do not remove the swiped item in a list
- `LinearLayoutManager` is a fork of RecyclerView's `LinearLayoutManager` to update the anchored view when the list is currently scrolled to the top and a new item is added to the top of the list. The existing `LinearLayoutManager` retains the same scroll position, so the new item won't be immediately visible.
- `ToolbarBottomSheet` is a fork of `BottomSheetDialogFragment` to support showing a Toolbar when the sheet is fully expanded.

### Library
This module contains two submodules: `html-cleaner` and `TokenAutoComplete`

`html-cleaner` is as the module name implies: It is a utility library whose function is to sanitize, or _clean_, HTML input by only allowing a whitelist of tags to remain in the outputted HTML. It makes use of the [`jsoup`](https://github.com/jhy/jsoup) HTML Parser library to facilitate the cleaning.

`TokenAutoComplete` is a fork of [Splitwise's TokenAutoComplete library](https://github.com/splitwise/TokenAutoComplete). Its purpose is to convert text into user-interactable tokens, such as email addresses in the email recipient field.

### Plugins
The plugins module contains only one submodule: `openpgp-api`, which is an implementation of the [OpenPGP standard](https://www.openpgp.org/) for encrypting and decrypting e-mails.

One notable issue with this module is the lifecycle awareness implementation of `OpenPgpApiManager`. It currently implements the `LifeCycleObserver` interface which is not recommended; instead it shoud implement either `DefaultLifeCycleObserver` to handle all events, or `LifecycleEventObserver` to handle specific events. It also annotates certain methods with the deprecated `@OnLifecycleEvent`, resulting in the lifecycle methods potentially not getting called.

# Points of Interest

## Kotlin

### Java Usage
The project contains a mixture of both Kotlin and Java code, most notably in older modules such as `legacy` and `mail`. As per the [project wiki](https://github.com/thunderbird/thunderbird-android/wiki/CodeStyle#java), all new code is to be written in Kotlin. However, it seems that the migration from Java to Kotlin is done sporadically whenever a contributor works on an issue.

### Usage of Not-null assertion operator
There are over 100 usages of the double bang (`!!`) operator in the project. While this may be used in cases where the compiler cannot confirm if a variable is null or not despite the developer confirming this with absolute certainty, there are many places where there is no null check in place. For example, take the `SmtpTransport` class:

```kotlin
class SmtpTransport(
    //...
    private val oauthTokenProvider: OAuth2TokenProvider?,
) {
    private fun attempOAuth(method: OAuthMethod, username: String) {
        val token = oauthTokenProvider!!.getToken(OAuth2TokenProvider.OAUTH2_TIMEOUT.toLong())
        //...
    }
}
```

There is no null check made, so if the token provider is null for any reason, this will throw a `RuntimeException` resulting in a crash. As an improvement for this case, we should either use a null-safety operator (`oauthTokenProvider?.getToken`) or make `oauthTokenProvider` non-null to enforce null safety.

## Modules

### App Common
As mentioned above, despite serving as a "common" point between the main app modules, all the code in this module could be migrated to the `feature/account` module instead, rendering this module redundant.

### Legacy modules
Even though many of the modules are defined as *legacy* modules, they are still actively used in the app with little to no sign of deprecation. The main entry point of the app (`MessageList`) is located in the `legacy.ui.legacy` module.

### Testing modules
There are several `testing` modules defined, each of which provides classes to assist with testing, such as assertion extensions or fakes of certain interfaces. These modules are only included as test dependencies via `testImplementation` and allow these files to be reused across multiple modules, rather than having to duplicate code in every module/test that requires it. For example, `core.testing` provides the following List assertion extension:

```kotlin
fun <T> Assert<List<T>>.containsNoDuplicates() = given { actual ->
    val seen: MutableSet<T> = mutableSetOf()
    val duplicates = actual.filter { !seen.add(it) }
    if (duplicates.isNotEmpty()) {
        expected("to contain no duplicates but found: ${show(duplicates)}")
    }
}
```

This allows for writing simpler tests such as:
```kotlin
// NotificationIdsTest.kt
@Test
fun `all general notification IDs are unique`() {
    val notificationIds = getGeneralNotificationIds()

    assertThat(notificationIds).containsNoDuplicates()
}
```

## MVI Architecture

The Feature modules make use of the MVI architecture, the foundation of which is defined by `BaseViewModel` and `UnidirectionalViewModel`, as well as a third `Contract` class defined by each ViewModel.

### UnidirectionalViewModel

The `UnidirectionalViewModel` interface has three type arguments: 
- `STATE` represents the state of the data to be displayed. It differs from the typical MVVM implementation in that the data presented to the screen is wrapped in a single object, while MVVM allows for multiple streams of data to be observed at once.
- `EVENT` represents calls to action, typically through user events such as tapping a button, but may also be from other sources such as receiving a push notification or an update to network connectivity.
- `EFFECT` represents the outcome of a particular event. For example, after creating an account with `Event.CreateAccount`, the ViewModel may emit `Effect.NavigateNext` on success or `Effect.ShowError` on failure.

### BaseViewModel

`BaseViewModel` is an abstract `ViewModel` class that implements `UnidirectionalViewModel` and handles basic functionality to manage state and emit effects. It also provides an implementation for handling one-time events.

### Contracts

ViewModels make use of these in conjunction with a third interface as a contract. Each ViewModel defines its own contract, with the following common structure:

```kotlin
interface Contract {
    interface ViewModel: UnidirectionalViewModel<State, Event, Effect>

    data class State(
        val someStateValue: String,
    )

    sealed interface Event {
        data object SomeEvent: Event
    }

    sealed interface Effect {
        data object SomeEvent: Effect
    }
}
```

### Usage

As an example, the Create Account feature would define a `CreateAccountContract` with definitions of the expected state, events, and effects: 

```kotlin
interface CreateAccountContract {
    interface ViewModel : UnidirectionalViewModel<State, Event, Effect>
    data class State(
        val isLoading: Boolean = true,
        val error: Error? = null,
    )
    sealed interface Event {
        data object CreateAccount : Event
        data object OnBackClicked : Event
    }
    sealed interface Effect {
        data class NavigateNext(val accountUuid: AccountUuid) : Effect
        data object NavigateBack : Effect
    }
}
```

We would then construct the `CreateAccountViewModel` class extending both `BaseViewModel` and `CreateAccountContract` as follows:
```kotlin
class CreateAccountViewModel(
    initialState: State = State(),
) : BaseViewModel<State, Event, Effect>(initialState), 
    CreateAccountContract.ViewModel {
        override fun event(event: Event) {
            when (event) {
                Event.OnBackClicked -> emitEffect(Effect.NavigateBack)
                Event.CreateAccount -> { /*TODO*/ }
            }
        }
    }
```

This is then observed from the `CreateAccountScreen`:
```kotlin
@Composable
fun CreateAccountScreen(
    onBack: () -> Unit,
    viewModel: CreateAccountContract.ViewModel,
) {
    val (state, dispatch) = viewModel.observe { effect -> 
        when (effect) {
            Effect.navigateBack -> onBack()
        }
    }

    CreateAccountContent(
        state = state,
        onClickBack = { dispatch(Event.onBackClicked) }
    )
}
```

### Benefits

There are numerous advantages with this pattern, notably:

1. **The actions and reactions of the screen are clearly defined.** Each `Event` and `Effect` must be declared by it associated contract. This can help with clarity when checking what the capabilities and expected outputs are.
2. **Events and Effects are consolidated in one place.** Events are handled by the ViewModel's `onEvent` method, and Effects are handled by the screen observing with `val (state, dispatch) = viewModel.observe {}`. This makes it quick to find the entry/exit point of a code path.
3. **Simplified UI Testing.** As the composables now take in a relatively simple interface, it is much simpler to create Fakes of the ViewModel contracts.
4. **Reduction in callback parameters.** Rather than passing a unique lambda function for each action the user can take, we can pass a single `dispatch: (Event) -> Unit` argument. For example:
    ```kotlin
    SettingsScreenContent(
        dispatch: (SettingsContract.Event) -> Unit,
    )
    // VS
    SettingsScreenContent(
        onClickProfile: () -> Unit,
        onClickSecurity: () -> Unit,
        onClickHelp: () -> Unit,
        // ... Other events
    )

## Specific Classes and Files

### Logger
The projects provides an interface `Logger` as an abstraction for it's logging purposes. Many projects make use of `Timber`, but this does not work on non-Android modules. The `Logger` interface allows the project to supply different logging implementations to work around this issue, such as `SystemOutLogger`. There is also a fake `Timber` object whose purpose is to temporarily handle the logging with the minimal amount of code change (i.e., updating only the import on relevant files) until Timber supports non-Android classes, but this seems unlikely as the project hasn't had any feature updates [since 2021](https://github.com/JakeWharton/timber/releases).

### RealImapFolderIdler
This class implements IMAP's IDLE command, which is a feature that defines how real-time notifications are received by the client. The general flow of the logic is as follows:

```mermaid
flowchart LR
    start[Send IDLE command]
        start --> isSupported
    isSupported{Is IDLE supported?}
        isSupported -- Yes --> connect
        isSupported -- No --> done
    connect[Establish connection]
        connect --> receive
    receive[Receive Message]
        receive --> relevant
    relevant{Is message relevant?}
        relevant -- Yes --> handle
        relevant -- No --> terminate
    handle[Handle updated message]
        handle --> terminate
    terminate{should terminate connection?}
        terminate -- No --> receive
        terminate -- Yes --> done
    done[End]
```

### Clock
The app makes use of the `Clock` interface from the [`kotlinx.datetime`](https://github.com/Kotlin/kotlinx-datetime) library for time-related code. This allows for simpler testing by allowing the usage of fake clocks rather than having to mock static methods (e.g., `LocalDateTime` from `java.time` library) for each test group. For example, if we were to test an expiring cache:

```kotlin
// With Clock
Class TestClock(var currentTime: Instant = Clock.System.now()): Clock {
    override fun now(): Instant = currentTime
    fun changeTime(time: Instant) { currentTime = time }
    fun advanceTime(duration: Duration) { currentTime += duration }
}

@Test
fun `cache expires after 5 minutes`() {
    val clock: Clock = TestClock()
    val cache = Cache(clock)
    clock.advanceTime(Duration.minutes(5))
    assertThat(cache.isExpired).isTrue()
}

// With LocalDateTime
@Test
fun `cache expires after 5 minutes`() {
    mockkStatic(LocalDateTime::class)
    every { LocalDateTime.now() } returns LocalDateTime.of(2024, 3, 7, 9, 15, 0)
    val cache = Cache(expiresAt = LocalDateTime.now().plusMinutes(5))
    every { LocalDateTime.now() } returns LocalDateTime.of(2024, 3, 7, 9, 21, 0)
    assertThat(cache.isExpired).isTrue()
}
```

## Jetpack Compose

### Previews
Previews of composables are not located in the same file as the composable they are showcasing. Instead, previews are located in the `debug` build type folder. The intention behind this is so that when we build the `release` version of the app, no previews are included in the final APK, resulting in smaller file sizes. However, previews are not included in the final APK when using ProGuard, so this implementation may be unnecessary and the previews may be better organised in the same file as their content.

### PreviewDevices
Annotation class to collate multipe preview annotations into one, allowing previews to show their layouts on multiple devices from small phones up to desktop size screens. This allows the developer to apply one preview annotation and output multiple previews.

<img src="assets/preview_devices.png">

### ResponsiveContent / ResponsiveWidthContainer
Wrappers to adapt content to multiple screen sizes. This allows for the appropriate placement of content on multiple screen sizes when that content is not designed to occupy the full screen size, such as the Welcome Screen. The available screen size classes are defined by the `WindowSizeClass` enum

<img src="assets/responsive_content_welcome.png">

# Conclusion

The Thunderbird Android project has a large, modular codebase that is continuously evolving over time. It's architecture is divided into both major and minor module groups, with the goal of providing clearly defined responsibilities for each. The projects modularity provides a base for isolated feature-focused development, allowing multiple contributors to work on multiple modules with a lower chance of conflicts, though this still has room for improvement especially with reference to the older legacy modules.

The codebase is currently in the process of transitioning from Java and legacy XML UI layouts to Kotlin and Jetpack Compose, but it still retains a sizeable amount of technical debt, such as redundant modules and a mixture of Java and Kotlin within the same module. This should eventually be reduced to a non-issue as older code is updated to follow the latest practices and be less coupled with other modules.

Overall, as Thunderbird Android continues to mature, its commitment to modularity and modern development practices position it well for future development and maintainability. Efforts to refactor legacy components, reduce technical debt, and embrace new technologies should enhance the codebase, ensuring that it remains accessible to contributors to help with development.