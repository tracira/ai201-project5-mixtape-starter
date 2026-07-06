# Project 5: Mixtape Bug Hunt — Submission

## Codebase Map

### Main Files

**`app.py`** — Flask application factory. Defines the `create_app()` function that configures the SQLite database, initializes SQLAlchemy, and registers the four route blueprints (`songs`, `playlists`, `users`, `feed`). The `db` object is imported from here everywhere else. Running the app with `python app.py` triggers a double-import of `db` via `__main__`, so `FLASK_APP=app:create_app flask run` must be used instead.

**`models.py`** — Five SQLAlchemy models plus three association tables:
- `User` — username, email, `listening_streak` (int), `last_listened_at` (datetime), friend relationships (many-to-many, self-referential via `friendships` table)
- `Song` — title, artist, album, genre, `shared_by` (FK to User), `share_note`
- `Tag` — simple name string; linked to Song via `song_tags` many-to-many table
- `ListeningEvent` — records each time a user listens to a song (user_id, song_id, listened_at)
- `Rating` — a user's 1–5 score for a song; unique constraint on (user_id, song_id) enforces one rating per user per song
- `Playlist` — name, creator, `is_collaborative` flag
- `Notification` — user_id, notification_type, body, read flag
- `playlist_entries` — join table adding `position` (int) and `added_by` columns to the Playlist↔Song many-to-many relationship

**Routes layer** (`routes/`) — Thin HTTP handlers. Each route parses the request, calls exactly one service function, and formats the JSON response. No business logic lives here.

| Route file | Prefix | What it handles |
|---|---|---|
| `songs.py` | `/songs` | Song search, detail, listen events, ratings |
| `playlists.py` | `/playlists` | Create playlist, get songs, add song to playlist |
| `users.py` | `/users` | User profile, streak, notifications |
| `feed.py` | `/feed` | Friends Listening Now, activity feed |

**Services layer** (`services/`) — All business logic. One file per domain:
- `streak_service.py` — `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares today's date to `user.last_listened_at` to decide whether to increment or reset.
- `feed_service.py` — `get_friends_listening_now()` fetches `ListeningEvent` rows within a recency window and deduplicates to one entry per friend. `get_activity_feed()` returns the last N events across all friends with no recency filter.
- `search_service.py` — `search_songs()` queries Song with a case-insensitive `ILIKE` on title and artist, outer-joining the `song_tags` table to include tag data.
- `notification_service.py` — `add_to_playlist()` appends a song to a playlist and fires a notification to the original sharer. `rate_song()` upserts a Rating. `get_notifications()` / `mark_as_read()` handle retrieval and status.
- `playlist_service.py` — `get_playlist_songs()` queries songs joined via `playlist_entries` ordered by `position`. `create_playlist()` and `get_user_playlists()` handle playlist CRUD.

**`seed_data.py`** — Creates Users, Songs, Playlists, ListeningEvents, and Friendships in the SQLite DB for manual testing.

---

### Data Flow: User Rates a Song → Notification Sent

1. Client sends `POST /songs/<song_id>/rate` with `{user_id, score}`.
2. `routes/songs.py:rate()` extracts the fields and calls `notification_service.rate_song(user_id, song_id, score)`.
3. `rate_song()` validates the score (1–5), looks up the Song and User, then upserts a `Rating` row (creates new or updates existing via unique constraint on user_id+song_id).
4. *(Bug #4: the function stops here — it never notifies the song's original sharer.)*
5. The route returns `rating.to_dict()` with HTTP 201.

### Data Flow: User Listens to a Song → Streak Updated

1. Client sends `POST /songs/<song_id>/listen` with `{user_id}`.
2. `routes/songs.py:listen()` calls `streak_service.record_listening_event(user_id, song_id)`.
3. `record_listening_event()` creates a `ListeningEvent` row, then calls `update_listening_streak(user, now)`.
4. `update_listening_streak()` computes `days_since_last` by diffing today's date against `user.last_listened_at`. If 0 → no change; if 1 → increment streak; else → reset to 1.
5. The event is committed and returned.

### Data Flow: User Fetches Playlist Songs

1. Client sends `GET /playlists/<playlist_id>/songs`.
2. `routes/playlists.py:get_songs()` calls `playlist_service.get_playlist_songs(playlist_id)`.
3. `get_playlist_songs()` queries `Song` joined to `playlist_entries` filtered by playlist_id, ordered by `position` ascending.
4. *(Bug #5: the function returns `songs[:-1]`, slicing off the last song.)*
5. Route returns `{songs: [...], count: N}`.

### Patterns I noticed

- **Thin routes, fat services**: Every route is ≤10 lines of logic. All decisions and queries are in the service layer.
- **No authentication**: `user_id` is passed in the request body or URL. There's no session or token verification — this is a learning project.
- **UUID primary keys**: All models use `generate_uuid()` as the default PK, which avoids sequential-ID enumeration.
- **Consistent error pattern**: Services raise `ValueError` with a message; routes catch it and return a 400 or 404 JSON error.
- **SQLAlchemy associations have extra columns**: `playlist_entries` adds `position` and `added_by` beyond a plain join table — songs have explicit ordering, not just insertion order.

---

## The Five Issues

### Issue 1 — My listening streak keeps resetting (`streak_service.py:73`)

**Bug:** `elif days_since_last == 1 and today.weekday() != 6:` — `weekday() == 6` is Sunday. This condition means "only increment the streak if today is NOT Sunday." So if you listen on Saturday and then on Sunday (one consecutive day), the condition is false and the streak falls through to the `else` branch, resetting to 1.

**Fix:** Remove `and today.weekday() != 6`.

```python
# Before
elif days_since_last == 1 and today.weekday() != 6:

