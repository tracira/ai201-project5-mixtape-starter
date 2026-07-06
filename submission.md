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
4. After the commit, if `song.shared_by != user_id`, a `Notification` is created for the sharer.
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
4. The sorted list is returned as a list of dicts.
5. Route returns `{songs: [...], count: N}`.

### Patterns I noticed

- **Thin routes, fat services** — Every route is ≤10 lines of logic. All decisions and queries live in the service layer.
- **No authentication** — `user_id` is passed in the request body or URL with no session or token check. This is a learning project.
- **UUID primary keys** — All models use `generate_uuid()` as the default PK.
- **Consistent error pattern** — Services raise `ValueError` with a message; routes catch it and return a 400 or 404 JSON error.
- **Association tables with extra columns** — `playlist_entries` adds `position` and `added_by` beyond a plain join, giving songs explicit ordering within a playlist.

---

## Root Cause Analysis

Each entry covers: **Symptom · Navigation strategy · Root cause · Reproduction steps · Fix · Side-effect check**.

---

### Issue 1 — My listening streak keeps resetting

**Symptom:** Users reported their streak resetting to 1 unexpectedly, even after listening on consecutive days.

**Navigation strategy:** Started at `routes/users.py` — the streak endpoint `GET /users/<id>/streak` calls `streak_service.get_streak()`, which just reads `user.listening_streak`. That told me the streak value itself was wrong, not the read. Traced back to where the streak is written: `routes/songs.py:listen()` calls `streak_service.record_listening_event()`. Opened `streak_service.py` and read `record_listening_event()` — it creates the event then calls `update_listening_streak(user, now)`. Read that function line by line. The three branches (same day / consecutive day / else) looked correct in principle, but the `elif` at line 73 had an extra clause: `and today.weekday() != 6`. That was the moment — the condition says "increment only if today is not Sunday," which is wrong for any user listening Saturday → Sunday.

**Root cause:** `streak_service.py:73` contains `elif days_since_last == 1 and today.weekday() != 6:`. In Python's `datetime`, `weekday() == 6` is Sunday. This guard prevents the streak from incrementing whenever today is Sunday, regardless of how many days have passed. A user who listens Saturday and then Sunday (days_since_last == 1) hits the `elif` but the condition evaluates to False on Sunday, so execution falls to the `else` branch and resets the streak to 1. The guard has no documented business justification.

**Reproduction steps:** The bug only manifests when today is a Sunday, which can't be forced via the live API. Wrote a Python script that called `update_listening_streak()` directly with a synthetic Sunday datetime: set `last_listened_at` to Saturday 2026-07-04 and `listening_streak` to 5, then called the function with `now = datetime(2026, 7, 5, tzinfo=UTC)` (weekday=6). Streak reset from 5 → 1 instead of incrementing to 6.

**Fix:** Removed the Sunday guard.
```python
# Before
elif days_since_last == 1 and today.weekday() != 6:
# After
elif days_since_last == 1:
```

**Side-effect check:** After removing the guard, ran all five streak tests to confirm the fix didn't break adjacent behavior: same-day listening still makes no change, a skipped day still resets to 1, and a brand-new user still starts at 1. The only test that changed state was `test_streak_increments_on_sunday` (FAIL → PASS). Also checked manually that Monday–Saturday consecutive listening still increments correctly by running the script again with a Monday synthetic date — streak incremented as expected.

---

### Issue 2 — Friends Listening Now shows people from yesterday

**Symptom:** The Friends Listening Now feed included friends who had listened many hours ago — sometimes the previous day — rather than showing only people currently active.

**Navigation strategy:** Started at `routes/feed.py` — the `GET /feed/<id>/listening-now` endpoint calls `feed_service.get_friends_listening_now()`. Opened `feed_service.py`. The very first thing after the imports is a module-level constant: `RECENT_THRESHOLD = timedelta(hours=24)`. That was immediately suspicious for a "listening now" feature. Scrolled down to confirm the constant was used as the cutoff in the query filter (`ListeningEvent.listened_at >= cutoff`). The query logic itself was correct — only the threshold value was wrong.

