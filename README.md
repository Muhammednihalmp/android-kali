# OnTime — Kali Linux Boot Animation Clock

```
  ██╗  ██╗ █████╗ ██╗     ██╗
  ██║ ██╔╝██╔══██╗██║     ██║
  █████╔╝ ███████║██║     ██║
  ██╔═██╗ ██╔══██║██║     ██║
  ██║  ██╗██║  ██║███████╗██║
  ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝╚═╝
  | The quieter you become, the more you hear |
```

---

## Files Generated

| File | Path |
|---|---|
| `MainActivity.java` | `src/main/java/com/kali/ontime/` |
| `activity_main.xml` | `src/main/res/layout/` |
| `AndroidManifest.xml` | `src/main/` |
| `themes.xml` | `src/main/res/values/` |
| `dot_green.xml` | `src/main/res/drawable/` |
| `build.gradle` | `app/` |

---

## Boot Sequence Flow

```
onCreate()
  └─ initViews()
  └─ startBootSequence()
       ├─ Logo fade-in  (800ms)
       ├─ Tagline fade  (600ms)
       └─ startBootLogs()
            ├─ 22 kernel log lines
            ├─ Progress 0 → 100%
            └─ transitionToClockScreen()
                 ├─ bootScreen fade OUT
                 ├─ clockScreen fade IN
                 └─ startClock() + animations
```

---

## Permissions & Config

**Permissions:**

```
INTERNET    WAKE_LOCK    VIBRATE    RECEIVE_BOOT_COMPLETED
```

**Config:**

```
minSdk       21  (Android 5.0)
targetSdk    34  (Android 14)
orientation  portrait · fullscreen
theme        NoActionBar · black bg
```

---

## Clock Animations

```
scanline   translationY   -1800→1800px   3.5s   ∞
cursor     alpha           1→0           0.5s   ∞
timeflick  alpha           0→1→0.7→1     400ms  once
progress   ofInt           0→100         boot phase
```

---

## Live Preview — Boot Screen

```
┌──────────────────────────┐
│                          │
│   ██╗  ██╗  █████╗  ██╗  │
│   ██║ ██╔╝ ██╔══██╗ ██║  │
│   █████╔╝  ███████║ ██║  │
│   ██╔═██╗  ██╔══██║ ██║  │
│   ██║  ██╗ ██║  ██║ ██║  │
│   ╚═╝  ╚═╝ ╚═╝  ╚═╝ ╚═╝  │
│                          │
│  | The quieter you       │
│    become, the more      │
│    you hear |            │
│                          │
│  [ 0.000000] Booting     │
│    Kali Linux...         │
│  [ 0.000001] BIOS-       │
│    provided physical     │
│    RAM map:              │
│  [ 0.125000] Init        │
│    cgroup subsys...      │
│  [ 0.250000] Starting    │
│    kernel cryptography   │
│  [ 0.375000] Loading     │
│    network drivers...    │
│  [ 0.500000] eth0: link  │
│    up 1000Mbps full-dup  │
│  [ 0.650000] Starting    │
│    system logger: syslog │
│  [ 0.800000] Mounting    │
│    virtual filesystem... │
│  [ 1.000000] Starting    │
│    udev daemon...        │
│  [ 1.200000] Loading     │
│    security: AppArmor    │
│  [ 1.400000] Starting    │
│    OpenSSH server daemon │
│  [ 1.600000] Init        │
│    Metasploit Framework  │
│  [ 1.800000] Loading     │
│    Nmap network scanner  │
│  [ 2.000000] Starting    │
│    Wireshark capture svc │
│  [ 2.200000] Configuring │
│    iptables firewall...  │
│  [ 2.400000] Loading     │
│    exploit database...   │
│  [ 2.600000] Mounting    │
│    encrypted volumes...  │
│  [ 2.800000] Starting    │
│    anonymization service │
│                          │
│  ████████████████░░  82% │
└──────────────────────────┘
        Tap to restart boot
        [ ↺ Restart boot  ]
```

---

## Live Preview — Clock Screen

```
┌──────────────────────────┐
│ ●  root@kali — OnTime v1 │
│ ──────────────────────── │
│                          │
│         SUNDAY           │
│                          │
│      14:35:09█           │
│                          │
│       2025-04-05         │
│ ──────────────────────── │
│  root@kali:~$ time --sync│
│                          │
│  CPU 0.1%  MEM 128MB     │
│  UPTIME 99.9%            │
└──────────────────────────┘
```

---

## Project Structure

```
OnTime/
├── app/
│   ├── src/
│   │   └── main/
│   │       ├── java/
│   │       │   └── com/kali/ontime/
│   │       │       └── MainActivity.java
│   │       ├── res/
│   │       │   ├── layout/
│   │       │   │   └── activity_main.xml
│   │       │   ├── values/
│   │       │   │   ├── strings.xml
│   │       │   │   └── themes.xml
│   │       │   └── drawable/
│   │       │       └── dot_green.xml
│   │       └── AndroidManifest.xml
│   └── build.gradle
└── README.md
```

---

## Requirements

| Tool | Version |
|---|---|
| Android Studio | Hedgehog 2023.1 or newer |
| JDK | 8 or higher |
| compileSdk | 34 (Android 14) |
| minSdk | 21 (Android 5.0) |
| targetSdk | 34 (Android 14) |

---

## Build Commands

```bash
# Debug APK
./gradlew assembleDebug

# Release APK
./gradlew assembleRelease

# Install directly to connected device
./gradlew installDebug
```

**Debug output path:**

```
app/build/outputs/apk/debug/app-debug.apk
```

---

## Customization

**Change boot log lines** — edit `bootMessages[]` in `MainActivity.java`:

```java
private final String[] bootMessages = {
    "[    0.000000] Booting Kali Linux...",
    // add or remove lines here
};
```

**Change clock color** — edit `activity_main.xml`:

```xml
android:textColor="#1ED760"   <!-- Kali green (default) -->
android:textColor="#FF0000"   <!-- Red         -->
android:textColor="#00BFFF"   <!-- Cyan        -->
```

**Change boot speed** — edit delay in `startBootLogs()`:

```java
long delay = (bootStep < 5) ? 120 : (bootStep < 15) ? 160 : 200;
// lower values = faster boot animation
```

**Change scanline speed** — edit `startClockAnimations()`:

```java
scanAnim.setDuration(3500); // ms per full sweep — lower = faster
```

---

## Dependencies

```groovy
implementation 'androidx.appcompat:appcompat:1.7.0'
implementation 'com.google.android.material:material:1.12.0'
implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
```

No external libraries — all animations use Android's built-in `ObjectAnimator` and `Handler`.

---

## License

```
MIT License — Copyright (c) 2025 OnTime / Kali Clock Project

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND.
```

> **Kali Linux** is a trademark of Offensive Security.
> This project is an independent fan creation and is not affiliated
> with or endorsed by Offensive Security.
