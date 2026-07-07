# Mixtape Bug Hunt Submission

## AI Usage

For Milestone 1, I used AI to help organize my codebase map after reading the starter project files. I used AI to summarize the responsibilities of the uploaded files, connect the README structure to the actual Flask app setup, and write a clearer first draft of my `submission.md`. I still verified the structure by reading the files myself, especially `README.md`, `app.py`, `models.py`, and `seed_data.py`.

## Milestone 1: Codebase Map

### Main Project Structure

Mixtape is a Flask social music app where users can share songs, build playlists, rate music, receive notifications, and track listening activity.

The main files and folders are:

* `app.py`: Creates the Flask application using the `create_app()` factory function. It configures the database URL, disables SQLAlchemy modification tracking, sets a secret key, initializes the database, registers the route blueprints, and creates database tables. The registered blueprints are `songs_bp`, `playlists_bp`, `users_bp`, and `feed_bp`.

* `models.py`: Defines the database schema using SQLAlchemy. This file contains the main app entities: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, and `Notification`. It also defines association tables for friendships, song tags, and playlist entries.

* `routes/`: Contains the Flask route handlers. According to the README, this folder includes:

  * `routes/songs.py`: Handles song sharing, search, and rating routes.
  * `routes/playlists.py`: Handles playlist creation and playlist song management.
  * `routes/users.py`: Handles user profiles, streaks, and notifications.
  * `routes/feed.py`: Handles the activity feed and “friends listening now” feature.

* `services/`: Contains the business logic. The README says the bugs live in this layer. The service files include:

  * `streak_service.py`: Listening streak logic.
  * `feed_service.py`: Friends listening now feed logic.
  * `search_service.py`: Song search logic.
  * `notification_service.py`: Notification creation and retrieval.
  * `playlist_service.py`: Playlist retrieval logic.

* `seed_data.py`: Rebuilds and seeds the database with realistic test data. It creates users, friendships, tags, songs, listening events, playlists, playlist entries, and an example playlist notification.

* `requirements.txt`: Lists the project dependencies, including Flask, Flask-SQLAlchemy, SQLAlchemy, python-dotenv, and pytest.

* `.gitignore`: Ignores Python cache files, virtual environments, SQLite database files, environment variables, editor folders, and build artifacts.

### Data Model Notes

The app has several important database relationships:

* `User` represents a Mixtape user. A user has shared songs, ratings, listening events, notifications, playlists, and friends.
* Friendships are represented by the `friendships` association table. The seed file creates friendships in both directions, so friendships are treated as bidirectional.
* `Song` stores title, artist, album, genre, who shared the song, when it was shared, and an optional share note.
* `Tag` is connected to songs through the `song_tags` many-to-many association table.
* `ListeningEvent` records when a user listened to a song.
* `Rating` records a user’s rating for a song. The model has a unique constraint on `(user_id, song_id)`, so one user should only have one rating per song.
* `Playlist` stores playlist metadata such as name, creator, creation time, and whether it is collaborative.
* Playlist songs are stored through the `playlist_entries` association table. This table includes `playlist_id`, `song_id`, `position`, `added_by`, and `added_at`, which means playlist song order is tracked explicitly.
* `Notification` stores user notifications with a recipient user ID, notification type, body, creation time, and read/unread state.

### Organization Pattern I Noticed

The project follows a route-to-service pattern. The route files handle web requests and call service files for the real business logic. The README specifically says the bugs are in the `services/` layer, so when debugging an endpoint, I should start at the route and then trace the call into the relevant service file.

This means I should not fix bugs by only changing the route response. I need to trace the behavior into the service function that calculates or retrieves the incorrect result.

### Data Flow Example: User Rates a Song

One example data flow from the README is rating a song:

1. A user sends a request to `POST /songs/<song_id>/rate`.
2. The request is handled by `routes/songs.py`.
3. The route calls into `notification_service.rate_song()`.
4. The app creates or updates a `Rating` record for the user and song.
5. Since `Rating` has a unique constraint on `(user_id, song_id)`, the app should avoid creating duplicate ratings for the same song by the same user.
6. If the rated song belongs to another user, the notification service should create a notification for the original song sharer.
7. The database commit saves the rating and any notification.

This flow matters for Issue #4, where the user receives a notification when a friend adds their song to a playlist but does not receive one when the song is rated. To debug that issue, I should compare the working playlist-add notification path against the rating notification path.

### Data Flow Example: Playlist Song Display

Another important data flow is viewing playlist songs:

1. A user requests `GET /playlists/<id>/songs`.
2. The request is handled by `routes/playlists.py`.
3. The route calls `playlist_service.get_playlist_songs()`.
4. The service should query the playlist entries for that playlist.
5. Because `playlist_entries` has a `position` column, the returned songs should be ordered by their playlist position.
6. The route returns the playlist songs to the user.

This flow matters for Issue #5, where the last song in a playlist never shows up. Since the playlist entries table uses explicit positions, I should check whether the playlist service has an off-by-one boundary error or slices the result incorrectly.

### Seed Data Notes

The seed file creates useful data for reproducing the bugs:

