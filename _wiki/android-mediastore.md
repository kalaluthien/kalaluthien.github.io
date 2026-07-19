---
share: true
title: Android MediaStore
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-17
updated: 2026-07-17
sources:
  - "[[2026-07-17-how-android-apps-load-gallery-photos]]"
aliases:
  - MediaStore
---

# Android MediaStore

The content provider that indexes shared media on Android. It is the enumeration
half of "load the user's photos"; the other half is [Android image decoding](/wiki/android-image-decoding/).

## Data model

- **`EXTERNAL_CONTENT_URI` is a synthetic volume.** `VOLUME_EXTERNAL` is a merged
  view of all attached external storage (primary + SD + USB). You cannot insert
  into it — inserts target a real volume, usually `VOLUME_EXTERNAL_PRIMARY`.
  `MediaStore.getExternalVolumeNames()` enumerates the real ones. Production
  galleries query the synthetic volume once and never iterate volumes.
- **`MediaStore.Files`** is the superset collection (all types); `Images`/`Video`
  are the filtered ones.
- **Buckets are hashed parent paths.** `BUCKET_ID = parentPath.toLowerCase().hashCode()`,
  `BUCKET_DISPLAY_NAME` = the directory's original-case name
  (AOSP `FileUtils.computeValuesFromData`). So: an "album" is a directory and
  nothing more; case-differing folders collide; renaming a folder changes its
  bucket ID. Both columns are read-only.
- **`DCIM/Camera` is a convention, not a contract.** OEM camera apps vary; Samsung
  One UI puts screenshots in `DCIM/Screenshots` where stock Android uses
  `Pictures/Screenshots`.

## The `DATA` column: deprecated, then un-deprecated

Deprecated in Android 10 when scoped storage broke raw paths; Android 11 re-enabled
direct file-path access to media through FUSE, and the current AOSP javadoc carries
no `@Deprecated` annotation. It is read-only for apps targeting R+. Google's own
guidance concedes sequential reads via path are comparable to MediaStore and random
access is only "up to twice as slow". This is why path-based pipelines still ship in
2026.

## Traps

- **Date columns have mixed units.** `DATE_TAKEN` is milliseconds
  (`@CurrentTimeMillisLong`); `DATE_ADDED`/`DATE_MODIFIED` are seconds. Mixing them
  corrupts sorting by 1000×.
- **`DATE_TAKEN` is derived and unreliable.** The scanner guesses a timezone offset
  from GPS time or file mtime, rounds it to 15 minutes, and returns NULL when EXIF
  `DateTimeOriginal` is missing. The javadoc concedes images need both
  `TAG_DATETIME_ORIGINAL` *and* `TAG_OFFSET_TIME_ORIGINAL` for a reliable value.
  Screenshots, messenger downloads, and DSLR imports routinely have NULL or shifted
  values. Fossify Gallery ships a user-facing "Fix Date Taken value" action and its
  own override database because of this.
- **For sync, use `MediaStore.getGeneration()`**, not dates — dates change when an app
  calls `setLastModified()` or the user changes the clock; the generation is monotonic.
- **`IS_PENDING` rows are filtered out of queries by default on Q+** (opt in with
  `QUERY_ARG_MATCH_PENDING`). A camera app that sets `IS_PENDING=1` during capture is
  invisible to other galleries *and to its own default queries* until it flips it.
- **CursorWindow is a fixed 2 MB shared-memory region** (`config_cursorWindowSize`),
  not resizable from the client. Iterating 100k rows is fine — windows refill — but a
  single oversized row throws `SQLiteBlobTooBigException`. Defense: narrow projections,
  never blob columns.
- **`ContentObserver.onChange` runs on the looper of the `Handler` you registered
  with.** A main-looper observer that re-queries inline puts a binder call plus SQLite
  on the main thread, and a downstream `flowOn(Dispatchers.IO)` does not move it.

## The Android 11 rupture

Android 11's MediaProvider added strict SQL grammar checking, ending two decade-old
hacks:

- `sortOrder = "date_added DESC LIMIT 100"` — LIMIT injected into ORDER BY.
- `selection = "1) GROUP BY 1,(2"` — the pre-Q album-list trick that turned the
  provider's `WHERE (<sel>)` into a GROUP BY, yielding one row per bucket with a free
  cover image.

