# Project 5: Mixtape Bug Hunt — Submission

---

## AI Usage

I used Claude Code (the Claude Sonnet 4.6 CLI) throughout this project. Here is an honest account of what it helped with and where I had to verify or push back.

**Codebase orientation.** I gave the AI the contents of all five service files and asked it to summarize each one's responsibility, list its main functions, and trace two data flows (rate a song → notification; listen to song → streak update). This got me to a working mental model faster than reading alone — the data flow traces in the Codebase Map section are directly from that exercise. I read each file myself afterward to confirm the summary was accurate.

**Bug identification.** After reading all five files I asked the AI: "Given the issue titles, what exact lines in these files are the bugs?" It identified all five immediately — the `songs[:-1]` slice, the `weekday() != 6` guard, the missing `.distinct()`, the missing `create_notification()` call, and the 24-hour feed threshold. I verified each identification by reading the relevant line in context before accepting it.

**Reproduction.** For bugs with visible API symptoms (issues 2, 4, 5), I wrote the curl commands myself and ran them. For issue 1 (Sunday-only condition), I couldn't reproduce it through the live API without date manipulation, so I wrote a direct Python script calling the service function with a synthetic Sunday datetime — the AI suggested this approach and I implemented and ran it. For issue 3, the AI flagged that SQLAlchemy 2.0 would mask the duplication at the ORM layer, so I ran a raw SQL query to confirm the bug existed at the database level. I checked this myself and confirmed it.

**Where I verified the AI independently.** The AI initially described the search bug as "songs appear three times in the result list" — that's true at the SQL level but not in the Python output because SQLAlchemy 2.0 deduplicates by identity. Running the tests showed all search tests passing before the fix. I documented this discrepancy in the RCA rather than just accepting the AI's framing. The fix (`.distinct()`) is still correct — it eliminates the unnecessary join rows at the query level — but the reproduction story is more nuanced than the AI's first explanation.

**Code generation.** The AI wrote the one-line changes for issues 1, 2, 3, and 5. For issue 4, it generated the `create_notification()` block. I read each diff before committing to confirm the change was minimal and correct — none of the fixes introduced new logic beyond the specific thing being fixed.

---

## Codebase Map

### Main Files

**`app.py`** — Flask application factory. Defines `create_app()`, which configures the SQLite database, initializes SQLAlchemy, and registers the four route blueprints (`songs`, `playlists`, `users`, `feed`). The `db` object is imported from here everywhere else. Running with `python app.py` triggers a double-import of `db` via `__main__`, so `FLASK_APP=app:create_app flask run` is required.

**`models.py`** — Seven SQLAlchemy models and three association tables:
- `User` — username, email, `listening_streak` (int), `last_listened_at` (datetime), self-referential many-to-many friends via `friendships` table
- `Song` — title, artist, album, genre, `shared_by` (FK to User), `share_note`
- `Tag` — name string; linked to Song via `song_tags` many-to-many table
- `ListeningEvent` — records each time a user listens to a song (user_id, song_id, listened_at)
- `Rating` — a user's 1–5 score for a song; unique constraint on (user_id, song_id) enforces one rating per user per song
- `Playlist` — name, creator, `is_collaborative` flag
- `Notification` — user_id, notification_type, body, read flag
- `playlist_entries` — join table adding `position` (int) and `added_by` to the Playlist↔Song relationship — songs in a playlist have an explicit ordering, not just insertion order

**Routes layer** (`routes/`) — Thin HTTP handlers. Each route parses the request, calls exactly one service function, and formats the JSON response. No business logic lives here.

| File | URL prefix | Handles |
|---|---|---|
| `songs.py` | `/songs` | Song search, detail, listen events, ratings |
| `playlists.py` | `/playlists` | Create playlist, get songs, add song |
| `users.py` | `/users` | User profile, streak, notifications |
| `feed.py` | `/feed` | Friends Listening Now, activity feed |

