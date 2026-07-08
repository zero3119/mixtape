# Codebase Map

## Main Files and Responsibilities

### `app.py`

`app.py` is the main Flask setup file. It creates the Flask application using the `create_app()` factory function, configures the database, initializes SQLAlchemy, registers the route blueprints, and creates the database tables.

The main responsibilities of this file are:

- Create the Flask app.
- Configure the database connection.
- Initialize the shared SQLAlchemy `db` object.
- Register the route groups for songs, playlists, users, and feed.
- Create database tables with `db.create_all()`.

The blueprints are registered with these URL prefixes:

- `routes/songs.py` → `/songs`
- `routes/playlists.py` → `/playlists`
- `routes/users.py` → `/users`
- `routes/feed.py` → `/feed`

This file connects the app together, but it does not contain the main feature logic. The feature logic is mostly in the `services/` folder.

---

### `models.py`

`models.py` defines the database structure using SQLAlchemy. It contains the main database models and the association tables that connect them.

The main models are:

- `User`
- `Song`
- `Tag`
- `ListeningEvent`
- `Rating`
- `Playlist`
- `Notification`

The file also defines three association tables:

- `friendships`: connects users to other users as friends.
- `song_tags`: connects songs to tags.
- `playlist_entries`: connects playlists to songs and also stores extra playlist information.

The `playlist_entries` table is important because it is not just a basic many-to-many table. It also stores:

- the song position in the playlist,
- the user who added the song,
- and the time the song was added.

Most models have a `to_dict()` method. This pattern makes it easier for the routes and services to return JSON responses instead of returning raw SQLAlchemy model objects.

---

### `routes/`

The `routes/` folder contains the Flask API endpoints. These files receive requests, read request data, call service functions, handle errors, and return JSON responses.

The route files usually do not contain the deeper app logic. Instead, they delegate that work to the service files.

---

### `routes/songs.py`

`routes/songs.py` handles song-related endpoints.

It includes routes for:

- searching songs,
- getting one song by ID,
- rating a song,
- and recording that a user listened to a song.

Important endpoints include:

```text
GET /songs/search?q=<query>
GET /songs/<song_id>
POST /songs/<song_id>/rate
POST /songs/<song_id>/listen
```

This route file connects to three service files:

- `search_service.py` for searching songs and getting song details,
- `notification_service.py` for rating songs,
- `streak_service.py` for recording listening events and updating streaks.

---

### `routes/playlists.py`

`routes/playlists.py` handles playlist-related endpoints.

It includes routes for:

- creating a playlist,
- getting playlist metadata,
- getting the songs inside a playlist,
- and adding a song to a playlist.

Important endpoints include:

```text
POST /playlists/
GET /playlists/<playlist_id>
GET /playlists/<playlist_id>/songs
POST /playlists/<playlist_id>/songs
```

This file uses `playlist_service.py` for playlist creation and retrieval. It also uses `notification_service.py` when adding a song to a playlist because that action can create a notification for the person who originally shared the song.

---

### `routes/users.py`

`routes/users.py` handles user-related endpoints.

It includes routes for:

- getting a user profile,
- getting a user's listening streak,
- getting user notifications,
- and marking a notification as read.

Important endpoints include:

```text
GET /users/<user_id>
GET /users/<user_id>/streak
GET /users/<user_id>/notifications
POST /users/notifications/<notification_id>/read
```

This file directly uses the `User` model for basic user lookup. For streaks and notifications, it calls service functions from `streak_service.py` and `notification_service.py`.

---

### `routes/feed.py`

`routes/feed.py` handles the friend feed endpoints.

It includes routes for:

- seeing friends who are listening now,
- and seeing a general friend activity feed.

Important endpoints include:

```text
GET /feed/<user_id>/listening-now
GET /feed/<user_id>/activity
```

Both routes call functions from `feed_service.py`. The route returns the feed data and a count of how many items were returned.

---

## Services and Responsibilities

### `services/search_service.py`

`search_service.py` contains song search logic.

Main functions:

- `search_songs(query)`
- `get_song(song_id)`