**Root cause:** `feed_service.py:13` sets `RECENT_THRESHOLD = timedelta(hours=24)`. Any `ListeningEvent` within the past 24 hours qualifies as "listening now," so a friend who listened at 11pm yesterday still appears on the feed the following afternoon. A "listening now" window should be measured in minutes, not hours.

**Reproduction steps:** Queried `GET /feed/<kenji_id>/listening-now`. Kenji's only friend with a listening event is nova, whose most recent event was seeded 2 hours ago. The response returned nova as "listening now" with `listened_at` 2.2 hours in the past — well outside any reasonable "now" window.

**Fix:** Changed the threshold to 30 minutes.
```python
# Before
RECENT_THRESHOLD = timedelta(hours=24)
# After
RECENT_THRESHOLD = timedelta(minutes=30)
```

**Side-effect check:** After the fix, confirmed two behaviors. First, `GET /feed/<kenji_id>/listening-now` now returns count 0 — nova's 2.2-hour-old event is correctly excluded. Second, queried `GET /feed/<nova_id>/listening-now` — nova's friends (darius, simone, kenji) had events seeded 10–20 minutes ago and all three still appear, confirming the threshold doesn't exclude genuinely recent events. Also checked `GET /feed/<nova_id>/activity` (the activity feed, which has no recency filter) — it was unaffected since it uses no threshold constant.

---

### Issue 3 — The same song keeps showing up twice in search

**Symptom:** Searching for a song would occasionally return it multiple times in the result list.

**Navigation strategy:** Started at `routes/songs.py` — the `GET /songs/search` endpoint calls `search_service.search_songs(query)`. Opened `search_service.py` and read the query. The `ILIKE` filters looked correct. The join caught my eye: `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`. That's joining the songs table to the song_tags association table. The purpose of the join — to include tag data — made no sense here, because `Song.tags` is already loaded via a `lazy="subquery"` relationship and doesn't need to be joined in the search query. More importantly, an outer join to a many-to-many table without `DISTINCT` produces one row per tag per song. Searching for a song with 3 tags would produce 3 rows. That was the moment of confidence.

**Root cause:** `search_service.py:26–35` does `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` without `.distinct()`. The outer join produces one SQL row per tag per song. "Crown Heights Anthem" has 3 tags (rap, hip-hop, boom bap), so the query returns 3 rows for that song. Confirmed by running the raw SQL directly:
```sql
SELECT song.id, song.title, song_tags.tag_id
FROM song LEFT OUTER JOIN song_tags ON song.id = song_tags.song_id
WHERE lower(song.title) LIKE lower("%Crown%")
-- Returns 3 rows for one song
```
In SQLAlchemy 2.0, the ORM deduplicates mapped objects by primary key, so the Python result hides the duplicates. On SQLAlchemy 1.x or with dict-returning queries, all 3 copies would appear.

**Reproduction steps:** Ran the raw SQL query above directly against the database — confirmed 3 rows returned for "Crown Heights Anthem." The API and pytest tests did not surface visible duplicates due to ORM deduplication in SQLAlchemy 2.0. The bug is real at the query level regardless of that masking behavior.

**Fix:** Added `.distinct()` before `.all()`.
```python
.distinct()
.all()
```

