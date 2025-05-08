# Introduction
This document serves as an overview of Thunderbird Android's code base. It will analyze the project architecture and evaluate technical decisions and points of interest.

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

The most major modules to note are the App, Legacy, Feature, Backend and Core modules.

`TODO deeper dive into overall structure?`

# Modules

## App modules
The App modules, which serve as the entry points or highest level of the app architecture, depend on all other modules both directly and indirectly. They are responsible for defining which features are presented to the users and how the dependency tree is built.

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

`TODO expand on features?`

## Mail modules
The Mail modules are the most foundational modules of the app, responsible for handling mail messages and interacting with different email protocols.

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

#### 

### Protocols

The three sub-packages here provide protocol-specific implementations for three major email protocols: `IMAP`, `POP3` and `SMTP`.

`IMAP` is used for synchronization-based email access, meaning that all messages and their associated states are kept on the server. This module add support for synchronization via `RealImapFolder` and push notifications via `RealImapFolderIdler`.

`POP3` is used for download-based email access, meaning that all messages will be downloaded to the device.

`SMTP` is used for sending messages, and thus handles message transmission as well as formatting and authentication via `SmtpTransport`.

### Testing
This module provides utilities and helper classes to assist with testing mail related components. Among the many classes, we can note the following key functionalities:

**Test Message Creation**
</br>
The module provides both a mock message implementation `TestMessage` as well as builders and DSLs (`TestMessageBuilder` and `buildMessage()`) for creating test messages.

**Mail-Specific Testing Tools**
</br>
Custom assertions for email objects are provided in `MessageExtensions` with the aim of simplifying the tests. String helper utilities are also provided in the base package to handle linebreaks in messages.

**Security Testing Support**
</br>
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
- `Backend` classes implement the `Backend` interface from the `api` module, with the appropriate flags set based on the protocols supported features and implementation for said features. For example, `Pop3Backend` does not support moving messages, and so marks `supportsMove` as false and `moveMessages()` throws `UnsupportedOperationException`. The majority of the supported features rely on the `Command` classes
- `Sync` classes provides the concrete implementation of each features synchronization feature. It does not implement any interface, but instead provides methods relevant to the protocols functionality. For example, `ImapSync` provides `sync()` and `downloadMessage()` functions.

## Core modules
```
- Core Modules provide essential services used by nearly all other module groups.
```
The Core modules `TODO`

```mermaid
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

### Common

### Feature Flags

### Mail

### Outcome

### Preferences

### Testing

As with other testing modules, this module only provides test related code for reuse, namely assertion extensions to simplify tests as well as a `Clock` implementation.

### UI

This module provides UI for both legacy XML approaches and the modern Jetpack Compose approach. 

For Legacy UI, the module provides an `Icons` object to wrap the XML icons into a more readable format, as well as XML files containing the theme data for both k9mail and thunderbird.

For Jetpack Compose, the module provides much more utility. Compared to the relatively barebones Legacy UI module, this module provides:

- A deep theming scheme following Material Design principles (e.g., shapes and elevation)
- Composable components in multiple tiers:
  - **Atoms** are composables at the smallest level: Buttons, Text, Images, etc.
  - **Molecules** are composables that are slightly more complex than atoms: They generally combine multple atoms into a single reusable component such as `CheckboxInput`.
  - **Organisms** are generally more complex than molecules and typically override from Material components, such as `AlertDialog` or `NavigationDrawerItem`

## Other Modules 

### CLI
`TODO`

### UI Utils
`TODO`

### Library
`TODO`

### Plugin
`TODO`

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

There is no null check made, so if the token provider is null for any reason, this will throw a `RuntimeException` resulting in a crash.

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
Class TestClock(var currentTime: Instant = Clock.System.now()) {
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

The project presents multiple interesting Jetpack Compose design choices. The most notable are described below:

### Previews

Previews of composables are not located in the same file as the composable they are showcasing. Instead, previews are located in the `debug` build type folder. This means that when we build the `release` version of the app, no previews are included in the final APK, resulting in both smaller file sizes and more secure code. `TODO find preview thing from android template`

### PreviewDevices

Annotation class to collate multipe preview annotations into one, allowing previews to show their layouts on multiple devices from small phones up to desktop size screens. `TODO expand on benefits`

### ResponsiveContent

Wrapper to adapt content to multiple screen sizes `TODO expand on benefits`

# Conclusion

`>> TODO: Summarize above content and draw conclusion`



















a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a

a