`search_songs()` searches for songs where the title or artist contains the query string. The search is case-insensitive. It returns matching songs as dictionaries.

`get_song()` retrieves one song by ID. If the song does not exist, it raises a `ValueError`.

---

### `services/streak_service.py`

`streak_service.py` handles listening streak logic.

Main functions:

- `record_listening_event(user_id, song_id)`
- `update_listening_streak(user, now)`
- `get_streak(user_id)`

`record_listening_event()` records that a user listened to a song. It creates a `ListeningEvent`, updates the user's listening streak, commits the database changes, and returns the new event.

`update_listening_streak()` applies the streak rules:

- If the user has never listened before, the streak starts at 1.
- If the user already listened today, the streak does not change.
- If the user listened yesterday, the streak increases by 1.
- If more than one day has passed, the streak resets to 1.

`get_streak()` returns the current listening streak for a user.

---

### `services/feed_service.py`

`feed_service.py` handles the friend feed logic.

Main functions:

- `get_friends_listening_now(user_id)`
- `get_activity_feed(user_id, limit=20)`

`get_friends_listening_now()` gets the current user's friends, finds listening events from those friends within the last 24 hours, and returns the most recent song per friend.

`get_activity_feed()` gets recent listening events from the user's friends. Unlike `get_friends_listening_now()`, it is not limited to the last 24 hours. It returns the most recent friend activity up to the given limit.

---

### `services/playlist_service.py`

`playlist_service.py` handles playlist creation and retrieval logic.

Main functions:

- `create_playlist(name, created_by_user_id, is_collaborative=True)`
- `get_playlist_songs(playlist_id)`
- `get_playlist(playlist_id)`
- `get_user_playlists(user_id)`

`create_playlist()` checks that the user exists, creates a new playlist, saves it to the database, and returns it.

`get_playlist_songs()` retrieves the songs in a playlist by joining the `Song` table with the `playlist_entries` table. It orders songs by their stored playlist position.

`get_playlist()` returns playlist metadata without the songs.

`get_user_playlists()` returns all playlists created by a specific user.

---

### `services/notification_service.py`

`notification_service.py` handles notification creation and retrieval. It also contains logic for actions that can generate notifications.

Main functions:

- `create_notification(user_id, notification_type, body)`
- `add_to_playlist(playlist_id, song_id, added_by_user_id)`
- `rate_song(user_id, song_id, score)`
- `get_notifications(user_id, unread_only=False)`
- `mark_as_read(notification_id)`

`create_notification()` creates and saves a notification for a user.

`add_to_playlist()` adds a song to a playlist and creates a notification for the original song sharer if another user added their song.

`rate_song()` creates or updates a user's rating for a song. It checks that the score is between 1 and 5, verifies that both the user and song exist, and avoids duplicate ratings by updating an existing rating if one already exists.

`get_notifications()` retrieves notifications for a user, optionally filtering to unread notifications only.

`mark_as_read()` changes a notification's `read` value to `True`.

---

## Example Data Flow 1: User Listens to a Song

Feature: recording a song listen and updating the user's streak.

Endpoint:

```text
POST /songs/<song_id>/listen
```

Step-by-step flow:

1. The request goes to the `listen()` function in `routes/songs.py`.
2. The route reads `user_id` from the JSON request body.
3. If `user_id` is missing, the route returns a `400` error.
4. If `user_id` exists, the route calls `record_listening_event(user_id, song_id)` from `services/streak_service.py`.
5. `record_listening_event()` looks up the user in the database.
6. If the user does not exist, it raises a `ValueError`.
7. If the user exists, it creates a new `ListeningEvent` with the user ID, song ID, and current time.
8. It then calls `update_listening_streak(user, now)`.
9. `update_listening_streak()` compares today's date with the user's `last_listened_at` date.
10. Depending on the date difference, it starts, increments, keeps, or resets the streak.
11. The database changes are committed.
12. The route returns the listening event as JSON.

This flow uses:

```text
routes/songs.py
    → services/streak_service.py
        → models.User
        → models.ListeningEvent
        → database
```

---

## Example Data Flow 2: Adding a Song to a Playlist Creates a Notification

Feature: adding a song to a playlist and notifying the original sharer.

Endpoint:

```text
POST /playlists/<playlist_id>/songs
```

Step-by-step flow:

1. The request goes to the `add_song()` function in `routes/playlists.py`.
2. The route reads `song_id` and `added_by` from the JSON request body.
3. If either value is missing, the route returns a `400` error.
4. If both values exist, the route calls `add_to_playlist(playlist_id, song_id, added_by)` from `services/notification_service.py`.
5. `add_to_playlist()` looks up the song, the user who added it, and the playlist.
6. If any of those records do not exist, it raises a `ValueError`.
7. If the song is not already in the playlist, the song is appended to the playlist's songs relationship.
8. The database is committed.
9. The service checks whether the user who added the song is different from the user who originally shared the song.
10. If they are different, `create_notification()` is called.
11. `create_notification()` creates a notification for the original song sharer.
12. The route returns a success message as JSON.

This flow uses:

```text
routes/playlists.py
    → services/notification_service.py
        → models.Song
        → models.User
        → models.Playlist
        → models.Notification
        → database
```

---

## Patterns I Noticed

The app follows a clear `routes → services → models` structure.

The route files are responsible for:

- defining URLs,
- reading request data,
- checking for missing required fields,
- calling service functions,
- catching errors,
- and returning JSON responses.

The service files are responsible for:

- business logic,
- database queries,
- validation of model records,
- creating or updating records,
- and returning model objects or dictionaries.

The model file is responsible for:

- defining database tables,
- defining relationships between entities,
- and converting model objects into dictionaries with `to_dict()` methods.

A repeated pattern is that service functions check whether required database records exist before continuing. For example, playlist creation checks that the user exists, feed retrieval checks that the user exists, and song lookup checks that the song exists.

Another pattern is that many routes catch `ValueError` from the service layer and turn it into a JSON error response. This keeps the service layer focused on app logic while the route layer handles HTTP responses.

A third pattern is that most database models have a `to_dict()` method. This helps keep JSON formatting consistent across the app.

---

### Bugs