For apps targeting < R these are "gracefully lifted out" (`recoverAbusiveSortOrder`,
`recoverAbusiveLimit`, `recoverAbusiveSelection` — MediaProvider's own comment reads
"Some apps are abusing ORDER BY clauses to inject LIMIT clauses"). Targeting R+ they
throw `IllegalArgumentException: Invalid token LIMIT`.

The sanctioned replacement is **Bundle query args** (API 26+):
`QUERY_ARG_SQL_SELECTION`, `QUERY_ARG_SORT_COLUMNS`/`QUERY_ARG_SORT_DIRECTION`, and —
with no classic equivalent — `QUERY_ARG_LIMIT`/`QUERY_ARG_OFFSET`. Providers report
what they honored via `EXTRA_HONORED_ARGS`. For album lists, the replacement is to
fetch all rows and group in memory.

## Permission timeline

| API | Release | Request | Get |
|-----|---------|---------|-----|
| ≤ 28 | ≤ 9 | `READ_EXTERNAL_STORAGE` | raw paths work everywhere |
| 29 | 10 | + `requestLegacyExternalStorage` opt-out | scoped storage; owner rule begins |
| 30 | 11 | `READ_EXTERNAL_STORAGE` | opt-out ignored for target R+; FUSE re-enables media paths; strict query grammar |
| 33 | 13 | `READ_MEDIA_IMAGES`/`_VIDEO`/`_AUDIO` | `READ_EXTERNAL_STORAGE` has no effect; Photo Picker ships |
| 34 | 14 | + `READ_MEDIA_VISUAL_USER_SELECTED` | "Select photos" partial access |

**The owner rule**: since Android 10 an app needs *no* storage permission to read media
it contributed (`owner_package_name`). The caveat that bites: ownership does not survive
uninstall/reinstall — the system attributes old files to the previous installation, so
the app must request read permission to see its own old photos.

**Partial access (14+)** is the subtler trap: `checkSelfPermission(READ_MEDIA_IMAGES)`
can report *granted* while MediaStore returns only the user-selected subset. Code that
assumes "granted = everything" sees a mysteriously tiny gallery. Don't cache permission
state; re-check in `onResume`.

## Two production architectures

Neither paginates. Both group in memory rather than SQL. Both still depend on `DATA`.

- **Filesystem-first** (Fossify Gallery): MediaStore discovers folders and prefetches
  metadata; the `File` API enumerates; own Room DBs override bad columns. Chosen because
  the product is "browse any folder, including hidden ones".
- **MediaStore-first** (Telegram): one full-table cursor per collection, albums built as
  a `SparseArray` keyed by `BUCKET_ID`, everything grouped in Java. Chosen because the
  product is "pick recent media fast". Telegram refuses `DATE_TAKEN` entirely on API > 28,
  swapping in `DATE_MODIFIED` — a production vote of no confidence.

Both wrap every cursor row in an empty catch and skip bad rows rather than crash.
Telegram catches `Throwable`, not `Exception`.

**A third shape** exists for an app showing only photos it captured: `RELATIVE_PATH`-filtered
queries with Bundle args, content URIs only, no `DATA`, no permission needed on 10+
(subject to the reinstall caveat).

## When not to enumerate at all

If the need is "let the user hand me some photos", enumeration is the wrong tool post-2022:

- **Photo Picker** (`ACTION_PICK_IMAGES`, API 33; backported via GMS to API 19): system UI,
  no permission, read-only picker URIs. `ActivityResultContracts.PickVisualMedia` encodes the
  real fallback chain. Persisted grants cap at **5,000**, oldest evicted.
- **SAF** (`ACTION_OPEN_DOCUMENT`): persistable grants across reboots; the cap was 128, raised
  to **512 in Android 11**.
- **`ACTION_PICK`**: not formally deprecated, but treat as legacy.

## Scan latency

A captured file can exist on disk before its row is visible (pending flag + scanner).
Telegram's belt-and-suspenders answer: debounced `ContentObserver`s (2000 ms) on four URIs
*plus* a cheap `COUNT(_id)` poll to catch missed events.
