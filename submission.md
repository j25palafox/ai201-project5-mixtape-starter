# Mixtape Bug Hunt Submission

## AI Usage

I used AI during the orientation phase to help summarize files, explain unfamiliar functions, and trace call chains through the app. I verified the explanation by reading the code myself and comparing it to the actual routes, services, models, and tests.

---

## App Overview

Mixtape is a social music app where users can share songs, rate songs, build playlists, receive notifications, view friend activity, and track listening streaks.

The app follows a route/service/model structure:

```text
HTTP request
→ route file in routes/
→ service function in services/
→ SQLAlchemy models/database
→ JSON response
```

---

## Codebase Map

## Main Files and Folders

#### `app.py`

`app.py` is the Flask application entry point. The project is started with:

```bash
FLASK_APP=app:create_app flask run
```

This means the app uses a Flask application factory named `create_app()`. Its role is to create and configure the Flask app, initialize shared app infrastructure such as the database, and register the route files so the endpoints are available.

#### `models.py`

`models.py` defines the SQLAlchemy models and association tables used by the app.

The file imports `db` from `app.py`, uses UUID strings for primary keys, and defines a helper function called `generate_uuid()` for creating IDs. It also defines three association tables before the model classes: `friendships`, `song_tags`, and `playlist_entries`.

`friendships` is a many-to-many self-relationship between users. This supports users having friends.

`song_tags` is a many-to-many relationship between songs and tags.

`playlist_entries` is a many-to-many relationship between playlists and songs, but it stores extra information: `position`, `added_by`, and `added_at`. This means songs in a playlist have explicit ordering and metadata about who added them, not just membership in the playlist. That table will matter when tracing playlist behavior.

The main model classes are:

- `User`: represents a Mixtape user. It stores `username`, `email`, `listening_streak`, `last_listened_at`, and `created_at`. It also has relationships to shared songs, ratings, listening events, notifications, playlists, and friends.
- `Tag`: represents a tag that can be attached to songs.
- `Song`: represents a song shared by a user. It stores `title`, `artist`, `album`, `genre`, the user who shared it, when it was shared, and an optional `share_note`.
- `ListeningEvent`: represents a user listening to a song at a specific time. It connects `user_id`, `song_id`, and `listened_at`.
- `Rating`: represents a user rating a song. It stores `user_id`, `song_id`, `score`, and `rated_at`. It also has a uniqueness constraint on `user_id` and `song_id`, so one user should only have one rating per song.
- `Playlist`: represents a playlist created by a user. It stores `name`, `created_by`, `created_at`, and whether the playlist `is_collaborative`. Its songs are connected through `playlist_entries`.
- `Notification`: represents a notification shown to a user. It stores the recipient user, notification type, body text, creation time, and read/unread status.

Important model pattern: some features have both event/history records and summary state. For example, individual listens are stored as `ListeningEvent` records, while the current streak summary is stored directly on the `User` model through `listening_streak` and `last_listened_at`.

#### `routes/`

The `routes/` folder contains the HTTP entry points. These files receive requests, read URL parameters or JSON request data, call service functions, and return JSON responses. The route files are organized by feature area.

The route files in this repo are:

- `routes/feed.py`
- `routes/playlists.py`
- `routes/songs.py`
- `routes/users.py`

`routes/feed.py` defines the feed-related endpoints. It creates a `feed_bp` blueprint and exposes two user-specific routes:

- `/<user_id>/listening-now`: calls `get_friends_listening_now(user_id)` from `services.feed_service`.
- `/<user_id>/activity`: calls `get_activity_feed(user_id)` from `services.feed_service`.

Both feed routes return a JSON object with `feed` and `count`, and both return a `404` JSON error if the service raises a `ValueError`.

`routes/playlists.py` defines playlist endpoints through a `playlists_bp` blueprint. It imports playlist service functions including `create_playlist`, `get_playlist_songs`, `get_playlist`, and `get_user_playlists`. It also imports `add_to_playlist` from `services.notification_service`.

The playlist routes are:

- `POST /`: reads `name`, `created_by`, and optional `is_collaborative` from the JSON body, then calls `create_playlist(...)`.
- `GET /<playlist_id>`: calls `get_playlist(playlist_id)`.
- `GET /<playlist_id>/songs`: calls `get_playlist_songs(playlist_id)`.
- `POST /<playlist_id>/songs`: reads `song_id` and `added_by` from the JSON body, then calls `add_to_playlist(playlist_id, song_id, added_by)`.

