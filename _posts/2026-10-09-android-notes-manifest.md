---
title: "Android Notes: AndroidManifest.xml — What It Is and What to Look For"
date: 2026-10-09
categories: [Mobile, Android, Notes]
tags: [android, jadx, manifest, permissions, recon, mobile]
---
 
## What is AndroidManifest.xml?
 
Every Android APK contains exactly one `AndroidManifest.xml`. It is the central configuration file of the app, the first thing you open in jadx-gui when you start analyzing an APK.
 
It defines:
 
- The app's **package name** (e.g. `fr.vinted`)
- All **Activities, Services, Broadcast Receivers, and Content Providers**
- The **permissions** the app requests from the OS and the user
- Any **custom permissions** the app declares
- Minimum and maximum supported Android SDK versions
---
 
## Permissions: The 3 Types That Matter
 
### 1. Dangerous permissions (user must accept at runtime)
 
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>
```
 
These require an explicit popup to the user on Android 6+. If the app requests more than it needs, that's worth noting.
 
### 2. Normal permissions (granted automatically)
 
```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.VIBRATE"/>
```
 
Granted silently at install time. No popup.
 
### 3. Custom permissions (app-defined)
 
```xml
<permission
    android:name="fr.vinted.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION"
    android:protectionLevel="signature"/>
```
 
The app defines its own permission. Only other apps signed with the **same certificate** can use it, protects internal components from being accessed by third-party apps.
 
---
 
## What to Look For in Bug Bounty
 
| Field | Why It Matters |
|---|---|
| `android:exported="true"` on Activities/Services | Component is accessible from other apps — possible attack surface |
| `android:debuggable="true"` | App is in debug mode — can be attached with a debugger |
| `android:allowBackup="true"` | App data can be extracted with `adb backup` without root |
| `android:networkSecurityConfig` | Check if it allows cleartext HTTP or trusts user-installed CAs |
| Deep links (`intent-filter` with `ACTION_VIEW`) | Potential open redirect, XSS via intent, or path traversal |
| Legacy storage permissions | `WRITE_EXTERNAL_STORAGE` → data accessible to other installed apps |
 
---
 
## Real Example: Vinted APK
 
Analyzing `base.apk` from the Vinted Android app in jadx-gui, the manifest reveals:
 
- Requests `ACCESS_FINE_LOCATION` and `ACCESS_COARSE_LOCATION`
- Uses `USE_BIOMETRIC` and `USE_FINGERPRINT` for authentication
- `READ_EXTERNAL_STORAGE` scoped to `maxSdkVersion="32"`, good practice, uses newer media APIs on Android 13+
- Custom permission `fr.vinted.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` with `protectionLevel="signature"`, protects internal broadcast receivers
- Multiple Google Ads and Firebase permissions declared

![vinted-manifest](/vintes-manifest.png)

Nothing critical on the surface, but the manifest is always step one of any Android security review.
 
---
 
## Vulnerable Example: DIVA APK
 
[DIVA](http://www.payatu.com/damn-insecure-and-vulnerable-app/) (Damn Insecure and Vulnerable App) is an intentionally vulnerable Android app used for learning. Its manifest is a textbook example of what **not** to do, and exactly what you look for in real targets.
 
```xml
<application
    android:debuggable="true"
    android:allowBackup="true"
    ...>
```
![diva1](/diva-1.png)
 
### 🔴 Critical findings at a glance
 
| Finding | Field | Impact |
|---|---|---|
| Debug mode enabled | `android:debuggable="true"` | Any app can attach a debugger via ADB and inspect memory, bypass logic |
| Backup enabled | `android:allowBackup="true"` | Full app data extracted with `adb backup -f diva.ab jakhar.aseem.diva` — no root needed |
| Old target SDK | `targetSdkVersion="23"` | Misses all security improvements from Android 7+ (cleartext blocked, scoped storage, etc.) |
| Legacy storage | `WRITE/READ_EXTERNAL_STORAGE` (no maxSdkVersion) | Files written to external storage are readable by any installed app |
| Exported Content Provider | `NotesProvider exported="true"` | Any app on the device can query, insert, update or delete the notes database — no permission required |
| Exported Activities with credentials | `APICredsActivity`, `APICreds2Activity` | Both have `intent-filter` with custom actions — any app can launch them and view API credentials |
| URI scheme handler | `InputValidation2URISchemeActivity` | Handles custom URI schemes — potential entry point for intent-based attacks |
 
### Exported Content Provider: high impact
 
```xml
<provider
    android:name="jakhar.aseem.diva.NotesProvider"
    android:enabled="true"
    android:exported="true"
    android:authorities="jakhar.aseem.diva.provider.notesprovider"/>
```

![diva2](/diva-2.png)
  
No `android:permission` attribute. Any app can do:
 
```bash
adb shell content query --uri content://jakhar.aseem.diva.provider.notesprovider/notes
```
 
And read all notes. On a real app this could be messages, tokens, or PII.
 
### Exposed credential Activities
 
```xml
<activity android:name="jakhar.aseem.diva.APICredsActivity">
    <intent-filter>
        <action android:name="jakhar.aseem.diva.action.VIEW_CREDS"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</activity>
```
![diva3](/diva-3.png)
  
Because this Activity has an `intent-filter` it is **implicitly exported**, any malicious app can fire an intent with action `jakhar.aseem.diva.action.VIEW_CREDS` and open the credentials screen directly.
 
```bash
adb shell am start -a jakhar.aseem.diva.action.VIEW_CREDS
```
 
### Summary
 
From the manifest alone — before looking at a single line of Java, DIVA is already confirmed vulnerable to:
 
- ADB debugging and memory inspection
- Backup-based data extraction
- Content provider data theft (no-permission required)
- Unauthorized Activity launch to view API credentials
- Potential URI scheme abuse via deep links
