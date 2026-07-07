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
