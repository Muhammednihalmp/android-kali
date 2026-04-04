# OnTime — Kali Linux Boot Animation Clock

```
  ██╗  ██╗ █████╗ ██╗     ██╗
  ██║ ██╔╝██╔══██╗██║     ██║
  █████╔╝ ███████║██║     ██║
  ██╔═██╗ ██╔══██║██║     ██║
  ██║  ██╗██║  ██║███████╗██║
  ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝╚═╝
  ─────────────────────────────
        L I N U X  O S
  ─────────────────────────────
  [ The quieter you become, the more you are able to hear ]
```

> A Kali Linux–themed Android clock app with a full boot animation sequence on startup, live digital clock, CRT scanline effects, and a hacker-green terminal aesthetic.

---

## Table of Contents

- [Screenshots](#screenshots)
- [Features](#features)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Setup & Installation](#setup--installation)
- [Building the APK](#building-the-apk)
- [File Reference](#file-reference)
- [Permissions](#permissions)
- [Customization](#customization)
- [License](#license)

---

## Screenshots

```
┌─────────────────────┐     ┌─────────────────────┐
│  BOOT SCREEN        │     │  CLOCK SCREEN        │
│                     │     │                      │
│  ██╗  ██╗  █████╗   │     │ ● root@kali — OnTime │
│  ██║ ██╔╝ ██╔══██╗  │     │ ─────────────────── │
│  █████╔╝  ███████║  │     │      SUNDAY          │
│  ██╔═██╗  ██╔══██║  │  →  │                      │
│  ██║  ██╗ ██║  ██║  │     │   14:35:09█          │
│                     │     │                      │
│ [0.000000] Booting  │     │    2025-04-05        │
│ [0.125000] cgroup.. │     │ ─────────────────── │
│ [1.200000] AppArmor │     │ root@kali:~$ time    │
│ ████████░░░░  68%   │     │ CPU 0.1% MEM 128MB   │
└─────────────────────┘     └─────────────────────┘
```

---

## Features

- **Kali Boot Animation** — 22 authentic kernel-style log messages scroll on startup with real timestamps
- **Animated Progress Bar** — green progress bar fills from 0% to 100% during boot
- **Smooth Screen Transition** — fade-out boot screen, fade-in clock screen
- **Live Digital Clock** — updates every second in `HH:MM:SS` format
- **Terminal Aesthetic** — monospace font, hacker green (`#1ED760`), neon glow shadow on time digits
- **CRT Scanline Effect** — sweeping scanline animation loops across the clock screen
- **Blinking Cursor** — cursor blinks beside the time display
- **Fullscreen Immersive** — hides status bar and navigation bar
- **Screen Always On** — keeps display alive while clock is showing
- **Lightweight** — no third-party dependencies beyond AndroidX

---

## Project Structure

```
OnTime/
├── app/
│   ├── src/
│   │   └── main/
│   │       ├── java/
│   │       │   └── com/kali/ontime/
│   │       │       └── MainActivity.java          ← Main activity (boot + clock logic)
│   │       ├── res/
│   │       │   ├── layout/
│   │       │   │   └── activity_main.xml          ← Two-screen XML layout
│   │       │   ├── values/
│   │       │   │   ├── strings.xml                ← App name & strings
│   │       │   │   └── themes.xml                 ← Fullscreen dark Kali theme
│   │       │   └── drawable/
│   │       │       └── dot_green.xml              ← Green status dot shape
│   │       └── AndroidManifest.xml                ← App manifest & permissions
│   └── build.gradle                               ← App-level Gradle config
└── README.md
```

---

## Requirements

| Tool | Version |
|------|---------|
| Android Studio | Hedgehog (2023.1) or newer |
| JDK | 8 or higher |
| Android Gradle Plugin | 8.x |
| `compileSdk` | 34 (Android 14) |
| `minSdk` | 21 (Android 5.0 Lollipop) |
| `targetSdk` | 34 (Android 14) |

---

## Setup & Installation

### 1. Clone or download the project

```bash
git clone https://github.com/yourname/ontime-kali-clock.git
cd ontime-kali-clock
```

### 2. Open in Android Studio

```
File → Open → select the project root folder
```

Wait for Gradle sync to complete.

### 3. Place files in the correct directories

| File | Destination |
|------|------------|
| `MainActivity.java` | `app/src/main/java/com/kali/ontime/` |
| `activity_main.xml` | `app/src/main/res/layout/` |
| `AndroidManifest.xml` | `app/src/main/` |
| `themes.xml` | `app/src/main/res/values/` |
| `strings.xml` | `app/src/main/res/values/` |
| `dot_green.xml` | `app/src/main/res/drawable/` |
| `build.gradle` | `app/` |

### 4. Sync Gradle

```
File → Sync Project with Gradle Files
```

---

## Building the APK

### Debug APK (for testing)

```bash
./gradlew assembleDebug
```

Output location:
```
app/build/outputs/apk/debug/app-debug.apk
```

### Release APK (for distribution)

1. Generate a signing keystore:

```bash
keytool -genkey -v -keystore ontime-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias ontime
```

2. Add signing config to `app/build.gradle`:

```groovy
android {
    signingConfigs {
        release {
            storeFile     file("ontime-release.jks")
            storePassword "your_store_password"
            keyAlias      "ontime"
            keyPassword   "your_key_password"
        }
    }
    buildTypes {
        release {
            signingConfig signingConfigs.release
        }
    }
}
```

3. Build:

```bash
./gradlew assembleRelease
```

Output:
```
app/build/outputs/apk/release/app-release.apk
```

### Install directly to a connected device

```bash
./gradlew installDebug
```

---

## File Reference

### `MainActivity.java`

Controls the full app lifecycle across two screens.

| Method | Description |
|--------|-------------|
| `onCreate()` | Sets fullscreen mode, inflates layout, kicks off boot |
| `initViews()` | Binds all XML view references |
| `startBootSequence()` | Fades in logo and tagline |
| `startBootLogs()` | Posts 22 log lines with a Handler loop |
| `transitionToClockScreen()` | Cross-fades boot → clock screen |
| `startClock()` | Starts a 1-second repeating Handler for time updates |
| `updateTime()` | Updates time, date, day, terminal prompt TextViews |
| `startClockAnimations()` | Starts scanline sweep, cursor blink, time flicker |

---

### `activity_main.xml`

Two-screen `FrameLayout`:

| View ID | Type | Purpose |
|---------|------|---------|
| `bootScreen` | `LinearLayout` | Container for entire boot UI |
| `bootLogo` | `TextView` | Kali ASCII dragon art |
| `bootTagline` | `TextView` | Famous Kali quote |
| `bootLog` | `TextView` | Scrolling kernel log output |
| `bootProgress` | `ProgressBar` | Horizontal green progress bar |
| `bootPercentage` | `TextView` | Percentage counter |
| `clockScreen` | `FrameLayout` | Container for clock UI |
| `timeDisplay` | `TextView` | `HH:MM:SS` live time (68sp, glowing) |
| `dateDisplay` | `TextView` | `YYYY-MM-DD` date |
| `dayDisplay` | `TextView` | Day of week in caps |
| `terminalPrompt` | `TextView` | `root@kali:~$ time --sync` |
| `scanline` | `View` | CRT scanline (animated via code) |
| `cursorBlink` | `View` | Blinking cursor bar |

---

### `AndroidManifest.xml`

```xml
package:         com.kali.ontime
minSdk:          21
targetSdk:       34
orientation:     portrait
theme:           Theme.OnTime.Fullscreen
keepScreenOn:    true
```

---

### `themes.xml`

| Style | Parent | Purpose |
|-------|--------|---------|
| `Theme.OnTime` | `MaterialComponents.DayNight.NoActionBar` | Base dark Kali theme |
| `Theme.OnTime.Fullscreen` | `Theme.OnTime` | Fullscreen overlay for MainActivity |

Primary color: `#1ED760` (Kali green)
Background: `#000000` (pure black)

---

## Permissions

| Permission | Reason |
|-----------|--------|
| `INTERNET` | Future NTP time sync support |
| `WAKE_LOCK` | Keeps screen on while clock displays |
| `RECEIVE_BOOT_COMPLETED` | Future: auto-start on device boot |
| `VIBRATE` | Future: alarm vibration support |

---

## Customization

### Change the boot log messages

In `MainActivity.java`, edit the `bootMessages[]` array:

```java
private final String[] bootMessages = {
    "[    0.000000] Booting Kali Linux...",
    // add or remove lines here
};
```

### Change the clock color

In `activity_main.xml`, find `timeDisplay` and update `android:textColor`:

```xml
android:textColor="#1ED760"   <!-- Kali green (default) -->
android:textColor="#FF0000"   <!-- Red -->
android:textColor="#00BFFF"   <!-- Cyan -->
```

Also update `android:shadowColor` to match.

### Change boot speed

In `MainActivity.java`, inside `startBootLogs()`:

```java
long delay = (bootStep < 5) ? 120 : (bootStep < 15) ? 160 : 200;
// Lower values = faster boot animation
```

### Change scanline speed

In `startClockAnimations()`:

```java
scanAnim.setDuration(3500); // milliseconds per sweep — lower = faster
```

### Disable fullscreen mode

In `onCreate()`, remove or comment out:

```java
getWindow().getDecorView().setSystemUiVisibility(
    View.SYSTEM_UI_FLAG_HIDE_NAVIGATION | ...
);
```

---

## Dependencies

```groovy
implementation 'androidx.appcompat:appcompat:1.7.0'
implementation 'com.google.android.material:material:1.12.0'
implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
```

No external libraries. All animations use Android's built-in `ObjectAnimator` and `Handler`.

---

## License

```
MIT License

Copyright (c) 2025 OnTime / Kali Clock Project

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
```

---

> **Kali Linux** is a trademark of Offensive Security. This project is an independent fan creation and is not affiliated with or endorsed by Offensive Security.