**Services layer** (`services/`) — All business logic, one file per domain:
- `streak_service.py` — `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares today's date to `user.last_listened_at` to decide whether to increment or reset the streak.
- `feed_service.py` — `get_friends_listening_now()` fetches `ListeningEvent` rows within a recency window and deduplicates to one entry per friend. `get_activity_feed()` returns the last N events across all friends with no recency filter.
- `search_service.py` — `search_songs()` queries Song with a case-insensitive `ILIKE` on title and artist, outer-joining `song_tags` to include tag data.
- `notification_service.py` — `add_to_playlist()` appends a song to a playlist and fires a notification to the original sharer. `rate_song()` upserts a Rating. `get_notifications()` / `mark_as_read()` handle retrieval and status.
- `playlist_service.py` — `get_playlist_songs()` queries songs joined via `playlist_entries` ordered by `position`. `create_playlist()` and `get_user_playlists()` handle playlist CRUD.

**`seed_data.py`** — Creates 5 Users, 13 Songs, 3 Playlists, ListeningEvents, and Friendships for manual testing. Songs are split into groups with 0, 1, and 3+ tags to expose the search duplication issue.

---

### Data Flow: User Rates a Song → Notification Sent

1. Client sends `POST /songs/<song_id>/rate` with `{user_id, score}`.
2. `routes/songs.py:rate()` extracts the fields and calls `notification_service.rate_song(user_id, song_id, score)`.
3. `rate_song()` validates the score (1–5), looks up the Song and User, then upserts a `Rating` row (creates new or updates existing via unique constraint on user_id+song_id).
4. *(Before fix: the function stopped here — it never notified the song's original sharer.)*
5. After the commit, if `song.shared_by != user_id`, a `Notification` is created for the sharer.
6. The route returns `rating.to_dict()` with HTTP 201.

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
4. *(Before fix: the function returned `songs[:-1]`, silently dropping the last track.)*
5. Route returns `{songs: [...], count: N}`.

### Patterns I noticed

- **Thin routes, fat services** — Every route is ≤10 lines of logic. All decisions and queries live in the service layer.
- **No authentication** — `user_id` is passed in the request body or URL with no session or token check. This is a learning project.
- **UUID primary keys** — All models use `generate_uuid()` as the default PK.
- **Consistent error pattern** — Services raise `ValueError` with a message; routes catch it and return a 400 or 404 JSON error.
- **Association tables with extra columns** — `playlist_entries` adds `position` and `added_by` beyond a plain join, giving songs explicit ordering within a playlist.

---

## Root Cause Analysis

Each entry covers: **Symptom · Root cause · How I reproduced it · Fix · Verification**.

---

### Issue 1 — My listening streak keeps resetting

**Symptom:** Users reported their streak resetting to 1 unexpectedly, even after listening on consecutive days.

**Root cause:** `streak_service.py:73` contains an extra condition: `elif days_since_last == 1 and today.weekday() != 6:`. In Python's `datetime`, `weekday() == 6` is Sunday. This guard prevents the streak from incrementing whenever today is Sunday — so a user who listens Saturday and then Sunday (one consecutive day apart) falls through to the `else` branch and resets to 1. The Sunday guard has no valid business reason and is never documented; it is a latent bug in the condition.

**How I reproduced it:** Wrote a direct Python script (the live API couldn't be used without time manipulation since bugs only appear when "today" is a specific weekday). Set a test user's `last_listened_at` to Saturday 2026-07-04 and `listening_streak` to 5, then called `update_listening_streak(user, sunday)` with `sunday = datetime(2026, 7, 5, tzinfo=UTC)` (weekday=6). Result: streak went from 5 → 1 instead of 5 → 6.

**Fix:** Removed the Sunday guard from the condition.
```python
# Before
elif days_since_last == 1 and today.weekday() != 6:
# After
elif days_since_last == 1:
```

**Verification:** `pytest tests/test_streaks.py` — 5/5 passing, including `test_streak_increments_on_sunday` which was FAILING before the fix. All other streak tests (new user, consecutive day, same day, skipped day) continue to pass.

---

### Issue 2 — Friends Listening Now shows people from yesterday

**Symptom:** The Friends Listening Now feed included friends who had listened many hours ago — sometimes the previous day — rather than showing only people currently active.

**Root cause:** `feed_service.py:13` sets `RECENT_THRESHOLD = timedelta(hours=24)`. This cutoff is far too wide for a "listening now" feature. Any `ListeningEvent` within the past 24 hours qualifies, meaning someone who listened at 11pm yesterday still shows as "listening now" the following afternoon.

**How I reproduced it:** Queried `GET /feed/<kenji_id>/listening-now`. Kenji's friend nova had a listening event seeded 2 hours ago with no more recent event. The response returned nova as listening now with `listened_at` 2.2 hours in the past. With any reasonable "now" window (e.g., 30 minutes), nova should not appear.

**Fix:** Changed the threshold to 30 minutes.
```python
# Before
RECENT_THRESHOLD = timedelta(hours=24)
# After
RECENT_THRESHOLD = timedelta(minutes=30)
```

**Verification:** After restarting the server, `GET /feed/<kenji_id>/listening-now` returned `count: 0` — nova's 2.2-hour-old event was excluded. Nova's own feed still returned 3 friends whose events were within 30 minutes (seeded at 10–20 minutes ago).

---

### Issue 3 — The same song keeps showing up twice in search

**Symptom:** Searching for a song would occasionally return it multiple times in the result list.

**Root cause:** `search_service.py:26–35` does `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` without `.distinct()`. The outer join produces one row per tag per song. A song with 3 tags generates 3 rows in the join result, which should cause it to appear 3 times. In the current environment (SQLAlchemy 2.0), the ORM deduplicates by primary key identity when returning mapped objects, so the duplication is masked at the Python level. However, the raw SQL is still wrong — confirmed by running the query directly:

```sql
SELECT song.id, song.title, song_tags.tag_id
FROM song LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
WHERE lower(song.title) LIKE lower("%Crown%")
-- Returns 3 rows for "Crown Heights Anthem" (one per tag)
```

On SQLAlchemy 1.x, or with any query that returns raw dicts rather than ORM objects, all 3 duplicates would appear.

**How I reproduced it:** Ran the raw SQL query above directly against the database. It returned 3 rows for "Crown Heights Anthem" (tags: rap, hip-hop, boom bap). The API search and pytest tests did not surface the issue in the current setup due to ORM deduplication — I documented this discrepancy rather than just accepting the symptom at face value.

**Fix:** Added `.distinct()` to the ORM query before `.all()`.
```python
.distinct()
.all()
```

**Verification:** All 5 search tests passing. The raw SQL query with `SELECT DISTINCT` now returns 1 row for "Crown Heights Anthem". The fix is correct regardless of the SQLAlchemy version.

---

### Issue 4 — I got notified when a friend added my song to a playlist but not when they rated it

**Symptom:** Users received notifications when friends added their songs to playlists, but received nothing when friends rated those songs.

**Root cause:** `notification_service.py:rate_song()` saves the `Rating` and calls `db.session.commit()`, then returns — it never calls `create_notification()`. By contrast, `add_to_playlist()` in the same file demonstrates the correct pattern: after the write, it checks `song.shared_by != added_by_user_id` and fires a notification to the original sharer. The `rate_song()` function was implemented without this final step.

**How I reproduced it:**
1. Checked the song sharer's (simone's) notifications before the test: count = 0.
2. Posted `POST /songs/<song_id>/rate` with another user's ID (nova) and score 5 — HTTP 201, rating saved.
3. Checked simone's notifications again: still count = 0.

**Fix:** Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, guarded by `song.shared_by != user_id` to avoid self-notification.
```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

