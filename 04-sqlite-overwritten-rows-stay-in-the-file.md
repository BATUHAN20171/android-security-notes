# 04 — Overwriting a row in SQLite doesn't erase it: secure_delete, WAL, and why exports should use VACUUM INTO

*Found in one of my own Android apps during a self-audit, fixed, and measured with a synthetic database on desktop SQLite (3.45.1 and 3.54.1). The app is private, so the examples below use a generic package name (`com.example.tracker`) and simplified code. **I did not measure this on a phone.** The last section says what that means for Android.*

## The situation

The app is a local analytics app that stores notification metadata in a Room database. An older version saved one metadata field (the notification channel ID) without redacting it, and for some apps that field can contain a personal identifier. I fixed the redaction for new rows. For old rows, the new version runs a one-time maintenance job that rewrites them:

```kotlin
object NotificationMaintenance {
    suspend fun scrubOldChannelIds(context: Context) = withContext(Dispatchers.IO) {
        val db = AppDatabase.get(context)
        if (db.settings().get(SCRUB_DONE) == "1") return@withContext
        val sdb = db.openHelper.writableDatabase
        db.runInTransaction {
            // "Don't leave the old bytes in free space: the export copies the file."
            sdb.query("PRAGMA secure_delete = ON").use { it.moveToFirst() }
            for (old in distinctChannelIds(sdb)) {
                val clean = redact(old)
                if (clean != old) {
                    sdb.execSQL("UPDATE event SET channel = ? WHERE channel = ?", arrayOf(clean, old))
                }
            }
            sdb.execSQL("INSERT OR REPLACE INTO settings VALUES (?, '1')", arrayOf(SCRUB_DONE))
        }
    }
}
```

The app also has an export button. It runs the job first, then copies the database file somewhere I can pull it to my PC:

```kotlin
object Exporter {
    suspend fun export(context: Context): File = withContext(Dispatchers.IO) {
        val db = AppDatabase.get(context)
        NotificationMaintenance.scrubOldChannelIds(context)
        db.openHelper.writableDatabase.query("PRAGMA wal_checkpoint(TRUNCATE)").use { it.moveToFirst() }
        val target = File(context.getExternalFilesDir(null), "tracker_$stamp.db")
        context.getDatabasePath("tracker.db").copyTo(target, overwrite = true)
        target
    }
}
```

The comment in the job shows what I believed: with `secure_delete` on, the old values are gone, so copying the file is fine. The SQLite documentation seems to agree: *"Applications that wish to avoid leaving forensic traces after content is deleted or updated should enable the secure_delete pragma prior to performing the delete or update, or else run VACUUM after the delete or update."* I enabled it before the update. I wanted to check what the file really contains afterwards.

## Where old bytes can hide in a database file

A SQLite file is a list of fixed-size pages (4096 bytes here). A table is a b-tree of pages. A leaf page holds a header, an array of cell pointers at the top, and the cells (the rows) packed at the bottom. Between the pointer array and the cells is **unallocated space**. When a row is deleted or rewritten, its old cell becomes a **freeblock** inside the page. Pages that are no longer needed go to the **freelist**. None of these are erased by default: SQLite only marks the space as reusable.

