# 01 — Exported debug receivers: a backdoor you add yourself

*Found in one of my own Android apps during a self-audit. The app is not public yet, so the examples below use a generic package name (`com.example.tracker`) and simplified code.*

## The situation

While building an on-device usage-tracker app, I added a few **debug-only BroadcastReceivers** so I could drive the app from my PC with `adb`:

- one that **exports the app's database** (usage history, call log summary, contacts-derived data) to a file,
- one that turns on a **"discovery mode"** that logs the structure of whatever app is on screen,
- one that creates a **fake media session** for testing.

They live in `src/debug/AndroidManifest.xml`, so they are never part of the release build. That felt safe.

```xml
<!-- src/debug/AndroidManifest.xml (vulnerable) -->
<receiver
    android:name=".debug.ExportReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="com.example.tracker.EXPORT" />
    </intent-filter>
</receiver>
```

```kotlin
class ExportReceiver : BroadcastReceiver() {
    override fun onReceive(c: Context, i: Intent) {
        val result = goAsync()
        CoroutineScope(Dispatchers.IO).launch {
            val file = Exporter.copyDatabase(c.applicationContext) // getExternalFilesDir()/tracker_<time>.db
            result.setResult(0, file.absolutePath, null)
            result.finish()
        }
    }
}
```

## The problem

`android:exported="true"` with **no `android:permission`** means *every app on the device* can send this broadcast — not just `adb`. A zero-permission app can do:

```kotlin
context.sendOrderedBroadcast(
    Intent("com.example.tracker.EXPORT")
        .setClassName("com.example.tracker", "com.example.tracker.debug.ExportReceiver"),
    null
)
```

And "debug only" was a weaker guarantee than I thought: **the build I actually run every day on my own phone is the debug build**, installed with `adb install`. So the exposed surface was live on a real device holding real personal data.

What a malicious app could do:

| Receiver | Effect when triggered by another app |
|---|---|
| Export | Creates a new timestamped copy of the full database on every call. Repeated calls fill storage (no rate limit, no cleanup) and multiply copies of sensitive data outside the app's private database directory. The result path is returned to the caller. |
| Discovery mode | Starts up to 5 minutes of logging the on-screen view tree of *other* apps (view IDs, classes, bounds, short content descriptions) to logcat, which then ends up in bug reports. |
| Fake media | Injects fake playback sessions, corrupting the statistics the app records. |

Mitigating factors (to be fair): on Android 11+ other apps cannot read `Android/data/<package>/` directly, and normal apps cannot read other apps' logcat. So this is not a one-click data theft — it is **unauthenticated control of privileged functionality** (CWE-926, *Improper Export of Android Application Components*), which is exactly the kind of bug that gets chained with something else.

## The fix

The receivers only ever needed to be called from `adb shell`. The shell user holds `android.permission.DUMP`, which normal third-party apps cannot obtain. Requiring it keeps the `adb` workflow working and blocks every other app:

```xml
<receiver
    android:name=".debug.ExportReceiver"
    android:exported="true"
    android:permission="android.permission.DUMP">
    ...
</receiver>
```

`adb shell am broadcast -a com.example.tracker.EXPORT -n com.example.tracker/.debug.ExportReceiver` still works; a third-party app's broadcast is now dropped by the system.

## Lessons

1. **"Debug only" is not "safe".** If you run debug builds on your daily phone, your debug surface is production surface.
2. **Every `exported="true"` needs a reason and a guard.** If only `adb` should reach it, use `android.permission.DUMP`. If only your own apps should, use a `signature`-level custom permission.
3. **Audit the merged manifest**, not just `src/main` (`app/build/intermediates/merged_manifests/...`), because build-type manifests add components silently.
4. Quick check you can run on any of your apps:

```bash
adb shell dumpsys package com.example.tracker | grep -A3 "Receiver"
```

Look for receivers with an action filter and no `permission` line.
