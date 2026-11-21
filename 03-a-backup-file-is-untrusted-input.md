# 03 — A backup file is untrusted input too: one integer freezes the app, one date breaks a screen for good

*Found in one of my own apps (a small private Flutter app) during a self-audit, fixed, and verified with before/after tests on the Dart VM. The app is private, so the examples below use generic names and simplified code. Field names like `pages` and `finishedAt` are kept because the bugs live in them.*

## The situation

The app logs reading sessions: each entry has a number of `pages`, each book has `totalPages` and, once finished, a `finishedAt` date. Pages turn into XP, and XP turns into a level shown on the home screen:

```dart
// XP needed to finish level n: 100, 300, 600, 1000, ...
int xpToFinishLevel(int n) => 100 * n * (n + 1) ~/ 2;

int levelFor(int xp) {
  var n = 1;
  while (xp >= xpToFinishLevel(n)) {
    n++;
  }
  return n;
}
```

There is no server. All data lives in a local database, and the only way data moves between devices is a JSON **backup file**: export shares it through the system share sheet, import picks a file and replaces everything with its contents.

```dart
static Future<AppData?> pickAndParse() async {
  final bytes = await pickFile();
  final j = jsonDecode(utf8.decode(bytes));
  if (j is! Map) throw const FormatException('not a JSON object');
  return AppData.fromJson(Map<String, dynamic>.from(j)); // types checked, values not
}
```

`fromJson` checks that `pages` is an `int` and `finishedAt` is a parseable date. It does not check that the int is a sensible number of pages or that the date is a sensible date. I treated the backup as "my own data". But a backup file is a file: it gets sent through chat apps, copied to other devices, and edited by hand. Once it leaves the app, it is input like any other.

## Problem 1: one big number, and the level loop never ends

On native platforms (the Dart VM, and AOT-compiled code on Android and iOS), Dart's `int` is a 64-bit two's-complement integer, and arithmetic **wraps around silently**. There is no overflow exception.

Look at `xpToFinishLevel` again. `100 * n * (n + 1)` wraps once `n` gets large, and after the `~/ 2` the result can never be more than 2^62 - 1. So for any `xp >= 2^62`, the condition `xp >= xpToFinishLevel(n)` is true for every `n`. The loop has no reachable exit (CWE-190 leading to CWE-835).

A backup with one entry of `"pages": 4611686018427387904` (2^62) is enough. During import the app computes the new level before it saves anything, and that call never returns: the UI thread is stuck and the app freezes. Ironically, the hang protects the data: the save is never reached, so after a force-close the old data is still there.

Values just below 2^62 are worse, because they get saved. With `"pages": 4000000000000000000` the loop does finish, at level 282,842,712, after 282 million iterations. I measured one `levelFor` call at about 2.1 s in JIT and 248 ms in an AOT build on my laptop. The import "succeeds", and from then on the level is computed on every launch and twice on every home screen build. The whole import-and-render path took 10.7 s in my test. I did not measure this on a phone.

And one more value: `9223372036854775807` (2^63 - 1). Adding it to the other entries wraps the XP total to a negative number. Nothing freezes, but the level progress bar gets a value of about -9.2e16. Silent wraparound produces wrong answers as well as hangs.

## Problem 2: a date that parses but can't be used

`DateTime` in Dart covers ±8.64e15 ms around 1970 (100,000,000 days each way), from the year −271821 to the year 275760. `DateTime.parse('-271821-04-20T00:00:00.000Z')` is exactly the earliest moment it can represent. It parses, it saves, it loads back.

The statistics screen groups finished books by month:

```dart
for (final b in s.finishedBooks) {
  final f = b.finishedAt!;
  final k = DateTime(f.year, f.month); // first day of that month
  perMonth[k] = (perMonth[k] ?? 0) + 1;
}
```

The first day of April −271821 is **before** the earliest representable moment, so the `DateTime` constructor throws `ArgumentError`. Any `finishedAt` from 20 to 30 April −271821 passes import and breaks the screen. And it does not happen once: the date is now in the local database, so the screen throws on every build, on every launch, until the user clears the app's data. In a release build Flutter replaces a widget that throws with a grey box (I did not check this on a phone).

Two smaller cases on the same path:

- A book with status "finished" and no `finishedAt` at all. The app itself can't create this state, but a hand-made backup can. The `!` above throws a null-check error, and the monthly summary card has the same `!`.
- `"totalPages": -5`. The `remaining` getter does `(totalPages - pagesRead).clamp(0, totalPages)`, and `int.clamp` throws `ArgumentError` when the upper bound is below the lower bound. That getter is used on the home screen.

All of these are CWE-20 (*Improper Input Validation*). None of them leaks data, and all of them need the user to import a crafted file. But the result is an app that stays broken after a restart, and "send me your backup, I added some books" is a very ordinary message.

## The fix

Two layers, because one isn't enough.

**1. Validate at the import boundary, before anything is saved.** A new `validateBackup` function runs after `fromJson` and before `replaceAll`:

```dart
AppData validateBackup(AppData d, {required DateTime now}) {
  final latest = now.add(const Duration(days: 1));
  void number(String field, int v, int min, int max) {
    if (v < min || v > max) {
      throw FormatException('Invalid value in backup: $field = $v');
    }
  }
  void date(String field, DateTime t) {
    if (t.isBefore(DateTime.utc(1900)) || t.isAfter(latest)) {
      throw FormatException('Invalid date in backup: $field');
    }
  }
  for (final b in d.books) {
    number('totalPages', b.totalPages, 1, maxPages);
    number('pagesRead', b.pagesRead, 0, maxPages);
    date('addedAt', b.addedAt);
    if (b.finishedAt != null) date('finishedAt', b.finishedAt!);
    else if (b.isFinished) throw const FormatException('finished book without a date');
  }
  for (final e in d.entries) {
    number('pages', e.pages, 0, maxPages);
    date('date', e.date);
  }
  // ...settings: daily goal >= 1, reminder hour 0..23, minute 0..59
  return d;
}
```

