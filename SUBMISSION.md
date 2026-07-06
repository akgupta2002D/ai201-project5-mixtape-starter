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

## Milestone 2: Bug Reproduction (before any fixes)

Reproduction was done on the `main` branch (unfixed code) using pytest, direct service calls, and seed data. No fix code was written until each bug's behavior was confirmed.

### Chosen bugs

All five issues were reproduced. Minimum of three required; all five documented below.

---

## Bug Fixes (5/5)

### Issue #1 — Listening streak keeps resetting

| Field | Detail |
|-------|--------|
| **Symptom** | Users who listen on consecutive days lose their streak when the second day is Sunday |
| **How I reproduced it** | 1. Checked out `main` (buggy `streak_service.py`). 2. Ran `pytest tests/test_streaks.py::test_streak_increments_on_sunday -v`. 3. Test creates a user, calls `update_listening_streak(user, saturday)` → streak = 1, then `update_listening_streak(user, sunday)` → **expected 2, got 1**. 4. Confirmed the failure only happens when the second listen is on Sunday (`weekday() == 6`); Mon→Tue increments correctly (`test_streak_increments_on_consecutive_day` passes). |
| **Root cause** | `update_listening_streak` had `today.weekday() != 6` on the consecutive-day branch, so Sunday listens fell through to the reset branch |
| **Fix** | Removed the Sunday check; consecutive calendar days always increment regardless of weekday |
| **Commit** | `fix: remove Sunday boundary that incorrectly resets listening streak` |

### Issue #2 — Friends Listening Now shows people from yesterday

| Field | Detail |
|-------|--------|
| **Symptom** | The "listening now" feed includes friends who listened hours ago, not just people listening right now |
| **How I reproduced it** | 1. On `main`, ran `python seed_data.py` then called `get_friends_listening_now(nova.id)` (or used the seeded friendship graph manually). 2. Seed data creates recent events at 10–20 min ago **and** older events at 2–18 hours ago. 3. With the 24-hour `RECENT_THRESHOLD`, both appear in the feed. 4. Minimal repro without seed: created `nova` with friends `darius` (listened 15 min ago) and `simone` (listened 5 hours ago) → `get_friends_listening_now` returned **both** users; only `darius` should qualify for "listening now". |
| **Root cause** | `RECENT_THRESHOLD` was `timedelta(hours=24)`, far too wide for a real-time "listening now" feature |
| **Fix** | Changed threshold to `timedelta(minutes=30)` to match seed-data intent |
| **Commit** | `fix: narrow Friends Listening Now window to exclude stale events` |
| **Regression test** | `tests/test_feed.py::test_listening_now_excludes_stale_events` |

### Issue #3 — Same song appears twice (or more) in search

| Field | Detail |
|-------|--------|
| **Symptom** | Searching can return the same song multiple times — but only for songs with multiple tags |
| **How I reproduced it** | 1. On `main`, created a song with **3 tags** (`rap`, `hip-hop`, `boom bap`) and one with **0 tags**. 2. Ran the search query from `search_songs` directly: `db.session.query(Song).outerjoin(song_tags, ...).filter(title ilike '%Crown Heights%')`. 3. Called `.count()` on that query → **3 rows** for the 3-tag song; same query for a no-tag song → **1 row**. 4. Duplicates are **conditional on tag count**: 0 tags = 1 row, 1 tag = 1 row, 3 tags = 3 SQL rows because `outerjoin(song_tags)` multiplies results per tag. 5. The existing pytest `test_search_no_duplicates_multi_tag_song` documents the expected vs buggy behavior. |
| **Root cause** | `search_songs` uses `outerjoin(song_tags)` without deduplication; each tag association adds another row to the SQL result set |
| **Fix** | Added `.distinct()` to the query before `.all()` |
| **Commit** | `fix: deduplicate search results when songs have multiple tags` |

### Issue #4 — No notification when a friend rates your song

| Field | Detail |
|-------|--------|
| **Symptom** | Sharers get notified when a friend adds their song to a playlist, but not when a friend rates it |
| **How I reproduced it** | 1. On `main`, compared `add_to_playlist()` (calls `create_notification` when `song.shared_by != added_by_user_id`) with `rate_song()` (saves rating, no notification call). 2. Minimal repro: created `nova` (sharer) and `darius` (rater), song owned by `nova`. 3. Called `rate_song(darius.id, song.id, 5)`. 4. Called `get_notifications(nova.id)` → **empty list** (0 notifications). 5. Seed data already includes a working `song_added_to_playlist` notification for `nova`, confirming notifications work — only the rating path is missing. |
| **Root cause** | `rate_song()` saved the rating but never called `create_notification()`, unlike `add_to_playlist()` which notifies `song.shared_by` |
| **Fix** | After commit, if `song.shared_by != user_id`, create a `song_rated` notification mirroring the playlist pattern |
| **Commit** | `fix: notify song sharer when a friend rates their song` |

### Issue #5 — Last song in a playlist never shows up

| Field | Detail |
|-------|--------|
| **Symptom** | A playlist with N songs returns only N−1 in `GET /playlists/<id>/songs` |
| **How I reproduced it** | 1. On `main`, ran `pytest tests/test_playlists.py::test_playlist_returns_all_songs -v`. 2. Test creates a playlist with 5 songs (`Track 1` … `Track 5`) at positions 1–5. 3. `get_playlist_songs(playlist_id)` returned **4 songs**; `Track 5` was missing. 4. `test_playlist_returns_songs_in_order` also fails — titles end at `Track 4`. |
| **Root cause** | Return statement used `songs[:-1]`, slicing off the final element |
| **Fix** | Return the full `songs` list without slicing |
| **Commit** | `fix: return all playlist songs including the last entry` |

---

## AI Tool Disclosure

AI (Cursor) was used to:
- Orient in the codebase and trace import/call chains
- Run reproduction scripts against `main` branch code before documenting fixes
- Draft and refine this submission document

All fixes were verified by running `pytest tests/` (14 passed after fixes).
