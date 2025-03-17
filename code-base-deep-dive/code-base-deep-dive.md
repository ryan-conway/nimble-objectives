# Introduction

This document serves as... `TODO`

# Project Overview

Thunderbird is an Android mail client... `TODO`

# Prerequisites

Working with this project requires prerequisite knowledge and installed software. Ensure having prior knowledge of the following:

- Kotlin
- MVVM
- `TODO`

The following software must also be available:

- IDE (Android Studio preferred)
- Git
- `TODO`

# Building the Project

- Checkout the project via Git from `todo url`
- `TODO`

# Project Structure

`>> TODO: App architecture diagram and overview`

```mermaid
architecture-beta
    group sample(cloud)[API]

    service db(database)[Database] in sample
    service disk1(disk)[Storage] in sample
    service disk2(disk)[Storage] in sample
    service internet(internet)[Internet] in sample
    service server(server)[Server] in sample
    
    junction diskJunction in project

    internet:B -- T:server
    server:R -- L:db
    db:R -- L:diskJunction
    disk1:B -- T:diskJunction
    disk2:T -- B:diskJunction
```

```mermaid
architecture-beta
    group project[Project]

    group app[App] in project
    group backend[Backend] in project
    group core[Core] in project
    group feature[Feature] in project
    group legacy[Legacy] in project

    service activity[Activity] in app
    service viewmodel[ViewModel] in app

    activity:R <--> L:viewmodel

```

# Modules

## App

`TODO`

## Backend

`TODO`

## Core

`TODO`

### Core Module 1 `TODO`

### Core Module 2 `TODO`

## Feature

`TODO`

### Feature Module 1 `TODO`

### Feature Module 2 `TODO`

## Legacy Modules

`TODO`

# Points of Interest

`>> TODO: discuss modules/class/files of note`
<br> `>> - Interesting code`
<br> `>> - Points to improve`
<br> `>> - Unusual designs`

# Conclusion

`>> TODO: Summarize above content and draw conclusion`