| # | Title | Affected service |
|---|-------|-----------------|
| 1 | My listening streak keeps resetting | `streak_service.py` |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` |
| 3 | The same song keeps showing up twice in search | `search_service.py` |
| 4 | I got notified when a friend added my song to a playlist but not when they rated it | `notification_service.py` |
| 5 | The last song in a playlist never shows up | `playlist_service.py` |


## Bug Fixes

### Issue #1: My listening streak keeps resetting

**Reproduction steps:**  
I reproduced this issue by testing a user whose `last_listened_at` value was on a Saturday and then calling the streak update logic with a Sunday timestamp. Before the fix, the streak reset to 1 instead of increasing by 1, even though the user listened on consecutive calendar days.

**Navigation strategy:**  
I started from the route that records a listen, `POST /songs/<song_id>/listen`, in `routes/songs.py`. That route calls `record_listening_event()` in `services/streak_service.py`. From there, I followed the call to `update_listening_streak(user, now)`, because that function contains the logic for deciding whether the streak stays the same, increments, or resets. I became confident this was the root cause when I saw the condition `days_since_last == 1 and today.weekday() != 6`.

**Root cause explanation:**  
The bug was caused by the condition `today.weekday() != 6` inside `update_listening_streak()`. In Python, `weekday() == 6` represents Sunday. This meant that if a user listened on Saturday and then listened again on Sunday, `days_since_last` was correctly equal to 1, but the Sunday check prevented the streak from incrementing. The code then went to the `else` branch and reset the streak to 1. The correct behavior only needs to check whether the last listen was exactly one calendar day ago.

**Fix description:**  
I removed the Sunday exclusion and changed the logic so that any `days_since_last == 1` increments the streak. I also kept the same behavior for same-day listens and missed-day resets.

**Side-effect check:**  
I ran `pytest tests/test_streaks.py`, and all 5 streak tests passed. I also checked that the same-day case still does not increment the streak and that missing more than one day still resets it to 1.

---

### Issue #2: Friends Listening Now shows people from yesterday

**Reproduction steps:**  
I seeded the database and tested the `/feed/<user_id>/listening-now` endpoint. With the original code, the service used a 24-hour window, so listening events from many hours earlier could still appear in the “Listening Now” feed. I verified the issue by checking that recent events within 30 minutes still appeared, while older events were supposed to be ignored.

**Navigation strategy:**  
I started from `routes/feed.py`, where the `/feed/<user_id>/listening-now` route calls `get_friends_listening_now(user_id)` in `services/feed_service.py`. In that service function, I followed the query that gets the user's friends, filters `ListeningEvent` records, and compares `ListeningEvent.listened_at` against a cutoff time. I became confident I had found the root cause when I saw `RECENT_THRESHOLD = timedelta(hours=24)` being used to define the cutoff for “listening now.”

**Root cause explanation:**  
The variable `RECENT_THRESHOLD` was set to 24 hours. Because `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`, the query allowed any friend listening event from the last full day to count as “listening now.” That caused older activity, including events from many hours earlier or potentially yesterday, to appear in a feature that should only show current listening activity.

**Fix description:**  
I changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. This keeps the feature focused on recent listening activity while still allowing seeded events from 10, 15, and 20 minutes ago to appear.

**Side-effect check:**  
I manually tested the feed after the fix. Friends who listened within 30 minutes still appeared, and older events were ignored. I also checked that the endpoint still returned the same JSON structure with `feed` and `count`.

---

### Issue #5: The last song in a playlist never shows up

**Reproduction steps:**  
I tested the playlist songs endpoint after seeding the database. The playlists were populated with multiple songs, but the response was missing the final song in the ordered playlist. This matched the issue report that the last song never shows up.

**Navigation strategy:**  
I started from `routes/playlists.py`, specifically the route for `GET /playlists/<playlist_id>/songs`. That route calls `get_playlist_songs(playlist_id)` in `services/playlist_service.py`. I followed the query that joins `Song` with `playlist_entries` and orders by `playlist_entries.c.position`. The query itself looked correct, so I checked the return statement and found `songs[:-1]`, which removes the final song from the result list.

**Root cause explanation:**  
The query correctly retrieved the playlist songs in ascending position order, but the return statement used `songs[:-1]`. In Python slicing, `[:-1]` means “all items except the last one.” Because of that slice, every playlist response intentionally dropped the final song after retrieving it from the database. The correct behavior requires returning the full `songs` list.

**Fix description:**  
I changed the return statement from `return [song.to_dict() for song in songs[:-1]]` to `return [song.to_dict() for song in songs]`.

**Side-effect check:**  
I ran `pytest tests/test_playlists.py`, and all playlist tests passed. I also manually checked the playlist songs endpoint to confirm that songs still appeared in playlist position order and that the final song was now included.

## AI Usage

## AI Usage

## AI Usage

I used AI in three specific debugging moments.

First, I used AI to understand how to test the API routes. It helped me figure out that `/songs/search?q=<query>` uses a query parameter and that feed routes need a real user UUID from the database.

Second, I used AI while debugging Issue #2. It helped me realize that I misscomprehended the issue at hand and that I was testing for the wrong outputs. Once corrected I used AI to figure out the bug

Third, I used AI while debugging Issue #1. It pointed out that `today.weekday() != 6` was excluding Sundays, which explained why a Saturday-to-Sunday streak reset. I verified the fix by running `pytest tests/test_streaks.py`.

I also course-corrected during the project: instead of using the duplicate search issue, I switched to Issue #5 after finding the clearer root cause, `songs[:-1]`, in `get_playlist_songs()`.
## Commit History

## Commit History

I worked on the required branch:

```text
bugfix/mixtape

f4ce45c fix: correct Sunday streak reset logic
b0b4f0e fix: narrow listening now recency window
1abc2ed fix: include last song in playlist results
```