# After
elif days_since_last == 1:
```

---

### Issue 2 — Friends Listening Now shows people from yesterday (`feed_service.py:13`)

**Bug:** `RECENT_THRESHOLD = timedelta(hours=24)` — 24 hours is far too wide for a "listening now" feature. Someone who listened 23 hours ago (yesterday) shows up as currently listening.

**Fix:** Change the threshold to 30 minutes (or whatever the product intent is for "now").

```python
# Before
RECENT_THRESHOLD = timedelta(hours=24)

# After
RECENT_THRESHOLD = timedelta(minutes=30)
```

---

### Issue 3 — The same song keeps showing up twice in search (`search_service.py:26`)

**Bug:** The query does `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`. A song with two tags produces two rows in the join result, so it appears twice in the output list.

**Fix:** Add `.distinct()` to the query before `.all()`.

```python
# Before
.all()

# After
.distinct()
.all()
```

---

### Issue 4 — No notification when a friend rates my song (`notification_service.py:73`)

**Bug:** `rate_song()` saves the `Rating` and commits, but never calls `create_notification()`. The `add_to_playlist()` function (same file) demonstrates the correct pattern — it notifies the song's sharer after the write.

**Fix:** Add a `create_notification` call after saving the rating, mirroring `add_to_playlist`.

```python
# After db.session.commit() in rate_song():
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

---

### Issue 5 — The last song in a playlist never shows up (`playlist_service.py:66`)

**Bug:** `return [song.to_dict() for song in songs[:-1]]` — `[:-1]` is a Python slice that drops the last element of the list. Every playlist query silently omits the final song.

**Fix:** Remove the slice.

```python
# Before
return [song.to_dict() for song in songs[:-1]]

# After
return [song.to_dict() for song in songs]
```

---

## Bugs I'm Fixing

I'll fix Issues **1, 3, 4, and 5** — four of the five. Issue 2 (the feed threshold) is a one-liner I'll include as well.

- Issue 1: `streak_service.py` — remove Sunday guard condition
- Issue 2: `feed_service.py` — fix recency threshold
- Issue 3: `search_service.py` — add `.distinct()`
- Issue 4: `notification_service.py` — add notification on rating
- Issue 5: `playlist_service.py` — remove `[:-1]` slice

---

## Bug Reproduction Log

### Issue 1 — Streak reset on Sunday

**How I reproduced it:** Ran a direct Python script against the service function with synthetic data. Set a user's `last_listened_at` to Saturday 2026-07-04 and `listening_streak` to 5, then called `update_listening_streak(user, sunday)` where `sunday = datetime(2026, 7, 5, ...)` (weekday=6). The streak reset from 5 to 1 instead of incrementing to 6.

The condition `today.weekday() != 6` evaluates to `False` on Sunday, so the `elif` branch (increment) is skipped and the `else` branch (reset) fires even though only one day has passed.

**Test that catches it:** `tests/test_streaks.py::test_streak_increments_on_sunday` — FAILING before fix.

---

### Issue 2 — Stale events in Friends Listening Now

