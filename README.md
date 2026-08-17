# Cindy Timer

A private, offline-first iOS timer for the CrossFit benchmark workout "Cindy"
(20-minute AMRAP: 5 Pull-Ups, 10 Push-Ups, 15 Air Squats), plus custom AMRAP /
fixed-round workouts, workout history, streaks, and personal records.

Everything lives on your device. No account, no server, no analytics, no ads.

---

## What's included

- **CindyTimerApp.swift** — app entry point, SwiftData container setup
- **Engine/** — `WorkoutEngine` (the state machine driving every workout) and
  `TimerEngine` (a wall-clock-based countdown that can't drift when the app
  is backgrounded, the phone locks, or a call comes in)
- **Models/** — SwiftData models for workout history and custom workouts
- **Persistence/** — crash/force-quit recovery snapshot, streak & PR math,
  saving a finished workout exactly once
- **Services/** — haptics, optional voice coaching (AVSpeechSynthesizer),
  local notifications, and the settings store
- **Views/** — Home, pre-workout countdown, the active workout screen, rest
  screen, end-of-workout screen, history, progress dashboard, settings, and
  the custom-workout builder
- **CindyTimerTests/** — unit tests for timer accuracy, round/partial-round
  counting, debounce protection, and streak/PR calculations

## Scope notes (read this first)

This build focuses on the highest-priority items from the spec: reliability,
simplicity, accurate timing, workout tracking, motivation, privacy, and
accessibility basics. A few things were intentionally left out of this first
pass so the core experience is solid rather than half-finished everywhere:

- **Live Activities / Dynamic Island** are not implemented. They require a
  separate Widget Extension target (a second, more involved Xcode target
  with its own entitlements and App Group). The app still works great with
  the screen on or off — the timer is wall-clock accurate regardless — but
  there's no lock-screen live view yet. This is a good "phase two" addition.
- **HealthKit** integration isn't included (the spec didn't ask for it, and
  it adds a permission + capability most people won't want for a private
  timer app).
- The **app icon** is a simple placeholder I generated (a radial ring +
  three stacked bars representing the three Cindy movements) — you'll
  probably want to swap in your own artwork before shipping to the App Store,
  but it's there so the app looks finished today.

Everything else from the spec — accurate backgrounding, guardrails against
double-taps/negative reps/duplicate history rows, crash recovery, streaks,
PRs, custom workouts, audio coaching, settings, data export/delete — is in.

---

## 1. Opening the project in Xcode

1. Unzip the download.
2. Open **`CindyTimer.xcodeproj`** (double-click it, or `File > Open` in
   Xcode and select the file).
3. Xcode will index the project for a moment — that's normal.

Requires **Xcode 16** or later (for Swift 5/SwiftUI features used here) and
targets **iOS 17+** (SwiftData requires iOS 17).

## 2. Selecting your iPhone

1. Plug your iPhone into your Mac with a cable, or make sure it's on the
   same Wi-Fi network as your Mac with Wireless Debugging enabled
   (`Settings > Privacy & Security > Developer Mode` must be turned on on
   the phone the first time).
2. In Xcode's toolbar, click the device picker (it might say "Any iOS
   Device" or show a simulator name) and choose your iPhone from the list.

## 3. Configuring signing (required to run on a real device)

1. Click **CindyTimer** (the blue project icon) at the top of the left
   sidebar.
2. Select the **CindyTimer** target, then the **Signing & Capabilities** tab.
3. Under **Team**, choose your Apple ID. If none is listed, click
   **Add an Account...** and sign in with any Apple ID (a free account works
   for running on your own device; it just can't publish to the App Store).
4. Xcode will auto-generate a bundle identifier signing certificate. If you
   see a red error about the bundle identifier being taken, change
   `PRODUCT_BUNDLE_IDENTIFIER` (currently `com.samirsarsia.CindyTimer`) to
   something more unique, like `com.samirsarsia.cindytimer.yourname`.
5. Repeat for the **CindyTimerTests** target if you plan to run the tests
   on-device (running tests in the Simulator doesn't require this).

## 4. Building and running

1. With your iPhone selected as the destination, press **⌘R** (or click the
   ▶️ Play button).
2. The first time, your iPhone may show an "Untrusted Developer" alert.
   Go to **Settings > General > VPN & Device Management** on the iPhone,
   tap your developer profile, and tap **Trust**.
3. Run again — the app should launch on your phone.

## 5. Running the unit tests

Press **⌘U**, or use **Product > Test**. Tests run fastest in the Simulator
(`Product > Destination > any iPhone Simulator`).

## 6. Creating an Archive (for TestFlight / App Store prep)

1. Select **Any iOS Device (arm64)** as the destination (not your specific
   phone, and not a Simulator) from the device picker.
2. **Product > Archive**.
3. When it finishes, the **Organizer** window opens showing your archive.
4. Click **Distribute App** to upload to App Store Connect (for TestFlight)
   or export an `.ipa` for ad-hoc distribution.

## 7. Before submitting to the App Store

- Replace the placeholder app icon in
  `CindyTimer/Assets.xcassets/AppIcon.appiconset` with final artwork.
- Double check `PRODUCT_BUNDLE_IDENTIFIER` and `MARKETING_VERSION` in the
  target's Build Settings.
- Fill in App Store Connect's privacy questionnaire — this app collects
  nothing, stores everything locally, and requests only local notification
  permission (optional, for reminders).

---

## Design decisions worth knowing about

- **Timer accuracy**: `TimerEngine` never does `time -= 1` on a repeating
  tick. It always recomputes `elapsedSeconds` from `Date()` minus a fixed
  start timestamp minus accumulated paused time. Backgrounding, locking, or
  a slow UI frame can never cause drift — the next read is always correct.
- **Single source of truth**: `WorkoutEngine` is the only thing allowed to
  change workout state. Views only call its methods and read its published
  properties — this is what makes "can a double-tap corrupt my rep count?"
  and "what if my phone died mid-workout?" answerable in one place.
- **Crash recovery**: after every rep tap, exercise completion, pause, or
  resume, a small JSON snapshot is written to disk. On next launch, if that
  snapshot exists, the engine rebuilds the exact in-progress workout from it
  rather than losing your progress.
- **No duplicate history rows**: every workout result carries a stable
  `sessionID`. Saving checks for that ID first, so even if the save path
  were somehow triggered twice, you won't get a duplicate history entry.