This file shows that adding a song to a playlist goes through notification-related service logic, not only playlist service logic.

`routes/songs.py` defines song-related endpoints through a `songs_bp` blueprint. It imports `search_songs` and `get_song` from `services.search_service`, `rate_song` from `services.notification_service`, and `record_listening_event` from `services.streak_service`.

The song routes are:

- `GET /search`: reads the query parameter `q`, calls `search_songs(query)`, and returns matching results with a count.
- `GET /<song_id>`: calls `get_song(song_id)`.
- `POST /<song_id>/rate`: reads `user_id` and `score` from the JSON body, then calls `rate_song(user_id, song_id, int(score))`.
- `POST /<song_id>/listen`: reads `user_id` from the JSON body, then calls `record_listening_event(user_id, song_id)`.

This file shows two cross-feature flows: rating a song enters through the song route but uses notification service logic, while listening to a song enters through the song route but uses streak service logic.

`routes/users.py` defines user-related endpoints through a `users_bp` blueprint. It directly uses `db.session.get(User, user_id)` for the basic user lookup route, and delegates streak and notification behavior to services.

The user routes are:

- `GET /<user_id>`: looks up a `User` directly through the database session and returns `user.to_dict()`.
- `GET /<user_id>/streak`: calls `get_streak(user_id)` from `services.streak_service`.
- `GET /<user_id>/notifications`: reads an optional `unread_only` query parameter, then calls `get_notifications(user_id, unread_only=unread_only)` from `services.notification_service`.
- `POST /notifications/<notification_id>/read`: calls `mark_as_read(notification_id)` from `services.notification_service`.

Pattern I noticed: most routes are thin controllers. They validate required request data, call one service function, and return JSON. A few routes, such as `GET /<user_id>` in `routes/users.py`, query the model/database directly, but most feature behavior is delegated into the service layer.

#### `services/`

The `services/` folder contains the app’s business logic. These files decide what records to query, create, update, or return after a route receives a request.

The service files in this repo are:

- `services/feed_service.py`
- `services/notification_service.py`
- `services/playlist_service.py`
- `services/search_service.py`
- `services/streak_service.py`

`services/feed_service.py` handles friend listening activity. It defines `get_friends_listening_now(user_id)`, which finds the current user, gets that user’s friends, filters recent `ListeningEvent` records within a 24-hour threshold, and returns the most recent song per friend. It also defines `get_activity_feed(user_id, limit=20)`, which returns recent listening events from friends without the same recency filter. Both functions return dictionaries containing friend data, song data, and `listened_at` timestamps.

`services/notification_service.py` handles notification creation, retrieval, and some cross-feature actions that should generate notifications. It defines `create_notification(user_id, notification_type, body)`, which creates a `Notification` record and commits it. It also defines `add_to_playlist(playlist_id, song_id, added_by_user_id)`, which adds a song to a playlist and notifies the song’s original sharer if someone else added it. The same file defines `rate_song(user_id, song_id, score)`, which validates the score, finds the song and rater, creates or updates a `Rating`, and commits it. It also provides `get_notifications(user_id, unread_only=False)` and `mark_as_read(notification_id)`.

`services/playlist_service.py` handles playlist creation and retrieval. It defines `create_playlist(name, created_by_user_id, is_collaborative=True)`, which validates that the creator exists, creates a `Playlist`, and commits it. It defines `get_playlist_songs(playlist_id)`, which looks up the playlist, joins `Song` through `playlist_entries`, orders songs by `playlist_entries.position`, and returns song dictionaries. It also defines `get_playlist(playlist_id)` for playlist metadata and `get_user_playlists(user_id)` for all playlists created by a user.

`services/search_service.py` handles song lookup and search. It defines `search_songs(query)`, which searches `Song.title` and `Song.artist` using case-insensitive matching and returns song dictionaries. It also defines `get_song(song_id)`, which returns one song by ID or raises `ValueError` if the song does not exist.

