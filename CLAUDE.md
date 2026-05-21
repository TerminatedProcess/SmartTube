# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

SmartTube is a free, open-source media client for Android TVs and TV boxes (no Google Services required). It targets Android 4.3+ (`minSdk 19`, `stfdroid` flavor uses 21), runs only on leanback/TV form factors, and is built around the AndroidX Leanback UI.

## Critical build requirements

- **JDK 14 or older is required.** Newer JDKs cause runtime crashes. This is non-negotiable per the README.
- **Git submodules must be initialized before building.** The build depends on two submodules that are *not* checked out by default in this repo:
  - `MediaServiceCore` → provides the `:youtubeapi` and `:mediaserviceinterfaces` modules (the YouTube backend / data layer).
  - `SharedModules` → provides `:sharedutils`, `:appupdatechecker2`, and other shared modules via its `core_settings.gradle`.
  - Run `git submodule update --init` first. Without these, Gradle module resolution in `settings.gradle` fails.
- Gradle wrapper is 7.5; Android Gradle Plugin is 7.4.2.

## Common commands

```bash
git submodule update --init                  # required once, before any build
./gradlew clean installStstableDebug         # build + install stable debug to connected device
./gradlew assembleStbetaDebug                # build beta debug APKs (no install)
adb connect <device_ip>                      # connect to a TV/box over network first
./gradlew :common:testStstableDebugUnitTest  # run common module unit tests (Robolectric/JUnit)
./gradlew test                               # all unit tests
```

Build variant = flavor + build type, e.g. `Ststable` + `Debug` → `installStstableDebug`. Output APKs are renamed to `SmartTube_<flavor>_<version>_<arch>.apk` (see `applicationVariants` in `smarttubetv/build.gradle`).

## Product flavors

Three flavors under the single `default` dimension, each a distinct `applicationId`:
- `stbeta` (`org.smarttube.beta`) — new features first; includes Firebase Crashlytics.
- `ststable` (`org.smarttube.stable`) — includes Firebase Crashlytics.
- `stfdroid` (`app.smarttube.fdroid`) — F-Droid build, `minSdk 21`, no Firebase/Google services.

Firebase (Crashlytics + `google-services.json`) is only wired in for `stbeta`/`ststable` builds (guarded in `smarttubetv/build.gradle`). The fdroid flavor must build without it. APK signing reads from a `keystore.properties` file at repo root if present; absent = unsigned.

## Module layout & architecture

This is a multi-module Gradle project. The big picture:

- **`:smarttubetv`** — the Android application module (TV UI layer only). Package `com.liskovsoft.smartyoutubetv2.tv`. Contains Leanback `Activity`/`Fragment`/`Presenter`/adapter classes — the *View* side of MVP. Entry points: `MainApplication` (a `MultiDexApplication`) and `SplashActivity` (launcher).
- **`:common`** — the bulk of app logic, shared across flavors. Package `com.liskovsoft.smartyoutubetv2.common`. Contains:
  - `app/presenters/` — the *Presenter* layer (MVP). Presenters are **singletons** accessed via a static `instance(Context)` method (e.g. `BrowsePresenter.instance(context)`), holding their `View` via a `WeakReference`. `BasePresenter<T>` is the shared base.
  - `app/views/` — View interfaces (`BrowseView`, `PlaybackView`, etc.) implemented by classes in `:smarttubetv`. `ViewManager` handles navigation between them.
  - `app/models/` — data models (`Video`, `VideoGroup`, `Playlist`) and playback controllers.
  - `prefs/` — persistent settings as singleton "Data" classes (`GeneralData`, `MainUIData`, `PlayerData`, `PlayerTweaksData`, `SponsorBlockData`, etc.) backed by `AppPrefs`. This is where most user-facing options live.
  - `exoplayer/` — ExoPlayer integration (track selection, controllers, error handling, version shims).
  - `autoframerate/` — auto frame-rate switching (AFR) for TVs.
- **`MediaServiceCore` (submodule)** — `:youtubeapi` + `:mediaserviceinterfaces`. The data/backend layer; presenters talk to it through `MediaServiceManager` and the `*Service` interfaces (`ContentService`, `MediaItemService`, `SignInService`, etc.). Reactive (RxJava2).
- **`SharedModules` (submodule)** — `:sharedutils` (cross-cutting helpers), `:appupdatechecker2` (in-app updater), and more.

### Bundled / forked local modules

Several dependencies are vendored as local Gradle modules rather than pulled from Maven, because they are patched:
- **`exoplayer-amzn-2.10.6/`** — a fork of the Amazon ExoPlayer port (itself a fork of Google ExoPlayer). Included via its own `core_settings.gradle` with module prefix `exoplayer-` (`:exoplayer-library-core`, `:exoplayer-extension-leanback`, etc.).
- **`leanback-1.0.0/`** and **`fragment-1.1.0/`** — modded copies of AndroidX Leanback/Fragment. The root `build.gradle` **excludes** the upstream `androidx.leanback:leanback` and `androidx.fragment:fragment` so these local forks are used instead. Do not re-add the upstream artifacts.
- **`doubletapplayerview/`**, **`slidableactivity/`**, **`filepicker-lib/`**, **`chatkit/`**, **`leanbackassistant/`** — other bundled UI/feature modules.

### Dependency-resolution gotchas

The root `build.gradle` `allprojects.configurations.all` block force-pins many versions (okhttp, kotlin stdlib, coroutines, androidx core/annotation, cronet, multidex) for **Android 4.x backward compatibility** and to satisfy WorkManager. Be very cautious bumping these — the constraints exist to keep the app running on very old devices and to avoid `VerifyError`/`NoSuchMethodError`. Minification (R8/ProGuard) is currently disabled for release; `multidex-keep.pro` lists classes that must stay in the primary dex.

## Conventions

- The codebase is primarily **Java** (some Kotlin in `:common`). Reactive flows use **RxJava2**.
- Persistent feature toggles → add to the appropriate `prefs/*Data` singleton, not ad-hoc SharedPreferences.
- UI lives in `:smarttubetv`; logic that should be testable or shared belongs in `:common`.
- User-facing strings live in `common/src/main/res/values/strings.xml` and are translated via Crowdin (see `crowdin.yml`); do not hand-edit `values-<lang>/strings.xml`.