**Verification:** After restarting the server, posted the same rating request. Simone's notification count went from 0 → 1, with body: `"nova rated your song 'Crown Heights Anthem' 4/5."`. Also verified that a user rating their own song does not generate a notification (the guard prevents self-notification).

---

### Issue 5 — The last song in a playlist never shows up

**Symptom:** Playlists consistently showed one fewer song than expected. Users couldn't see their final track.

**Root cause:** `playlist_service.py:66` returns `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` Python slice drops the last element from every list. The query correctly fetches all songs ordered by `position` ascending, but the return statement then discards the song with the highest position number on every call. This is a straightforward off-by-one error in the return value — the query itself is correct.

**How I reproduced it:** Queried `GET /playlists/<late_night_vibes_id>/songs`. Confirmed via direct SQL that `playlist_entries` had 7 rows for that playlist (positions 1–7). The API response returned `count: 6` and the song at position 7 ("Free Throws" by Hoop Dreams) was absent.

**Fix:** Removed the `[:-1]` slice.
```python
# Before
return [song.to_dict() for song in songs[:-1]]
# After
return [song.to_dict() for song in songs]
```

**Verification:** `pytest tests/test_playlists.py` — 3/3 passing. `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` were both FAILING before the fix and pass after. The API now returns `count: 7` for "Late Night Vibes".

---

## Commit History

```
7c9fa45 docs: add bug reproduction log and fix log to submission.md
d7ba164 fix: send notification to song sharer when a friend rates their song
53d8f46 fix: add DISTINCT to search query to prevent duplicate results for multi-tag songs
f584395 fix: reduce Friends Listening Now threshold from 24 hours to 30 minutes
60f3b3f fix: correct Sunday boundary condition in streak reset logic
78bb061 fix: remove off-by-one slice that dropped last playlist song
```

All five bugs fixed. Final test run: **13/13 passing**.