`services/streak_service.py` handles listening events and streak updates. It defines `record_listening_event(user_id, song_id)`, which validates the user, creates a `ListeningEvent`, calls `update_listening_streak(user, now)`, commits the database session, and returns the event. `update_listening_streak(user, now)` applies the streak rules: first listen starts at 1, another listen on the same day does not change the streak, listening on the next day increments the streak, and skipped days reset it to 1. `get_streak(user_id)` returns the user’s current `listening_streak`.

Pattern I noticed: services are mostly responsible for business rules, database reads/writes, and error handling through `ValueError`. Routes catch those `ValueError`s and turn them into JSON error responses. Some service files are feature-specific, like `search_service.py` and `streak_service.py`, but `notification_service.py` is cross-feature because both playlist actions and song rating actions pass through it.

#### `seed_data.py`

`seed_data.py` populates the local database with realistic test data. It uses `create_app()` from `app.py`, opens an app context, drops and recreates all tables, then inserts seeded records.

The seed file creates:

- 5 users: `nova`, `darius`, `simone`, `kenji`, and `aaliya`
- bidirectional friendships between several users
- 10 tags, including `rap`, `hip-hop`, `r&b`, `indie`, `lo-fi`, `jazz`, `soul`, `electronic`, and `afrobeats`
- songs with different tag counts: some with no tags, some with one tag, and some with three or more tags
- listening events, including recent events within the past 30 minutes and older events from hours or days ago
- existing listening streak values and `last_listened_at` values for some users
- 3 playlists: `Late Night Vibes`, `Friday Energy`, and `Study Mode`
- playlist entries with explicit `position`, `added_by`, and `added_at` values
- an existing `song_added_to_playlist` notification

#### `tests/`

The `tests/` folder contains pytest tests that exercise the service layer directly using an in-memory SQLite database. Each test file creates a test Flask app with `TESTING=True`, calls `db.create_all()` before the test, and drops the tables afterward.

The test files in this repo are:

- `tests/test_playlists.py`
- `tests/test_search.py`
- `tests/test_streaks.py`

`tests/test_playlists.py` tests playlist retrieval through `get_playlist_songs()`. It creates a playlist with 5 songs, inserts rows into `playlist_entries` with positions `1` through `5`, then checks that `get_playlist_songs()` returns all 5 songs and preserves the order `Track 1` through `Track 5`. It also checks that an empty playlist returns an empty list.

`tests/test_search.py` tests `search_songs()`. It creates songs with no tags, one tag, and three tags. The tests verify that matching songs are returned, that no-match searches return an empty list, and that songs with no tags, one tag, or multiple tags each appear exactly once in the results.

`tests/test_streaks.py` tests listening streak behavior through `update_listening_streak()`. It checks that a new user starts with a streak of `1`, consecutive-day listening increments the streak, two listens on the same day do not double-count, skipping a day resets the streak, and listening on Saturday then Sunday should increment rather than reset.

These tests line up with several reported bug areas: playlists, search, and streaks. Running `pytest tests/` is useful for reproducing current failures and verifying that fixes do not break related behavior.

---

### Data Flow: User Rates a Song

When a user rates a song, the request enters through `routes/songs.py` at `POST /songs/<song_id>/rate`.

The route function `rate(song_id)` reads the song ID from the URL, then parses `user_id` and `score` from the JSON request body. If either value is missing, the route returns a 400 error response.

If the request has the required fields, the route calls `rate_song(user_id, song_id, int(score))` from `services/notification_service.py`.

In `notification_service.py`, `rate_song()` validates that the score is between 1 and 5. It then looks up the `Song` being rated and the `User` submitting the rating. If either record does not exist, the service raises a `ValueError`, which the route turns into a 400 response.

If both records exist, the service checks whether that user has already rated the song. If an existing `Rating` record is found, the service updates its score. If no rating exists yet, the service creates a new `Rating(user_id=user_id, song_id=song_id, score=score)` record and adds it to the database session.

The rating is stored in the `Rating` model, not directly on the `Song`. After creating or updating the rating, the service commits the database change and returns the `Rating` object. The route then returns `rating.to_dict()` as JSON.

Pattern I noticed: the route handles URL input, JSON request parsing, response formatting, and HTTP error codes. The service handles validation, database lookups, create/update logic, and commits.

---

### Data Flow: Viewing Songs in a Playlist

