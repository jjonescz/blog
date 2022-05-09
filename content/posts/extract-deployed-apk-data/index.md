---
title: Extracting data from deployed APK
slug: extract-deployed-apk-data
date: 2021-01-13T21:31:09Z
summary: How to extract Android application data even if it is not deployed in
  Debug configuration.
tags:
  - Android
  - APK
categories:
  - Programming
cover:
  image: cover.png
  alt: Extract APK
  relative: true
---

I have recently struggled with getting data out of my own Android application
that I have installed from Play Store and wanted to backup my data before
deploying new development version. These so-called [app specific
data](https://developer.android.com/training/data-storage/app-specific) are
protected so that other apps cannot see them and what's more, it is not
straightforward for users to see them, either.

Note that any app [can disable](https://android.stackexchange.com/a/172623) data
extraction described in this post by setting
[`android:allowBackup`](https://developer.android.com/guide/topics/manifest/application-element#allowbackup)
to `false`.

## Extracting data

I assume you have `adb` installed and your Android device connected with
developer mode enabled on it. In following commands, replace `com.example.app`
with your chosen [application
ID](https://developer.android.com/studio/build/application-id).

To backup data, execute:

```bash
adb backup -f backup.ab -apk com.example.app
```

Note that `adb` reports that this command is
[deprecated](https://android.stackexchange.com/a/231237), but you can ignore it:

{{< samp >}}
WARNING: adb backup is deprecated and may be removed in a future release
{{< /samp >}}

Then, a prompt on your Android device will be shown:
{{< figure src="android-backup-screenshot.jpg" class="screenshot"
    link="android-backup-screenshot.jpg" rel="true"
    alt="Android Backup Confirmation Screenshot" >}}

Click "Backup My Data" to complete the backup.

The created `backup.ab` file is almost an archive but without header. You can
extract data from it using [one-line
command](https://stackoverflow.com/a/46500482) (if you are on Windows, use your
WSL `bash`):

```bash
( printf "\x1f\x8b\x08\x00\x00\x00\x00\x00" ; tail -c +25 backup.ab ) | tar xfvz -
```

Data of your app will be extracted into folder `apps/com.example.app/`.

## Restoring data

To restore the (possibly modified) data into a development version of my app, I
use Android Studio's [Device File
Explorer](https://developer.android.com/studio/debug/device-file-explorer) and
its "Upload" option.

If you need to restore the created backup (you don't even need to extract it in
the first place), you can simply use command inverse to `adb backup`:

```bash
adb restore backup.ab
```
