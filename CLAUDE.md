# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

**Prerequisite**: OpenJDK 14 or older is required. Newer JDKs cause runtime crashes.

Initialize submodules before first build:
```
git submodule update --init
```

Build and install to a connected ADB device:
```
# Connect device first
adb connect <device_ip_address>

# Install debug build (storig flavor is cleanest for development)
./gradlew clean installStorigDebug

# Other flavors
./gradlew installStbetaDebug    # beta channel
./gradlew installStstableDebug  # stable channel
./gradlew installStamazonDebug  # Amazon Fire TV
```

Run unit tests:
```
./gradlew :smarttubetv:test
./gradlew :common:test
```

Run a single test class:
```
./gradlew :smarttubetv:testStorigDebugUnitTest --tests "com.liskovsoft.smartyoutubetv2.tv.SomeTest"
```

## Architecture

### Module Structure

- **`smarttubetv/`** — Android TV UI layer. Contains all Activities, Fragments, Leanback Presenters, and custom Views. No business logic.
- **`common/`** — Platform-agnostic business logic: presenters, view interfaces, prefs, ExoPlayer integration, and utility services.
- **`MediaServiceCore/`** (git submodule) — YouTube API client. Provides `ContentService`, `MediaItemService`, `SignInService`, etc. via the `mediaserviceinterfaces` module.
- **`SharedModules/`** (git submodule) — Shared Android utilities used across Liskovsoft projects (`sharedutils`).
- **`exoplayer-amzn-2.10.6/`** — Forked Amazon ExoPlayer 2.10.6 bundled directly (not fetched from Maven).
- **`leanback-1.0.0/`**, **`fragment-1.1.0/`** — Patched local copies of AndroidX Leanback and Fragment libraries (the stock libraries are excluded via Gradle resolution strategy).

### MVP Pattern

The app follows MVP strictly:

- **View interfaces** (`common/app/views/`) define what the UI can do: `BrowseView`, `PlaybackView`, `SearchView`, etc.
- **Presenters** (`common/app/presenters/`) hold all logic. They keep a `WeakReference<T>` to their view. Most are singletons accessed via `getInstance(Context)`.
- **UI implementations** (`smarttubetv/tv/ui/`) are Fragments/Activities that implement the view interfaces and call `presenter.setView(this)` in `onStart()`.

`ViewManager` (singleton in `common/app/views/`) manages navigation between Activities using a `Map<ViewInterface, Activity>` mapping and an Activity back-stack.

### Playback Pipeline

`PlaybackPresenter` (singleton) orchestrates playback through a chain of `BasePlayerController` subclasses, all registered as `PlayerEventListener`:

1. `VideoStateController` — Saves/restores playback position
2. `SuggestionsController` — Loads related video suggestions
3. `PlayerUIController` — Controls UI state (buttons, overlays)
4. `VideoLoaderController` — Loads video format info and opens streams
5. `ContentBlockController` — SponsorBlock segment skipping
6. `AutoFrameRateController` — AFR (Auto Frame Rate) switching
7. `RemoteController` — YouTube remote control / casting
8. `ChatController` — Live chat integration
9. `HQDialogController` — Quality/format picker dialog

`ExoPlayerController` implements the `PlayerEngine` interface and wraps `SimpleExoPlayer`. The `PlaybackFragment` (TV layer) hosts the ExoPlayer instance and implements `PlaybackView`.

### Settings / Preferences

All persisted settings live in `common/prefs/` as singleton classes backed by `SharedPreferences` (via `AppPrefs`). Each domain has its own class: `PlayerData`, `GeneralData`, `MainUIData`, `ContentBlockData`, `PlayerTweaksData`, `RemoteControlData`, etc. Settings UI is built programmatically in `common/app/presenters/settings/`.

### Leanback UI

The browse screen uses AndroidX Leanback (`BrowseFragment` → `BrowseSupportFragment` from the patched local library). Video rows are powered by `VideoGroupObjectAdapter` wrapping `VideoGroup` model objects. Card rendering uses `VideoCardPresenter` (standard thumbnail cards) or `ShortsCardPresenter`/`TinyCardPresenter` for alternate layouts.

The player UI is built on Leanback's `PlaybackSupportFragment` with extensive modifications under `tv/ui/playback/mod/` and `tv/ui/mod/leanback/playerglue/` to add features (tooltips, frame-drop handling, seek preview, additional control buttons).

### Build Flavors

| Flavor | Application ID | Notes |
|--------|---------------|-------|
| `stbeta` | `com.liskovsoft.smartyoutubetv2.beta` | Firebase Crashlytics enabled |
| `ststable` | `com.teamsmart.videomanager.tv` | |
| `storig` | `org.smartteam.smarttube.tv.orig` | Best for development |
| `stamazon` | `com.amazon.firetv.youtube` | Amazon Fire TV |
| `staptoide` | `com.teamsmart.videomanager.tv` | Aptoide store, bumped versionCode |

APK output names follow the pattern: `SmartTube_<flavor>_<version>_<abi>.apk`.

### RxJava Usage

Async operations (API calls, background work) use RxJava 2. Disposables are typically managed via `RxHelper` from SharedModules. Presenters subscribe on `Schedulers.io()` and observe on `AndroidSchedulers.mainThread()`.

### Key Entry Points

- `MainApplication` — Application class; initializes logging, crash reporting
- `SplashActivity` → `SplashPresenter` — First screen, decides where to navigate
- `BrowseActivity` / `BrowseFragment` — Main home screen
- `PlaybackActivity` / `PlaybackFragment` — Video player
- `AppDialogActivity` / `AppDialogFragment` — Reusable settings/option dialog shell used for all in-app dialogs
