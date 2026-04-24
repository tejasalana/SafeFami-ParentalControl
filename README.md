# Tortoise Security

A prototype Android parental control application that enables parents to monitor their children's digital activity and device usage in real time.

---

## Overview

Tortoise Security runs silently in the background on a child's device, collecting usage data and reporting it to the parent's dashboard. The child consents to monitoring during the initial onboarding flow; after setup no notifications or indicators are shown to the child.

---

## Features

### Child Device
- Silent background service — no launcher icon once registered as a child
- Continuous collection of screen time, app usage, and device status
- Real-time location reporting via GPS
- Screenshot capture on parent request (image uploaded to Firebase Storage then deleted locally)
- Battery-optimised background operation (WorkManager + Doze-aware scheduling)
- Device admin registration to survive permission denial attempts

### Parent Dashboard
- Secure login via parent ID (created in backend)
- View all registered children
- Per-child analytics: screen time, per-app usage, usage trends
- Live activity panel per child:
  - Current GPS location on Google Maps
  - Device status (idle / locked / charging)
  - Foreground app name
  - On-demand screenshot

---

## User Flows

### Child Registration
```
Launch app → "Login as Child"
  → Enter child name + parent ID
  → Validate against Firebase (parent ID must exist)
  → Grant required permissions screen
      (Location, Usage Stats, Accessibility, Device Admin, etc.)
  → Registration complete — app icon hidden from launcher
```

### Parent Login
```
Launch app → "Login as Parent"
  → Enter parent ID
  → Navigate to dashboard
  → Select child → view analytics
  → Tap "Live Activity" → real-time data panel
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose + Material 3 |
| Backend / Auth | Firebase Authentication |
| Database | Firebase Firestore |
| File Storage | Firebase Storage (screenshots) |
| Maps | Google Maps SDK for Android |
| Background Work | WorkManager |
| Min SDK | API 24 (Android 7.0) |
| Target / Compile SDK | API 36 |

---

## Required Permissions

| Permission | Purpose |
|---|---|
| `ACCESS_FINE_LOCATION` | GPS tracking |
| `PACKAGE_USAGE_STATS` | App usage data |
| `BIND_ACCESSIBILITY_SERVICE` | Foreground app detection |
| `FOREGROUND_SERVICE` | Background service |
| `RECEIVE_BOOT_COMPLETED` | Auto-start on reboot |
| `DEVICE_ADMIN` | Prevent uninstall / re-grant permissions |
| `INTERNET` | Firebase sync |
| `MEDIA_PROJECTION` (runtime) | Screenshot capture |

> All permissions are requested during the child's one-time onboarding. The child explicitly consents before installation is finalised.

---

## Project Structure

```
app/src/main/java/com/skylinestudio/security/
├── MainActivity.kt
├── ui/
│   └── theme/
│       ├── Color.kt
│       ├── Theme.kt
│       └── Type.kt
```

> The project is in early prototype stage. Feature modules (auth, dashboard, tracking service, live activity) are to be added.

---

## Firebase Setup

1. Create a Firebase project at [console.firebase.google.com](https://console.firebase.google.com).
2. Register the Android app with package name `com.skylinestudio.security`.
3. Download `google-services.json` and place it in the `app/` directory.
4. Enable the following Firebase services:
   - Authentication (anonymous or email/password)
   - Firestore Database
   - Storage

---

## Getting Started

### Prerequisites
- Android Studio Hedgehog or later
- JDK 11
- A device or emulator running Android 7.0+ (API 24)

### Build & Run
```bash
git clone <repo-url>
cd TortoiseSecurity
# Add app/google-services.json (see Firebase Setup above)
./gradlew assembleDebug
```

---

## Privacy & Legal Notice

This application is designed strictly for monitoring minor children with their prior consent, as obtained during the onboarding flow. Deploying this application to monitor individuals without their knowledge and consent may violate local privacy and wiretapping laws. The developers assume no liability for misuse.

---

## Status

Prototype — Firebase integration and feature modules are pending implementation.