When a client requests the songs in a playlist, the request enters through `routes/playlists.py` at `GET /playlists/<playlist_id>/songs`.

The route function `get_songs(playlist_id)` reads the playlist ID from the URL, calls `get_playlist_songs(playlist_id)` from `services/playlist_service.py`, and returns the result as JSON with both the song list and a count.

In `playlist_service.py`, `get_playlist_songs()` first checks whether the playlist exists. If it does not, it raises a `ValueError`, which the route turns into a 404 response.

If the playlist exists, the service queries `Song`, joins through `playlist_entries`, filters to the requested playlist ID, and orders the songs by `playlist_entries.c.position` in ascending order. This matters because playlist membership is ordered data: the association table does not just connect playlists to songs; it also stores each song’s position.

The service returns each song as a dictionary using `song.to_dict()`, and the route wraps that list in a JSON response.

Pattern I noticed: the route handles URL input, JSON response formatting, and HTTP error status codes. The service handles database validation, joins, ordering, and model-to-dictionary conversion.

---

### Patterns I Noticed

- Routes act as thin controllers: they receive HTTP requests, parse inputs, delegate to services, and return JSON responses.
- Service files contain most of the business logic.
- Models define the shared database structure used across routes and services.
- The app uses many-to-many association tables for friendships, song tags, and playlist entries.
- `playlist_entries` is especially important because it stores `position`, `added_by`, and `added_at`, so playlist behavior depends on more than just the `Playlist` and `Song` models.
- Some features store both history and summary state. For example, listening history is stored in `ListeningEvent`, while the current streak is stored on `User`.
- Notification behavior is cross-feature. A song rating, playlist add, or other user action may need to create a `Notification` record.
- Bugs are likely best investigated by starting from the route that reproduces the symptom, then following the service calls and model relationships.

---

## Issues

## Bug fix 1: Issue 5 - The last song in a playlist never shows up

### 2. How you reproduced it

I reproduced this bug using the seeded "Late Night Vibes" playlist.

First, I used `flask shell` to inspect the database directly. I imported the `Playlist` model, listed the seeded playlists, and loaded the playlist with id `2eb1ddaa-5783-4daf-807a-2dfa49e3797d`.

In `flask shell`, the playlist's `songs` relationship contained 7 songs:

- Golden Hour
- Block Party
- Midnight Drive
- Late Night Session
- Still Waters
- Free Throws
- First Light

Then I requested the user-facing playlist songs endpoint with curl:

`GET /playlists/2eb1ddaa-5783-4daf-807a-2dfa49e3797d/songs`

The API response returned `"count": 6` and did not include `Free Throws`. This confirmed that the database contained 7 songs for the playlist, but the application response only returned 6.

### 3. How you found the root cause

I started from the user-facing endpoint for playlist songs. In `routes/playlists.py`, the `GET /playlists/<playlist_id>/songs` route calls `get_playlist_songs(playlist_id)`.

I followed that call into `services/playlist_service.py`. I became confident the bug was in `get_playlist_songs` because the database relationship itself contained all 7 songs, but the API response only returned 6. That meant the data existed correctly and was being dropped while the service built the response.

### 4. The root cause

The root cause was an off-by-one slicing error in `services/playlist_service.py`, inside `get_playlist_songs`.

The function correctly queried all songs for the playlist, joined through `playlist_entries`, filtered by `playlist_id`, and ordered the songs by `playlist_entries.c.position`. However, the return line used `songs[:-1]`:

`return [song.to_dict() for song in songs[:-1]]`

In Python, `songs[:-1]` returns every element except the final one. That meant the database query retrieved all playlist songs, but the service dropped the last song before returning the response to the route. This is why the database showed 7 songs for "Late Night Vibes" while the API response only returned 6.

### 5. Your fix and side-effect check

I changed the return line in `get_playlist_songs` from iterating over `songs[:-1]` to iterating over `songs`:

`return [song.to_dict() for song in songs]`

This fixes the bug because the service now converts every queried `Song` object into a dictionary instead of excluding the final item.

To check for side effects, I re-ran the same curl request for the seeded "Late Night Vibes" playlist and confirmed that the endpoint returned all 7 songs instead of 6. I also ran `pytest tests/test_playlists.py -vv` to verify the playlist tests still passed and that the playlist response behavior was not broken by the change.

---
