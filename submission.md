## Codebase Map

### Entry Point -- **`app.py`** 

### Data Models (`models.py`)

- `User`: It is a user account table holding `listening_streak` and `last_listened_at`.
- `Song`: It is a shared song table holding `shared_by`, which is a foreign key to a row in `User`.
- `ListeningEvent`: It is a play history table holding `user_id`, `song_id`, and `listened_at`. The feed is built by querying this table.
- `Rating`: It is a ratings table holding a user's 1–5 score for a song, with a unique constraint on `(user_id, song_id)` so a user can only rate a song once.
- `Playlist`: It is a named list table holding `created_by` (FK to `User`) and an `is_collaborative` flag.
- `Notification`: It is an in-app notification table holding a `type` string, `body` text, `read` boolean, and a FK to the recipient `User`.

Association tables:
- `friendships`: It is a many-to-many on `User` that uses explicit `primaryjoin`/`secondaryjoin` so that `user.friends` returns the correct set.
- `song_tags`: It is a many-to-many between `Song` and `Tag`.
- `playlist_entries`: It is a many-to-many between `Playlist` and `Song` with an extra `position` integer column for ordering and an `added_by` FK.

### Routes (`routes/`)

- `routes/songs.py`: It handles `/songs` routes for searching songs, getting a single song, submitting a rating, and recording a listen event.
- `routes/playlists.py`: It handles `/playlists` routes for creating playlists, retrieving playlist metadata, and getting/adding songs.
- `routes/users.py`: It handles `/users` routes for getting a user profile, getting the current streak, and getting/marking notifications.
- `routes/feed.py`: It handles `/feed` routes for the "Friends listening now" feed and the general activity feed.

### Services (`services/`)

- `streak_service.py`: It records a listen event and updates the user's streak based on how many days have passed since their last listen.
- `feed_service.py`: It queries recent `ListeningEvent` rows from a user's friends to build the "listening now" feed and the general activity feed.
- `search_service.py`: It searches songs by title or artist using a case insensitive filter.
- `notification_service.py`: It creates notifications and handles adding songs to playlists and rating songs.
- `playlist_service.py`: It creates playlists and retrieves a playlist's songs ordered by their position.


## Data Flow — User Rates a Song

- `POST /songs/<song_id>/rate` receives `user_id` and `score` in the request body.
- `routes/songs.py: rate()` validates that both fields are present and casts score to an int.
- It calls `notification_service.rate_song(user_id, song_id, score)`.
- `rate_song()` validates the score is between 1 and 5, fetches the Song and User, and upserts a Rating row.
- It commits to the database and returns the Rating, which the route serializes with `.to_dict()` and returns as a 201.


## Patterns in the Codebase

- Routes are thin — every route just parses the request, calls one service function, and returns the result. All the real logic is in the services.
- Services own validation — every service checks its inputs first and raises a `ValueError` if something is wrong. Routes catch that and return a 4xx error.

---

## Bug Inventory

- Bug 1 (`streak_service.py`): The streak never increments on Mondays because of an extra `today.weekday() != 6` check that blocks the Sunday-to-Monday transition.
- Bug 2 (`feed_service.py`): The "listening now" window is 24 hours, so people who listened yesterday still show up as currently listening.
- Bug 3 (`search_service.py`): A song with multiple tags appears multiple times in search results because the join with `song_tags` produces one row per tag.

## Bug Detection

- Bug 1 (`streak_service.py`): It was reproduced by setting a user's `last_listened_at` to a Saturday and calling `update_listening_streak` with a Sunday datetime. The streak reset to 1 instead of incrementing to 6, confirming the `today.weekday() != 6` condition blocks the increment whenever today is Sunday.
- Bug 2 (`feed_service.py`): It was reproduced by calling `GET /feed/<user_id>/listening-now` against the seeded database, which contains friends with listening events from 10–18 hours ago. Those friends appeared in the "listening now" response even though they had not listened recently, confirming the 24-hour threshold is too wide.
- Bug 3 (`search_service.py`): It was reproduced by calling `GET /songs/search?q=crown` against the seeded database, which contains "Crown Heights Anthem" with 3 tags. The song appeared 3 times in the results, confirming the `outerjoin` on `song_tags` produces one row per tag for songs with multiple tags.

---

## Root Cause Analysis

### Bug 1 — Streak does not increment on Sundays

**How I reproduced it:**
Set a user's `last_listened_at` to Saturday July 4 and called `update_listening_streak` directly with a Sunday July 5 datetime (weekday 6). The streak was 5 before the call and reset to 1 instead of incrementing to 6.