In WAL mode (Room's default on most devices) there is one more place. A write doesn't change the main file at all. The new version of every changed page is added to the `-wal` file, and the main file keeps the old version until a **checkpoint** copies the new pages back.

## The experiment

I built a synthetic database with the same shape as the app's: 800 rows, 200 of them with a ~140-byte "old" value that starts with a marker (`LEGACY-SECRET-0000` to `-0199`), the rest filler. Then, like the maintenance job, one transaction updates the 200 values to short ones. At each step a script counts the marker in the main file and in the `-wal`, takes a raw copy of the main file, and takes a `VACUUM INTO` copy. It also parses the pages to find where each leftover sits. SQLite 3.45.1 and 3.54.1 gave the same numbers, and repeated runs give identical output.

WAL mode, `auto_vacuum` off (FULL made no difference here). "Written with" is the `secure_delete` setting when the old rows were first stored; "scrubbed with" is the setting during the update. Each number is how many copies of the marker are in that file; there are 200 old values.

| Written with | Scrubbed with | `-wal` after the update | Raw copy after checkpoint | `VACUUM INTO` copy | Main file after `VACUUM` + checkpoint |
|---|---|---|---|---|---|
| off | off | 200 | 213 | 0 | 0 |
| off | **on** | 0 | **13** | 0 | 0 |
| on | on | 0 | 0 | 0 | 0 |
| on | off | 200 | 200 | 0 | 0 |

Rollback-journal (DELETE) mode gave the same main-file numbers: 213, 13 and 0.

Four things in this table surprised me.

**1. With `secure_delete` off, all 200 old values stay.** 193 sit in freeblocks of the leaf pages and 7 in unallocated space. That part is expected: it's what the pragma is for.

**2. With `secure_delete` on, 13 still stay, and they did not come from the update.** Before any update, the file already had 213 copies: 200 live values plus 13 extra. The page parser shows where the extra 13 are: in the unallocated space of the table's **root page**, which is an interior node, and they are exactly the first 13 rows ever inserted. My reading of SQLite's `btree.c` explains it. When the first leaf page fills up, `balance_deeper()` copies its content to a new child page and turns the root into an interior node with `zeroPage()`. `zeroPage()` clears the page only if `secure_delete` is on *at that moment*. It was off when the data was written, so the old cells stayed in the root page as dead bytes. The later update never touches the root page, and those bytes aren't "deleted content" in SQLite's eyes, so `secure_delete` has nothing to clear. (This is my interpretation of the source plus the measurement; the documentation doesn't describe it.) The current SQLite trunk has the same code.

So the documented advice is true for the content the update frees, but enabling the pragma before the scrub doesn't help with copies SQLite left behind *when the data was written*, if the pragma was off then.

**3. In WAL mode, a raw copy without a checkpoint is worse than leftovers.** Right after the update, the main file is still the old version. Opening that copy with SQL shows **all 200 old values as live rows**, whatever `secure_delete` was. My exporter did checkpoint first, but `wal_checkpoint` can return "busy" when another connection is reading, and the result was ignored. Also, with `secure_delete` off, the `-wal` itself holds the old values (200), because each WAL frame is a full page image, freeblocks included. `TRUNCATE` empties it; `PASSIVE` checkpoints (the automatic kind) don't. I didn't measure the PASSIVE case.

**4. `VACUUM` in WAL mode doesn't clean the main file by itself.** It writes the rebuilt database into the `-wal`. The main file still had 213 (or 13) markers after `VACUUM` and 0 only after the following `wal_checkpoint(TRUNCATE)`.

`VACUUM INTO` was 0 in every case, and so was `VACUUM` + `TRUNCATE` checkpoint. The documentation says this too: a `VACUUM INTO` copy has *"all deleted content ... purged from the backup, leaving behind no forensic traces"*.

## The fix

**1. Export with `VACUUM INTO`, not a file copy.** It writes a new database that contains only live data. It reads through the WAL, so it doesn't need a checkpoint, and it doesn't change the source file.

```kotlin
object DbFiles {
    /** VACUUM INTO arrived in SQLite 3.27.0. Android 9-10 ship 3.22, Android 11 ships 3.28. */
    fun hasVacuumInto(version: String): Boolean {
        val p = version.trim().split('.').map { it.toIntOrNull() ?: return false }
        if (p.size < 2) return false
        return p[0] > 3 || (p[0] == 3 && p[1] >= 27)
    }

    /** Must not run inside a transaction: VACUUM fails there. */
    fun compact(db: SupportSQLiteDatabase): Boolean {
        db.execSQL("VACUUM")
        repeat(3) { attempt ->
            val busy = db.query("PRAGMA wal_checkpoint(TRUNCATE)").use { it.moveToFirst() && it.getInt(0) != 0 }
            if (!busy) return true
            if (attempt < 2) Thread.sleep(100)
        }
        return false
    }

    fun cleanCopy(db: SupportSQLiteDatabase, source: File, target: File) {
        // VACUUM INTO fails if the target exists and isn't empty.
        if (target.exists() && !target.delete()) error("cannot replace ${target.name}")
        val version = db.query("SELECT sqlite_version()").use { if (it.moveToFirst()) it.getString(0) else "" }
        if (hasVacuumInto(version)) {
            db.execSQL("VACUUM INTO ?", arrayOf<Any?>(target.absolutePath))
        } else {
            // Old SQLite (API 29): rebuild the source, flush the WAL, then copy.
            check(compact(db)) { "database busy, try the export again" }
            source.copyTo(target, overwrite = true)
        }
    }
}
```

Three details:

- **Check the SQLite version, not only `SDK_INT`.** The official table lists 3.22 up to API 28 and 3.28 for API 30, and Android 10 (API 29) also has 3.22. The same page warns that some manufacturers ship different versions. `minSdk` is 29 in my app, so the fallback is needed.
- **Never call these inside `runInTransaction` / `withTransaction`.** `VACUUM` and `VACUUM INTO` fail with an open transaction on the connection.
- **On the fallback path, a busy checkpoint is an error.** A copy made after a partial checkpoint can still carry old pages, so the export fails with a message instead.

**2. After the scrub, compact the database.** In the maintenance job, after the transaction has finished:

```kotlin
if (changed > 0) DbFiles.compact(sdb)
```

**3. Know which fix protects whom.** The job sets a "done" flag and never runs again. On a device where it already ran, the new `compact` call will never happen, so the old bytes stay in the app's private database file. They only leave the phone through an export, and the export now writes a clean copy. That's why the export change is the important one.

## How I verified it

The before/after numbers are the table above: the old export path (checkpoint, then raw copy) left 13 or 213 markers; the new one (`VACUUM INTO`) and the fallback (`VACUUM` + `TRUNCATE`, then copy) left 0 in every scenario. The app builds, and its unit tests pass, including new tests for the version check (`3.22.0` → fallback, `3.27.0` and `3.28.0` → `VACUUM INTO`, and `3.9.2` → fallback, which a string comparison would get wrong).

## What I could not check

- **Nothing here was measured on a phone.** I didn't run `VACUUM INTO` through Android's own SQLite either; it's standard SQL in 3.27+, but I haven't seen it run on a device.
- **Android's framework SQLite probably has `secure_delete` on by default.** AOSP's build file (`external/sqlite/dist/Android.bp`, main branch) sets `-DSQLITE_SECURE_DELETE` and `-DSQLITE_DEFAULT_AUTOVACUUM=1`. If a device's build does the same, data is written with the pragma on, and my table says the leftovers should be close to the "on / on" row: 0. That would make the real impact on my phone smaller than the desktop numbers. I haven't confirmed the vendor build or measured the file. Apps that bundle their own SQLite (instead of using the framework one) get that library's defaults instead.
- The WAL and raw-copy problems (point 3) don't depend on `secure_delete`, so the export fix matters either way.

## Lessons

1. **Updating or deleting a row doesn't erase its bytes.** They stay in freeblocks, unallocated space, freelist pages, and the `-wal`, until something overwrites them.
2. **`secure_delete` only clears what the current operation frees.** It doesn't clear copies SQLite made earlier while it was off. If it matters, it has to be on from the first write, and even then it's not a substitute for the next point.
3. **Export with `VACUUM INTO`** (SQLite 3.27+, Android 11+ in practice). Check `sqlite_version()` at runtime and have a fallback for older versions: `VACUUM`, then `wal_checkpoint(TRUNCATE)` with the busy flag checked, then copy.
4. **In WAL mode, a raw copy of the main file is not the database.** Without a full checkpoint it can show old rows as live data. `VACUUM` in WAL mode only cleans the main file after a checkpoint.
5. **One-time jobs need a plan for devices where they already ran.** Fix the place where data leaves the device, not only the job.
6. Quick check you can run on your own database file (pull the main file **and** its `-wal`, for example from a debug build with `adb exec-out run-as com.example.tracker cat databases/tracker.db > tracker.db`):

```python
# usage: python leftover_check.py tracker.db "value-that-should-be-gone"
import os, sqlite3, sys, tempfile

path, needle = sys.argv[1], sys.argv[2].encode()
raw = open(path, "rb").read().count(needle)

out = os.path.join(tempfile.mkdtemp(), "live-only.db")
con = sqlite3.connect(f"file:{path}?mode=ro", uri=True)
con.execute("VACUUM INTO ?", (out,))   # SQLite 3.27+; writes live data only
con.close()
live = open(out, "rb").read().count(needle)

print(f"in the raw file: {raw}   in live data: {live}   leftover bytes: {raw - live}")
```

If "leftover bytes" isn't 0, a raw copy of that file carries data your app thinks it deleted. `strings tracker.db | grep -c "value"` gives the raw count too. Text is stored as UTF-8 by default, so a plain byte search works for text columns.
