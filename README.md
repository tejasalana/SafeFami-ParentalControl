# Tortoise Security

A prototype Android parental control application that enables parents to monitor their children's digital activity and device usage on demand.

---

## Overview

Tortoise Security runs silently in the background on a child's device. The parent dashboard provides on-demand monitoring actions and real-time app controls — the parent taps a button or toggles a setting, a command is written to Firestore, and the child's device responds or reacts instantly. No continuous polling, no screenshots, no dismissible system dialogs.

The child consents to monitoring during the initial one-time onboarding flow. After setup the app has no launcher icon and runs as a silent background service.

---

## Features

### Parent Dashboard
- Lists all children registered under the parent's ID
- Tap any child to open their **Child Analytics** screen

### Child Analytics Screen
Mirrors Android's built-in Digital Wellbeing view, powered by `UsageStatsManager`:

| Section | Data shown | Time range |
|---|---|---|
| Screen time | Total daily screen-on time | Today / last 7 days |
| App usage | Per-app foreground time, sorted by most used | Today / last 7 days |
| Unlocks | Number of times device was unlocked | Today / last 7 days |
| Notifications received | Total notification count | Today / last 7 days |

From this screen the parent can access three sub-sections:

---

### Blocked Apps

Parent toggles any installed app on/off from the child's app list. The change is written to Firestore and applied on the child's device **immediately** via a real-time listener — no delay, no restart required.

**How blocking works on child device:**
- `AccessibilityService` detects every foreground app change (`TYPE_WINDOW_STATE_CHANGED`)
- Package name is checked against the locally cached blocklist (synced from Firestore)
- If blocked → a full-screen "This app is blocked by your parent" Activity is launched over it instantly
- Child taps Back → returns to home screen
- The blocklist is also re-applied on every device reboot since the Accessibility Service auto-restarts

**Firestore structure:**
```
blockedApps/{childId}
  └── packages: ["com.android.vending", "com.gambling.app", ...]
```

---

### App Install & Uninstall History

From the moment TortoiseSecurity is installed on the child's device, every app install and uninstall is tracked as an event and synced to the parent in real time.

**On first launch — baseline scan:**
All apps currently installed on the device are recorded with their original install date sourced from `PackageInfo.firstInstallTime`.

**Ongoing tracking — `BroadcastReceiver` (manifest-registered):**
Fires even when the app is in the background or killed:

| Event | Trigger |
|---|---|
| App installed | `Intent.ACTION_PACKAGE_ADDED` |
| App uninstalled | `Intent.ACTION_PACKAGE_FULLY_REMOVED` |

Installing and uninstalling the same app multiple times each produce separate event entries — nothing is overwritten.

**Each event records:**

| Field | Source | Example |
|---|---|---|
| App name | `PackageManager.getApplicationLabel()` | "WhatsApp" |
| Package name | Intent data | `com.whatsapp` |
| Event type | Intent action | `INSTALLED` / `UNINSTALLED` |
| Timestamp | System time at event (IST, 12-hour format) | `26 Apr 2026, 2:32:00 PM` |
| Install source | `PackageManager.getInstallSourceInfo()` (API 30+) / `getInstallerPackageName()` (API 24–29) | See table below |
| App version | `PackageInfo.versionName` | `2.24.7.79` |

**Install source mapping:**

| Installer package | Shown as |
|---|---|
| `com.android.vending` | Google Play Store |
| `com.amazon.venezia` | Amazon Appstore |
| `com.sec.android.app.samsungapps` | Samsung Galaxy Store |
| `com.huawei.appmarket` | Huawei AppGallery |
| `com.android.packageinstaller` | APK File (Manual Install) |
| `null` / empty | Unknown / ADB Sideload |

**Firestore structure:**
```
appHistory/{childId}/events/{eventId}
  ├── appName:       "Stake - Sports Betting & Casino"
  ├── packageName:   "com.stake.app"
  ├── eventType:     "INSTALLED"
  ├── timestamp:     2026-04-26T14:32:00Z
  ├── installSource: "APK File (Manual Install)"
  └── version:       "1.4.2"
```

Parent sees a chronological feed of all install/uninstall events per child, updated in real time via a Firestore listener.

---

### Live Status Screen
Four on-demand action buttons — parent taps, child device responds in real time:

| Button | What it does | Child notified? |
|---|---|---|
| **Location** | Fetches current GPS coordinates and displays on a map | Yes — Android shows a system location-access notification (unavoidable on Android 10+) |
| **Device Status** | Returns battery %, charging state, screen on/off, Wi-Fi SSID, Bluetooth state, mobile data state | No |
| **Active Apps** | Lists all apps used today with last-used time, sorted by recency, including currently running background apps | No |
| **Live App + Context** | Returns the current foreground app name, all visible text on screen, and recent notification content. Returns app name only if the app blocks screen reading (e.g. banking apps, Netflix) | No |

---

### Child Device (always-on background services)

- Silent background service — no launcher icon once registered as a child
- **Accessibility Service** — persists across reboots; foreground app detection, visible screen text, and instant app blocking
- **Notification Listener Service** — intercepts incoming notifications from all apps to support the Live App + Context feature
- **Package Change Receiver** — manifest-registered broadcast receiver; tracks every app install, uninstall, and update in real time
- **Foreground Service** — keeps the above alive, listens for parent commands and blocklist changes via Firestore, syncs results back
- Device admin registration to survive permission denial attempts
- Battery-optimised background operation (Doze-aware)

---

## Command-Response Architecture

All four Live Status actions follow the same pattern:

```
Parent taps button
  └── Writes command to Firestore: commands/{childId}/pending
          │
          ▼  (Firestore real-time listener on child device)
Child's ForegroundService receives command
  └── Executes the action (location fix, status read, etc.)
  └── Uploads result to Firestore: results/{childId}/{feature}
          │
          ▼  (Firestore real-time listener on parent device)
Parent dashboard updates instantly
```

App blocking and app install events use the same Firestore real-time channel — changes propagate to the child device and parent dashboard without polling.

No FCM required — Firestore real-time listeners are persistent and survive app restarts.

---

## Live App + Context — How it works

Three layers are combined whenever the parent requests this feature:

1. **Foreground app name** — always available via `AccessibilityService.TYPE_WINDOW_STATE_CHANGED`, including for FLAG_SECURE apps (banking, Netflix, etc.)
2. **Visible screen text** — `rootInActiveWindow` node traversal via Accessibility Service. Returns empty if the active app sets `FLAG_SECURE`
3. **Recent notifications** — last N notifications from `NotificationListenerService`, including sender and message body

Example result for WhatsApp:
```json
{
  "currentApp": "com.whatsapp",
  "appLabel": "WhatsApp",
  "screenTextAvailable": true,
  "visibleText": ["John: are you coming today", "You: yeah 5 mins"],
  "recentNotifications": [{ "from": "Mom", "text": "Come home by 9" }]
}
```

Example result for Netflix (FLAG_SECURE):
```json
{
  "currentApp": "com.netflix.mediaclient",
  "appLabel": "Netflix",
  "screenTextAvailable": false,
  "visibleText": [],
  "flagSecureBlocked": true
}
```

---

## User Flows

### Child Onboarding
```
Launch app → "Set up as Child"
  → Enter child name + parent ID
  → Validate against Firebase (parent ID must exist)
  → Baseline app scan — all currently installed apps recorded with install dates
  → Permission grant screens (one per permission):
      1. Location (fine + background)
      2. Usage Access  →  deep-link to Settings → Special App Access
      3. Accessibility Service  →  deep-link to Settings → Accessibility
      4. Notification Access  →  deep-link to Settings → Notification Access
      5. Device Admin
  → Registration complete — launcher icon hidden
  → PackageChangeReceiver now active — all future installs/uninstalls tracked
```

### Parent Login
```
Launch app → "Login as Parent"
  → Enter parent ID
  → Dashboard — lists all children registered under this parent ID
  → Tap a child → Child Analytics screen
      Shows screen time, per-app usage, unlocks, notifications
      for Today and Last 7 Days (mirrors Android Digital Wellbeing)
      ├── Blocked Apps  →  toggle any app on/off → applies to child instantly
      ├── App History   →  chronological install/uninstall event feed
      └── Get Live Status → Live Status screen
              [Location]  [Device Status]  [Active Apps]  [Live App + Context]
  → Tap any live status button → result appears in real time via Firestore
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Kotlin |
| UI | Jetpack Compose + Material 3 |
| Backend / Auth | Firebase Authentication |
| Database + Commands | Firebase Firestore (real-time listeners) |
| Maps | Google Maps SDK for Android |
| Screen monitoring | Android Accessibility Service |
| Notification monitoring | NotificationListenerService |
| App usage data | UsageStatsManager |
| App install tracking | PackageManager + BroadcastReceiver |
| Location | FusedLocationProviderClient (Google Play Services) |
| Background execution | ForegroundService |
| Min SDK | API 24 (Android 7.0) |
| Target / Compile SDK | API 36 |

---

## Required Permissions

| Permission | How granted | Purpose |
|---|---|---|
| `ACCESS_FINE_LOCATION` | Runtime prompt | GPS coordinates |
| `ACCESS_BACKGROUND_LOCATION` | Runtime prompt (separate, Android 10+) | Location when app is in background |
| `PACKAGE_USAGE_STATS` | Settings → Special App Access | Screen time, per-app usage, active apps list |
| `BIND_ACCESSIBILITY_SERVICE` | Settings → Accessibility | Foreground app detection, screen text, app blocking |
| `BIND_NOTIFICATION_LISTENER_SERVICE` | Settings → Notification Access | Incoming notification content |
| `QUERY_ALL_PACKAGES` | Manifest only (Android 11+) | Read names and details of all installed apps |
| `FOREGROUND_SERVICE` | Manifest only | Keep background service alive |
| `RECEIVE_BOOT_COMPLETED` | Manifest only | Auto-start services on reboot |
| `BIND_DEVICE_ADMIN` | Device admin prompt | Prevent uninstall / permission revocation |
| `ACCESS_WIFI_STATE` | Manifest only | Wi-Fi status for device status feature |
| `ACCESS_NETWORK_STATE` | Manifest only | Mobile data status |
| `BLUETOOTH_CONNECT` | Runtime prompt (Android 12+) | Bluetooth status |
| `INTERNET` | Manifest only | Firebase sync |

> All user-facing permissions are requested during the child's one-time onboarding, one screen per permission with a plain-language explanation.

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

> The project is in early prototype stage. Feature modules (auth, onboarding, Firestore command bus, Accessibility Service, Notification Listener, Package Change Receiver, parent dashboard) are to be added.

---

## Firebase Setup

1. Create a Firebase project at [console.firebase.google.com](https://console.firebase.google.com).
2. Register the Android app with package name `com.skylinestudio.security`.
3. Download `google-services.json` and place it in the `app/` directory.
4. Enable the following Firebase services:
   - Authentication (email/password or anonymous)
   - Firestore Database

---

## Privacy & Legal Notice

This application is designed strictly for monitoring minor children with their prior consent, as obtained during the onboarding flow. Deploying this application to monitor individuals without their knowledge and consent may violate local privacy and wiretapping laws. The developers assume no liability for misuse.

---

## Status

Prototype — Firebase integration and feature modules are pending implementation.