**How I reproduced it:** Queried `GET /feed/<kenji_id>/listening-now`. Kenji's only friend with a listening event is nova, whose most recent event was seeded 2 hours ago. The response returned nova as "listening now" even though she listened 2.2 hours ago — well outside any reasonable "now" window. The 24-hour `RECENT_THRESHOLD` accepts events from up to yesterday.

**Expected behavior after fix:** With a 30-minute threshold, nova's 2.2-hour-old event drops off, the count goes to 0 for kenji.

---

### Issue 3 — Duplicate songs in search

**How I reproduced it:** Ran the raw SQL that the buggy ORM query generates against the database:
```sql
SELECT song.id, song.title, song_tags.tag_id
FROM song
LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
WHERE lower(song.title) LIKE lower("%Crown%")
```
This returned **3 rows** for "Crown Heights Anthem" — one per tag (rap, hip-hop, boom bap). The bug exists at the SQL level. SQLAlchemy 2.0's ORM layer happens to deduplicate the objects by primary key before returning them, so the duplication is hidden in the current Python output. On older SQLAlchemy (1.x) or if the query were rewritten to return dicts directly, all 3 copies would appear in the result.

**Test that catches it:** `test_search_no_duplicates_multi_tag_song` passes due to ORM deduplication — the test comment says `# bug causes it to be 3` but SQLAlchemy 2.0 masks it. The fix (`DISTINCT`) is still correct and removes the unnecessary join rows.

---

### Issue 4 — No notification on song rating

**How I reproduced it:**
1. Checked simone's notifications before the test: `GET /users/<simone_id>/notifications` → count: 0
2. Nova rated simone's song (Crown Heights Anthem): `POST /songs/<song_id>/rate` with `{"user_id": nova_id, "score": 5}` → returned the rating object successfully (HTTP 201)
3. Checked simone's notifications again: still count: 0

The rating was saved and committed, but no `Notification` row was created. The `add_to_playlist` function in the same file shows the pattern that was missing.

---

### Issue 5 — Last playlist song missing

**How I reproduced it:** Queried `GET /playlists/<late_night_vibes_id>/songs`. The "Late Night Vibes" playlist has 7 songs in the database (confirmed by direct SQL query on `playlist_entries`). The API returned `count: 6` — the song at position 7 ("Free Throws" by Hoop Dreams) was silently dropped.

The `songs[:-1]` slice in `playlist_service.py:66` removes exactly the last element of the sorted-by-position list, which is always the final track.

**Tests that catch it:** `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` — both FAILING before fix.

---

## Fix Log

### Issue 5 — Last playlist song missing
**File:** `services/playlist_service.py:66`  
**Change:** `songs[:-1]` → `songs`  
**Commit:** `fix: remove off-by-one slice that dropped last playlist song`  
**Tests:** `test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order` — both now passing.

---

### Issue 1 — Streak reset on Sunday
**File:** `services/streak_service.py:73`  
**Change:** `elif days_since_last == 1 and today.weekday() != 6:` → `elif days_since_last == 1:`  
**Commit:** `fix: correct Sunday boundary condition in streak reset logic`  
**Tests:** `test_streak_increments_on_sunday` — now passing.

---

### Issue 2 — Stale events in Friends Listening Now
**File:** `services/feed_service.py:13`  
**Change:** `timedelta(hours=24)` → `timedelta(minutes=30)`  
**Commit:** `fix: reduce Friends Listening Now threshold from 24 hours to 30 minutes`  
**Verified:** Kenji's feed returned count=0 after fix (nova's 2.2-hour-old event excluded). Nova's feed still returned 3 friends whose events were within 30 minutes.

---

### Issue 3 — Duplicate songs in search
**File:** `services/search_service.py:34`  
**Change:** Added `.distinct()` before `.all()` in the search query  
**Commit:** `fix: add DISTINCT to search query to prevent duplicate results for multi-tag songs`  
**Tests:** All 5 search tests passing. Raw SQL confirmed DISTINCT eliminates the 3 duplicate rows for "Crown Heights Anthem".

---

### Issue 4 — No notification on song rating
**File:** `services/notification_service.py` (after the `db.session.commit()` in `rate_song`)  
**Change:** Added `create_notification()` call for the song's original sharer, guarded by `song.shared_by != user_id`  
**Commit:** `fix: send notification to song sharer when a friend rates their song`  
**Verified:** After fix, rating a song triggers a `song_rated` notification to the sharer's inbox.