`maxPages` is the same limit the "add book" form already uses, so a real backup never hits it.

I had to choose between **rejecting the whole file** and **skipping the bad records**. I reject the whole file. Import already replaces all local data and already fails as a whole: any exception shows "Import failed: ..." and nothing is written. Skipping records would save an incomplete backup as if it were a full restore, and it would leave reading entries pointing to books that were dropped. The one exception is the cover image fields: they only affect how a book looks, so an unexpected cover URL or a local path outside the app's covers folder is cleared instead of rejected. The book stays and shows its plain colored spine.

I put the checks in the import path on purpose, **not** in `fromJson`. The local database loads through the same `fromJson`. If it threw there, someone who already had bad data on disk would get an app that can't even start.

**2. Make the use sites safe for data that is already on disk.** Validation at import doesn't help an install that imported the bad file last week. So:

```dart
const maxXp = 1 << 40; // far above anything reachable by reading

int levelFor(int xp) {
  if (xp < 100) return 1;
  final x = xp > maxXp ? maxXp : xp;
  // closed form for the n where 50n(n+1) <= x, then fix floating-point error
  var m = ((sqrt(1 + 8 * x / 100) - 1) / 2).floor();
  while (xpToFinishLevel(m + 1) <= x) m++;
  while (m > 0 && xpToFinishLevel(m) > x) m--;
  return m + 1;
}

// when summing XP: count at most maxPages per entry
var xp = entries.fold(0, (s, e) => s + e.pages.clamp(0, maxPages));
```

The two small `while` loops only correct the square-root estimate by a step or two, and with `x <= 2^40` nothing can wrap. For the statistics screen, the grouping moved into a small function that skips a missing date or one with a year outside 1..9999 instead of throwing. The summary card checks for `null`. `remaining` treats a negative `totalPages` as 0.

## How I verified it

Same idea as last time: run the same bad inputs before and after, and check that normal data behaves exactly as before.

I copied the app's pure Dart logic twice: once from the commit before the fix, once from the fixed code. Both copies run one shared test script. For each input it goes through the real chain: import, the work `replaceAll` does (queue, badges, XP, level), saving and loading back, then the computations the screens do. One detail matters here: **a synchronous infinite loop can't be stopped by a test timeout in the same isolate**, because the timer never gets to run. So every case runs in its own isolate, and the script kills it after 20 seconds.

Results (Dart VM on a laptop; a selection):

| Input | Before | After |
|---|---|---|
| entry `pages` = 2^62, import | timeout (20 s), killed | rejected, 1 ms |
| entry `pages` = 4e18, import | accepted, level 282,842,712, 10.7 s | rejected |
| entry `pages` = 2^62, put straight into the local store | timeout (20 s) | opens, level 20 |
| `levelFor(4e18)`, JIT / AOT | 2,117 ms / 248 ms | 2 µs / under 1 µs |
| `levelFor(2^62)` and `levelFor(2^63 - 1)` | timeout (20 s) | 2–6 µs |
| `finishedAt` = -271821-04-20Z, import | accepted, stats throws `ArgumentError` | rejected |
| same date, already on disk | stats throws | stats opens, book not counted |
| finished book without `finishedAt` | stats and summary throw | rejected; on disk: opens |
| `totalPages` = −5 | `remaining` throws | rejected; on disk: 0 |
| cover URL on an unknown host | saved as is | book kept, cover cleared |
| two normal backups | works | works, identical output |

For "normal data looks the same", I compared the new `levelFor` and `levelProgress` against the old loop for every XP value from -1,000 to 2,000,000 (2,001,001 values): 0 differences. I also checked every level boundary `50n(n+1)` and the value just below it for n = 1 to 148,290: 0 differences.

The fix also adds 9 permanent tests: the overflow values in an isolate with a 5-second limit, the equivalence check, the date grouping with an out-of-range date and a missing one, rejection of each bad field, cover clean-up, and a widget test that opens the statistics screen with bad data already in the local store. I ran that widget test once against the old screen code to make sure it catches the bug, and it failed with the null-check error. The full suite passes (134 tests) and the analyzer reports no issues.

## Lessons

1. **A backup file is untrusted input.** "It's our own format" says nothing about who edited it last. Validate values, not just types, at the point where outside data becomes stored data.
2. **On native platforms, Dart `int` wraps silently.** Any loop whose exit depends on arithmetic over user-controlled numbers can lose its exit. Bound the input, and prefer a closed-form calculation over "count up until it's big enough".
3. **"It parses" is not "it's usable".** `DateTime.parse` accepts dates that `DateTime(year, month)` rejects. Set a range that makes sense for your app (I used 1900 to tomorrow) and check it.
4. **Validate at the boundary, and defend where the value is used.** Import validation protects new data. Data already on disk only gets protection at the use site, and the use site must not throw on bad data, or the screen stays broken.
5. **Pick reject-whole or skip-record on purpose.** If the import replaces everything, partial success is a silent data loss. Rejecting with a clear message is easier to understand.
6. **Test hangs in a separate isolate.** A timeout in the same isolate can't stop a synchronous loop.
7. Quick check you can run on your own Flutter app:

```bash
# loops that count up until a computed threshold
grep -rnE "while \(.*>=?.*\(" lib/
# non-null assertions on dates read from storage
grep -rnE "(At|Date|date)!" lib/
# clamp whose upper bound comes from data
grep -rn "\.clamp(0, " lib/
```

Then ask, for each hit: what happens if the value came from a file someone else edited?