* It creates 5 users: `nova`, `darius`, `simone`, `kenji`, and `aaliya`.
* It creates bidirectional friendships between several users.
* It creates songs with 0 tags, 1 tag, and 3 or more tags.
* The seed comments say songs with 3 or more tags expose Issue #3, where the same song appears twice in search.
* It creates recent listening events within the past 30 minutes, which should appear in “friends listening now.”
* It also creates older listening events from 2 or more hours ago, which should not appear in “friends listening now” after the feed bug is fixed.
* It creates existing listening streak values and `last_listened_at` values for some users.
* It creates three playlists with 5–7 songs each.
* It creates an example `song_added_to_playlist` notification, which can be used as the working pattern when debugging missing rating notifications.

### Five Issues Read

I read the five open issues listed in the README:

1. “My listening streak keeps resetting” — affected service: `streak_service.py`
2. “Friends Listening Now shows people from yesterday” — affected service: `feed_service.py`
3. “The same song keeps showing up twice in search” — affected service: `search_service.py`
4. “I got notified when a friend added my song to a playlist but not when they rated it” — affected service: `notification_service.py`
5. “The last song in a playlist never shows up” — affected service: `playlist_service.py`

### Rough Plan for Which Bugs to Tackle First

I plan to start with Issues #1, #2, and #5 because the seed data and README give clear clues for reproducing them:

* Issue #1 can likely be reproduced by streak breaks every Sunday.
* Issue #2 can likely be reproduced by fix friends listening now showing old events.
* Issue #5 can likely be reproduced by viewing one of the seeded playlists and checking whether all expected songs appear, especially the final song.

If one of these is harder to reproduce than expected, I may switch to Issue #2 because the seed file includes both recent and old listening events.

## Root Cause Analysis Entries

I will complete one section like this for each bug I fix.

**Issue #1: My listening streak keeps resetting**
How I Reproduced It

I reproduced this bug after running python seed_data.py, starting the Flask app, and checking the seeded users' streak data.

I used the seeded user nova:

dbb89ea6-3d69-499a-85fa-a3e5586040e1

I first checked Nova's current streak with:

GET /users/dbb89ea6-3d69-499a-85fa-a3e5586040e1/streak

Then I triggered a new listening event using the seeded song Midnight Drive:

POST /songs/cdd1cffa-7709-4087-8901-f006192f7c4c/listen

with this JSON body:

{"user_id":"dbb89ea6-3d69-499a-85fa-a3e5586040e1"}

After the listen request, I checked Nova's streak again using the same streak endpoint. The streak changed unexpectedly instead of preserving or continuing the existing streak correctly. This confirmed the streak reset behavior before I changed any code.

How I Found the Root Cause

I started from the route that records a listening event: POST /songs/<song_id>/listen in routes/songs.py. That route calls record_listening_event() from services/streak_service.py.

I followed the logic in streak_service.py because the README lists Issue #1 as belonging to streak_service.py. I focused on the date comparison and weekly boundary logic because the bug happened around streak reset behavior.

The key clue was the weekday check. The code was using Python's weekday numbering incorrectly.

The Root Cause

The streak reset logic was checking the wrong weekday value for the week boundary. Python's datetime.weekday() returns 0 for Monday and 6 for Sunday. The code treated weekday() == 0 as the Sunday/week-boundary case, but that condition actually matches Monday.

Because of that mismatch, the streak logic reset or preserved streaks on the wrong day. A user listening around the Sunday boundary could have their streak handled incorrectly.

My Fix and Side-Effect Check

I changed the weekday check so Sunday is detected correctly. I used isoweekday() == 7, which is clearer because ISO weekday represents Sunday as 7.

After the change, I reran the seed script, restarted the Flask app, triggered a listening event with POST /songs/<song_id>/listen, and checked the user streak again with GET /users/<user_id>/streak. I also checked that normal same-day listening still did not incorrectly reset the streak.

### Issue #2: Friends Listening Now shows people from yesterday

#### How I Reproduced It

I reproduced this bug after running `python seed_data.py`, starting the Flask app, and using the listening-now feed endpoint.

I used one of the seeded users with friends, such as `nova`, and opened:

`GET /feed/<user_id>/listening-now`

The seed data creates recent listening events from the past 30 minutes, which should appear in the listening-now feed. It also creates older listening events from hours or days earlier, which should not appear as currently listening.

When I checked the endpoint response, I saw listening activity that was too old to count as “listening now.” This confirmed the bug that the feed includes stale listening events before I changed any code.

### Issue #5: The last song in a playlist never shows up

#### How I Reproduced It

I reproduced this bug after running `python seed_data.py`, starting the Flask app, and getting the seeded playlist IDs from the database.

I used this command to list the playlists:

`python -c "from app import create_app; from models import Playlist; app=create_app(); app.app_context().push(); [print(p.name, p.id) for p in Playlist.query.all()]"`

Then I opened the playlist songs endpoint:

`GET /playlists/<playlist_id>/songs`

The seed data creates playlists with multiple songs. I expected the playlist endpoint to return all songs in the playlist.

Instead, the endpoint returned one fewer song than expected. The final song in the playlist order was missing. This confirmed Issue #5 before I changed any code.