**How I found the root cause:**
Started at the route `POST /songs/<song_id>/listen` in `routes/songs.py`. The route calls `record_listening_event(user_id, song_id)` from `streak_service.py`. Inside `record_listening_event`, the only streak-related call is `update_listening_streak(user, now)` on line 36. Reading `update_listening_streak` in `streak_service.py`, the increment branch on line 73 had an extra condition: `today.weekday() != 6`. That was the only conditional guarding the increment and it was the only place the streak could be blocked while `days_since_last == 1`.

**The root cause:**
Python's `datetime.weekday()` returns 6 for Sunday. The condition `days_since_last == 1 and today.weekday() != 6` means the streak only increments if the user listened yesterday AND today is not Sunday. When a user listens on Saturday and again on Sunday, `days_since_last == 1` is true but `today.weekday() != 6` is false, so the condition fails and the streak resets to 1. The extra weekday check has no valid purpose — consecutive-day streak logic should apply equally on every day of the week.

**The fix and side-effect check:**
Removed the `and today.weekday() != 6` clause, leaving `elif days_since_last == 1:` as the only condition for incrementing. The other two branches (`days_since_last == 0` and the else reset) are unaffected. Verified that a Saturday-to-Sunday transition now increments correctly and that the `days_since_last == 0` (already listened today) and reset (gap > 1 day) paths still behave correctly with the same test inputs.

---

### Bug 2 — "Listening now" feed shows friends who listened up to 24 hours ago

**How I reproduced it:**
Called `GET /feed/<nova-id>/listening-now` against the seeded database. Nova's friends had listening events from 10–18 hours ago. Those friends appeared in the response even though they had not listened recently, confirming the recency window was too wide.

**How I found the root cause:**
Started at the route `GET /feed/<user_id>/listening-now` in `routes/feed.py`. The route calls `get_friends_listening_now(user_id)` from `feed_service.py`. At the top of `feed_service.py`, the module-level constant `RECENT_THRESHOLD = timedelta(hours=24)` is used to compute the cutoff: `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`. That single constant controls the entire window and was the only value to check.

**The root cause:**
`RECENT_THRESHOLD` was set to `timedelta(hours=24)`, making the cutoff 24 hours in the past. Any friend who listened within the last day would appear as "listening now", including people who listened 18 hours ago. The feature is meant to show who is actively listening at this moment, so the threshold should be a short window of around 30 minutes.

**The fix and side-effect check:**
Changed `RECENT_THRESHOLD = timedelta(hours=24)` to `RECENT_THRESHOLD = timedelta(minutes=30)`. The constant is only used in `get_friends_listening_now` — `get_activity_feed` has no recency filter by design, so it is unaffected. Verified that friends with events older than 30 minutes no longer appear and that friends with events within the last 30 minutes still do.

---

### Bug 3 — Songs with multiple tags appear multiple times in search results

**How I reproduced it:**
Called `GET /songs/search?q=crown` against the seeded database. "Crown Heights Anthem" has 3 tags (rap, hip-hop, boom bap) and appeared 3 times in the results. Searching for a song with 1 tag returned it once, and a song with no tags also returned once, confirming the duplicate count matched the number of tags.

**How I found the root cause:**
Started at the route `GET /songs/search` in `routes/songs.py`, which calls `search_songs(query)` from `search_service.py`. In `search_songs`, the query does an `outerjoin` on the `song_tags` association table before filtering by title and artist. An outer join on a many-to-many table produces one row per related row — so a song with 3 tags produces 3 rows in the result set, each becoming a separate entry in the returned list.

**The root cause:**
The `outerjoin(song_tags, Song.id == song_tags.c.song_id)` is not used anywhere in the filter — the filter only checks `Song.title` and `Song.artist`. The join was added unnecessarily, and because `song_tags` is a many-to-many table, SQLAlchemy returns one `Song` row per matching tag row. A song with 3 tags is returned 3 times. The tags are already available on each `Song` object via the `Song.tags` relationship, which `to_dict()` uses directly.

**The fix and side-effect check:**
Removed the `outerjoin` call entirely, leaving the query as a plain filter on `Song`. The filter logic and the tag loading in `to_dict()` are both unaffected. Verified that `GET /songs/search?q=crown` now returns "Crown Heights Anthem" exactly once with all 3 tags present, and that songs with 0 or 1 tags also return correctly.
