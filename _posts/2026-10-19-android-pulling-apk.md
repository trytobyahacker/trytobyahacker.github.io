---
title: "Android Notes: Pulling an APK from a Device to Your PC/Windows"
date: 2026-10-19
categories: [Mobile, Android]
tags: [android, apk, adb, bugbounty, mobile, security, mobile-security]
---
 
Look, to make it easier and actually understand it:
 
**`adb`** stands for: Android Debug Bridge. It's the tool that connects your
PC to the Android device or emulator and lets you send it commands.
 
## List installed apps
 
If you run `adb shell pm list packages` → you're listing the apps installed
on the device.

![Text widget form](/assets/img/posts/emulaorr.png)

```bash
adb shell pm list packages
```

![Text widget form](/assets/img/posts/packages.png)
 
> `shell` = run a command inside Android.
> `pm` = package manager (Android's package manager).
> `list packages` = show me the list of apps.
 
You'll find the app names, for example: `jakhar.aseem.diva`
 
## Find its path
 
Now to find its full path:
 
```bash
adb shell pm path jakhar.aseem.diva
```
![Text widget form](/assets/img/posts/path.png)

> `pm path <package>` = tell me which folder the app's APK is physically
> stored in inside Android.
 
And you'll get:
 
```
package:/data/app/~~71dlfvIPBtnK-uJQX_4vjQ==/jakhar.aseem.diva-J50fK23lZVkj3-P7vZz91w==/base.apk
```
 
## Pull it to Windows
 
Now that we have its path, to pull it to Windows and analyze it with
jadx-ui it would be:
 
```bash
adb pull /data/app/~~71dlfvIPBtnK-uJQX_4vjQ==/jakhar.aseem.diva-J50fK23lZVkj3-P7vZz91w==/base.apk "C:\Users\MilLordd\Downloads\diva.apk"
```
 
> `adb pull <path_in_android> <path_on_your_pc>` = copy that file from
> Android to your PC. The `diva.apk` at the end is the name we'll give it
> (it can be whatever you want).
 
## What if 3 or more files show up?
 
If listing the path returns 3 or more `.apk` files, to pull them all to
Windows at once (instead of one by one), you just copy the **folder**.
 
Example. If we run:
 
```bash
adb shell pm path io.getdelta.android
```
 
You'll get these 4 files:
 
```
1. package:/data/app/~~n8PmKeJJMUWOIYFvLQFBgQ==/io.getdelta.android-6APNP-e0E_eyRMyxL_7sAA==/base.apk
2. package:/data/app/~~n8PmKeJJMUWOIYFvLQFBgQ==/io.getdelta.android-6APNP-e0E_eyRMyxL_7sAA==/split_config.en.apk
3. package:/data/app/~~n8PmKeJJMUWOIYFvLQFBgQ==/io.getdelta.android-6APNP-e0E_eyRMyxL_7sAA==/split_config.tvdpi.apk
4. package:/data/app/~~n8PmKeJJMUWOIYFvLQFBgQ==/io.getdelta.android-6APNP-e0E_eyRMyxL_7sAA==/split_config.x86_64.apk
```
![Text widget form](/assets/img/posts/4-archives.png)
 
They're all in the **same folder**. To pull them all at once, copy the path
only up to the folder, that is, **without the `/base.apk` at the end**
(up to `7sAA==`):
 
```bash
adb pull /data/app/~~n8PmKeJJMUWOIYFvLQFBgQ==/io.getdelta.android-6APNP-e0E_eyRMyxL_7sAA== "C:\Users\MiLordd\Downloads\delta_apks"
```
 
> By dropping the file name and leaving just the folder, `adb pull` grabs
> **everything inside** it in one go.
 
You can name it whatever you want `delta_apks`, `delta`, or
`delta_files` and then open it with jadx-ui.
 
> For analyzing the app, **`base.apk`** alone is enough. The splits
> (`split_config.*`) only carry language, screen-density and architecture
> resources. You need all of them only if you want to reinstall the app
> with `adb install-multiple`.
 
