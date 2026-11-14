# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

SmartTube is an Android TV (leanback) YouTube client in Java with some Kotlin.

## This checkout is a personal fork

- `origin` = github.com/beta7xx/SmartTube, `upstream` = github.com/yuliskov/SmartTube.
- `main` holds the user's own changes; `upstream/master` is the upstream code. Update by merging `upstream/master` into `main`.
- The submodules (`MediaServiceCore`, `SharedModules`, and `MediaServiceCore/SharedModules`) still point to yuliskov's repos and can't be pushed. Keep edits to them as patch files in `patches/`, not as commits inside a submodule. `.gitattributes` marks `patches/*.patch -text` because the submodule sources are stored with CRLF.
- After any `git submodule update`, re-apply the patches:
  ```
  git -C MediaServiceCore reset --hard
  git submodule update --init --recursive
  git -C MediaServiceCore apply ..\patches\mediaservicecore-no-remote-constants.patch
  git -C MediaServiceCore apply ..\patches\mediaservicecore-plain-text-pairing-code.patch
  ```
- What each patch does:
  - `mediaservicecore-no-remote-constants.patch` – stops fetching `constants.json` from the upstream owner's GitHub (and deletes the test for that fetch). If it is missing, the app fetches that file again.
  - `mediaservicecore-plain-text-pairing-code.patch` – fixes "Add device" (link with TV code). YouTube's `get_pairing_code` endpoint answers in `text/plain`, but upstream parsed it as JSON (`$.code`) and got nothing. Drop this patch once upstream fixes it; it will then fail to apply.

## Build, install, test

The submodules must be initialised first (`git submodule update --init --recursive`). If `ANDROID_HOME` isn't set, point it at `%LOCALAPPDATA%\Android\Sdk`.

```
gradlew assembleStfdroidDebug        # APKs: smarttubetv/build/outputs/apk/stfdroid/debug/ (one per ABI + universal)
gradlew installStfdroidDebug         # build and install via adb in one step
gradlew :common:testStfdroidDebugUnitTest --tests "*SuggestionsControllerTest"   # single test
gradlew :smarttubetv:lintStfdroidDebug
```

- **Flavors** (`smarttubetv/build.gradle`): `stbeta`, `ststable` and `stfdroid`. Firebase Crashlytics is applied only to `stbeta`/`ststable`. The update checker URLs are only in `common/src/stbeta|ststable/res/values/update_urls.xml`. Use **`stfdroid`** for personal builds.
- **JDK:** use OpenJDK 17, the version the upstream CI builds with (`.github/workflows/CI.yml`). The Robolectric unit tests (`common` and `:youtubeapi`) can't run on it: Robolectric 4.6.1 can't read Java 17 class files (`Unsupported class file major version 61`, surfacing as `NoClassDefFoundError: android/webkit/RoboCookieManager`). This machine has only JDK 17, so verify changes on the TV instead. Many `:youtubeapi` tests also call the live YouTube API.
- **Gradle:** the wrapper is Gradle 7.5 with `distributionSha256Sum` pinned. Gradle versions and SDK versions come from `SharedModules/constants.gradle`.
- **Target TV:** a Sony BRAVIA on Android 9, 32-bit, so use the `armeabi-v7a` APK:
  ```
  adb connect 192.168.1.7:5555
  adb install -r <apk>
  ```
  The user keeps ADB enabled on purpose. Debug builds are signed with `~/.android/debug.keystore`; `install -r` only works with that same key.

## Module layout

`settings.gradle` assembles the project from submodules through their `core_settings.gradle` files:

- `smarttubetv` – the app module. It contains Activities and Fragments (`tv/ui/...`), `MainApplication` and the manifest.
- `common` – UI-independent app logic: presenters, playback controllers, preferences, ExoPlayer integration and misc services.
- `MediaServiceCore` (submodule):
  - `mediaserviceinterfaces` – service interfaces such as `ServiceManager`, `MediaItemService`, `SignInService` and `ContentService`.
  - `youtubeapi` – their implementation, which talks to YouTube's private InnerTube/TV APIs through Retrofit. It also contains SponsorBlock, DeArrow, Return YouTube Dislike, PoToken generation (WebView/V8) and the nsig challenge solver (JS in `assets/nsigsolver`, run in J2V8).
- `SharedModules` (submodule) – `sharedutils` (helpers, OkHttp setup in `okhttp/OkHttpCommons.java`, prefs base), `appupdatechecker2` and `j2v8` (a prebuilt `libj2v8.so`).
- `exoplayer-amzn-2.10.6` – a vendored, modified ExoPlayer. Modules are included with the `exoplayer-` prefix.
- `leanback-1.0.0` and `fragment-1.1.0` – vendored, patched copies of AndroidX libraries. `leanbackassistant` provides launcher channels and the search content provider.
- `chatkit`, `filepicker-lib`, `doubletapplayerview` and `slidableactivity` – vendored UI libraries.

## Architecture (MVP across `common` ↔ `smarttubetv`)

- **Views** are interfaces in `common/.../app/views` (for example `BrowseView` and `PlaybackView`). **Presenters** are singletons in `common/.../app/presenters` that get a `Context` and drive their view.
- `ViewManager` (`common/.../app/views/ViewManager.java`) maps view interfaces to Activities. The mapping is registered in `MainApplication.setupViewManager()`. Presenters open screens with `ViewManager.startView(SomeView.class)` and never reference Activity classes directly. A new screen needs a View interface, a Presenter, an Activity/Fragment and a `register(...)` call.
- **Playback**: `PlaybackPresenter` owns a list of controllers (`common/.../models/playback/controllers`, all extending `BasePlayerController`). Player engine and UI events are broadcast to every controller in registration order. Feature logic lives in its own controller (SponsorBlock, AFR, chat, error fixing and so on), not in the presenter.
- **Settings UI** is built from code, not XML: `common/.../presenters/settings/*SettingsPresenter` fill an `AppDialogPresenter` with `OptionItem`/`UiOptionItem` entries.
- **Preferences**: `common/.../prefs/*Data` classes (for example `PlayerData` and `GeneralData`) persist through `AppPrefs`. Account tokens live in `GlobalPreferences` (sharedutils), which is included in local backups. `HiddenPrefs` is excluded from backups.
- Data access from `common` goes through `MediaServiceManager` (in `common`) or `YouTubeServiceManager`, using RxJava 2 observables. `RxHelper` provides the threading helpers.

## Network endpoints outside Google/YouTube (audit baseline)

When reviewing upstream merges, diff for new `http(s)://` strings. Exclude test fixtures: `MediaServiceCore/youtubeapi/src/test` is full of captured YouTube JSON. The known endpoints are:

- `sponsor.ajay.app`, `dearrow-thumb.ajay.app`, `returnyoutubedislikeapi.com`, `api.qrserver.com` and `flagcdn.com`.
- DoH providers in `DohProviders.kt`.
- `github.com/yt-dlp/ejs` – solver JS fallback, downloaded only when the built-in and cached copies are unusable; there is no hash check.

Inbound listeners: `common/.../misc/DialServer.java` (DIAL casting, started from `RemoteControlService`). It is off by default (`RemoteControlData.isDialEnabled()`). When enabled, it listens on TCP 8090 and on UDP 1900 (the SSDP multicast port) on all interfaces, with no authentication. Anyone on the LAN can pair a YouTube sender and control playback, and they can read the device name and model. It is the only server socket in the app.

Upstream-owner endpoints that are disabled or hidden in `main`:

- `constants.json` – removed by the patch.
- The bridge-APK installer button in the About settings – its call is commented out.
- The update checker – absent in `stfdroid`.
