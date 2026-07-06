# Mixtape Bug Hunt — Submission

## Codebase Map

### Main files and roles

| File / directory | Role |
|------------------|------|
| `app.py` | Flask app factory; initializes SQLAlchemy (`db`), registers route blueprints, creates tables on startup |
| `models.py` | SQLAlchemy models (`User`, `Song`, `Playlist`, `ListeningEvent`, `Rating`, `Notification`, `Tag`) and association tables (`friendships`, `song_tags`, `playlist_entries`) |
| `routes/` | HTTP layer — parses requests, calls services, returns JSON |
| `services/` | Business logic — where all five bugs live |
| `seed_data.py` | Populates the SQLite DB with users, songs, playlists, and sample events |
| `tests/` | Pytest suite covering streaks, search, playlists, and feed |

### Feature data flow: rating a song

1. Client sends `POST /songs/<song_id>/rate` with `{ "user_id", "score" }`
2. `routes/songs.py` validates input and calls `notification_service.rate_song()`
3. `rate_song()` loads the `Song` and `User`, upserts a `Rating` row, commits
4. If the rater is not the original sharer, `create_notification()` notifies `song.shared_by`
5. Client can fetch notifications via `GET /users/<user_id>/notifications`

### Layering

```
routes/  →  services/  →  models.py + app.db
```

Routes should stay thin; persistence and rules live in services.

---

## Bug Fixes (5/5)

### Issue #1 — Listening streak keeps resetting

| Field | Detail |
|-------|--------|
| **Symptom** | Users who listen on consecutive days lose their streak when the second day is Sunday |
| **Reproduction** | Call `update_listening_streak` on Saturday, then again on Sunday; streak resets to 1 instead of incrementing to 2 |
| **Root cause** | `update_listening_streak` had `today.weekday() != 6` on the consecutive-day branch, so Sunday listens fell through to the reset branch |
| **Fix** | Removed the Sunday check; consecutive calendar days always increment regardless of weekday |
| **Commit** | `fix: remove Sunday boundary that incorrectly resets listening streak` |

### Issue #2 — Friends Listening Now shows people from yesterday

| Field | Detail |
|-------|--------|
| **Symptom** | The "listening now" feed includes friends who listened hours ago (e.g. yesterday evening) |
| **Reproduction** | Seed the DB, call `GET /feed/<user_id>/listening-now`; friends with events 2–18 hours old appear alongside truly recent listeners |
| **Root cause** | `RECENT_THRESHOLD` was `timedelta(hours=24)`, far too wide for a "listening now" feature |
| **Fix** | Changed threshold to `timedelta(minutes=30)` to match seed-data intent and real-time semantics |
| **Commit** | `fix: narrow Friends Listening Now window to exclude stale events` |
| **Regression test** | `tests/test_feed.py::test_listening_now_excludes_stale_events` |

### Issue #3 — Same song appears twice (or more) in search

| Field | Detail |
|-------|--------|
| **Symptom** | Searching for a multi-tag song returns duplicate rows (one per tag) |
| **Reproduction** | Search for "Crown Heights"; song with 3 tags appears 3 times |
| **Root cause** | `search_songs` uses `outerjoin(song_tags)` without deduplication; each tag row multiplies the result set |
| **Fix** | Added `.distinct()` to the query before `.all()` |
| **Commit** | `fix: deduplicate search results when songs have multiple tags` |

### Issue #4 — No notification when a friend rates your song

| Field | Detail |
|-------|--------|
| **Symptom** | Sharers get notified when a friend adds their song to a playlist, but not when a friend rates it |
| **Reproduction** | User A shares a song; User B rates it; User A's notifications list has no `song_rated` entry |
| **Root cause** | `rate_song()` saved the rating but never called `create_notification()`, unlike `add_to_playlist()` which notifies `song.shared_by` |
| **Fix** | After commit, if `song.shared_by != user_id`, create a `song_rated` notification mirroring the playlist pattern |
| **Commit** | `fix: notify song sharer when a friend rates their song` |

### Issue #5 — Last song in a playlist never shows up

| Field | Detail |
|-------|--------|
| **Symptom** | A playlist with N songs returns only N−1 in `GET /playlists/<id>/songs` |
| **Reproduction** | Create a playlist with 5 songs; `get_playlist_songs` returns 4, missing "Track 5" |
| **Root cause** | Return statement used `songs[:-1]`, slicing off the final element |
| **Fix** | Return the full `songs` list without slicing |
| **Commit** | `fix: return all playlist songs including the last entry` |

---

## AI Tool Disclosure

AI (Cursor) was used to:
- Orient in the codebase and trace import/call chains
- Identify root causes after reading service files and existing tests
- Draft this submission document

All fixes were verified by running `pytest tests/`.