**Side-effect check:** After the fix, ran all 5 search tests — all passing, including `test_search_no_duplicates_multi_tag_song`. Verified that songs with zero tags (no rows in song_tags) still appear in results — the outer join with `DISTINCT` preserves them because `LEFT OUTER JOIN` keeps rows with no matching tag. Also confirmed `GET /songs/<id>` (single song detail, a separate code path that doesn't use this query) was unaffected.

---

### Issue 4 — I got notified when a friend added my song to a playlist but not when they rated it

**Symptom:** Users received notifications when friends added their songs to playlists, but received nothing when friends rated those songs.

**Navigation strategy:** The issue described an asymmetry between two interactions, which pointed to `notification_service.py` since that's where both live. Opened the file and read both functions side by side. `add_to_playlist()` (line 35): writes to the database, then calls `create_notification()` for the song's sharer. `rate_song()` (line 73): writes to the database, calls `db.session.commit()`, then returns — no `create_notification()` call. The asymmetry was immediately visible by comparing the two function endings. No need to trace through routes at all; the README call chain (`POST /songs/<id>/rate` → `notification_service.rate_song()`) confirmed I was in the right file.

**Root cause:** `notification_service.py:rate_song()` saves the `Rating` and commits but never calls `create_notification()`. The function was implemented without the notification step that `add_to_playlist()` in the same file includes. The two functions handle parallel interactions (adding to playlist vs. rating) but only one was wired up to send a notification.

**Reproduction steps:**
1. Checked simone's notifications: `GET /users/<simone_id>/notifications` → count: 0.
2. Posted `POST /songs/<crown_heights_id>/rate` as nova with score 5 → HTTP 201, rating object returned.
3. Checked simone's notifications again → still count: 0. Rating persisted, no notification row created.

**Fix:** Added a `create_notification()` call after `db.session.commit()` in `rate_song()`, guarded by `song.shared_by != user_id` to prevent self-notification.
```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

**Side-effect check:** After the fix, reposted the same rating request — simone's notification count went from 0 → 1 with the correct body. Checked two specific edge cases: (1) a user rating their own song — notification count stayed 0, confirming the `song.shared_by != user_id` guard works; (2) updating an existing rating (posting a second rating for the same song) — the upsert path still works correctly and generates a second notification, which is expected behavior since the rating changed. Also confirmed `add_to_playlist()` behavior was unchanged by triggering it separately and verifying its notification still fires.

---

### Issue 5 — The last song in a playlist never shows up

**Symptom:** Playlists consistently showed one fewer song than expected. Users couldn't see their final track.

**Navigation strategy:** Started at `routes/playlists.py` — `GET /playlists/<id>/songs` calls `playlist_service.get_playlist_songs()`. Opened `playlist_service.py` and read the function. The query looked correct: joins `Song` to `playlist_entries`, filters by playlist ID, orders by `position` ascending. Scrolled to the return statement and saw `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice was the immediate tell — that drops the last element from any Python list. The query was fine; only the return statement was wrong.

**Root cause:** `playlist_service.py:66` returns `[song.to_dict() for song in songs[:-1]]`. Python's `[:-1]` slice drops the last element from the list. Since songs are sorted by `position` ascending, the dropped element is always the song with the highest position — the final track. The query itself retrieves all songs correctly; the bug is entirely in the return statement.

**Reproduction steps:** Queried `GET /playlists/<late_night_vibes_id>/songs`. Verified via direct SQL on `playlist_entries` that the playlist had 7 rows (positions 1–7). The API returned `count: 6` and the song at position 7 ("Free Throws" by Hoop Dreams) was absent.

**Fix:** Removed the `[:-1]` slice.
```python
# Before
return [song.to_dict() for song in songs[:-1]]
# After
return [song.to_dict() for song in songs]
```

**Side-effect check:** After the fix, queried the same playlist — response now shows `count: 7` and "Free Throws" is present in the correct final position. Ran `pytest tests/test_playlists.py` — 3/3 passing, including `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` (both were FAILING before). Also verified the empty-playlist case: `test_empty_playlist_returns_empty_list` still passes — `[][:-1]` equals `[]`, so the old code happened to be correct for empty playlists, and the fix (`[]` with no slice) is equally correct. The ordering behavior was unchanged since the sort happens in the SQL query, not the slice.

---

## Commit History

```
d4a5d3c docs: restructure submission.md with AI usage section and complete RCA entries
7c9fa45 docs: add bug reproduction log and fix log to submission.md
d7ba164 fix: send notification to song sharer when a friend rates their song
53d8f46 fix: add DISTINCT to search query to prevent duplicate results for multi-tag songs
f584395 fix: reduce Friends Listening Now threshold from 24 hours to 30 minutes
60f3b3f fix: correct Sunday boundary condition in streak reset logic
78bb061 fix: remove off-by-one slice that dropped last playlist song
```

All five bugs fixed. Final test run: **13/13 passing**.
