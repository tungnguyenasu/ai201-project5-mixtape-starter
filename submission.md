# Mixtape Bug Hunt Submission

## AI Usage

I used an AI assistant (Claude Code) throughout this project, mostly as a navigation and explanation tool rather than a code generator. Being specific about how:

**Codebase navigation.** Before touching any bug, I had the AI walk me through the project layer by layer. I asked it to explain what `models.py` defines (the SQLAlchemy tables and their relationships), what the `routes/` blueprints do, and how a request flows from a route into the matching `services/` function and down to the models. This gave me the route → service → model mental model quickly. I also had it trace specific flows end to end — for example `POST /songs/<id>/listen` → `record_listening_event()` → `update_listening_streak()`, and `GET /playlists/<id>/songs` → `get_playlist_songs()`. I confirmed every claim by opening the files myself (`app.py`, `models.py`, `seed_data.py`, and the individual service files) rather than trusting the summary.

**Debugging.** For each bug I fixed, I used the AI to help locate the exact line and explain why it was wrong: the `RECENT_THRESHOLD = timedelta(hours=24)` window in `feed_service.py` (Issue #2), the `songs[:-1]` slice in `get_playlist_songs()` (Issue #5), and the streak-increment condition in `update_listening_streak()` (Issue #1). It was most useful for narrowing down where to look from the symptom — e.g. "one fewer song than expected" pointing at a slicing/off-by-one problem rather than the query.

**Where I had to verify or correct the AI.** The collaboration was not hands-off:

* Its first explanation of the streak bug was wrong. It described the original condition as `weekday() == 0` and proposed a fix using `isoweekday() == 7`. When I checked the actual code with `git show` on the fix commit, the real bug was an extra `and today.weekday() != 6` condition, and the correct fix was to delete that condition entirely — not to change a comparison value. I rewrote the Issue #1 root-cause entry to match the real code.
* It surfaced extra "bugs" beyond the five assigned issues (for example a possible NOT NULL failure when adding a song to a playlist through the association table). These were speculative and out of scope, so I set them aside instead of acting on them.
* When I asked it to make the formatting of the issue write-ups consistent, it applied the change in the wrong direction the first time and I had to correct it.
* I did the reproduction and verification myself — running `seed_data.py`, starting the Flask app, and hitting the endpoints — rather than relying on the AI's assertion that a fix worked.

Overall the AI was most valuable for orientation (understanding an unfamiliar codebase fast) and for explaining suspect code, but I treated its explanations as leads to verify, not conclusions — and at least once it was confidently wrong.

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

### Issue #1: My listening streak keeps resetting

#### How I Reproduced It

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

#### How I Found the Root Cause

I started from the route that records a listening event: POST /songs/<song_id>/listen in routes/songs.py. That route calls record_listening_event() from services/streak_service.py.

I followed the logic in `streak_service.py` because the README lists Issue #1 as belonging to that file. I focused on `update_listening_streak()`, which decides whether to increment, keep, or reset the streak.

The key clue was in the branch that increments the streak. Besides checking that exactly one day had passed since the last listen, it carried an extra condition tied to the current day of the week, which has nothing to do with the documented streak rules.

#### The Root Cause

The branch that increments the streak read:

`elif days_since_last == 1 and today.weekday() != 6:`
`    user.listening_streak += 1`

Python's `datetime.weekday()` returns `6` for Sunday. The extra `and today.weekday() != 6` meant that whenever "today" was a Sunday, a user who had listened the day before — a valid consecutive-day streak — failed this condition and fell through to the `else` branch, which resets the streak to `1`. In effect, every Sunday wiped out an otherwise-valid streak.

The documented rules say a listen on a consecutive calendar day should always increment the streak regardless of which day of the week it is, so this weekday condition should not have been in the code at all.

#### My Fix and Side-Effect Check

I removed the spurious weekday condition so the branch is simply:

`elif days_since_last == 1:`
`    user.listening_streak += 1`

Now a listen exactly one calendar day after the previous one increments the streak on any day, including Sunday.

`update_listening_streak()` has only three paths — same day (`days_since_last == 0`, no change), consecutive day (`== 1`, increment), and a larger gap (`else`, reset to `1`). My change only affects the consecutive-day path, so the other two are unchanged. After the fix I reran `python seed_data.py`, restarted the Flask app, triggered a listen with `POST /songs/<song_id>/listen`, and re-checked `GET /users/<user_id>/streak`: a consecutive-day listen now increments on a Sunday instead of resetting, a same-day repeat listen still leaves the streak unchanged, and a gap of two or more days still resets to 1.

### Issue #2: Friends Listening Now shows people from yesterday

#### How I Reproduced It

I reproduced this bug after running python seed_data.py, starting the Flask app, and using the listening-now feed endpoint.

I used the seeded user nova because Nova has several friends in the seed data:

GET /feed/dbb89ea6-3d69-499a-85fa-a3e5586040e1/listening-now

The seed data creates recent listening events that should appear in the listening-now feed, but it also creates older listening events that should not appear as currently listening.

When I checked the endpoint response, I saw listening activity that was too old to count as “listening now.” This confirmed the bug that the feed includes stale listening events before I changed any code.

#### How I Found the Root Cause

I started from the route shown by flask routes:

GET /feed/<user_id>/listening-now

Then I traced that route into services/feed_service.py, because the README identifies Issue #2 as belonging to feed_service.py.

I looked for the query that retrieves listening events for a user's friends. The suspicious part was the timestamp filter because the bug was not about missing data; it was about old data being included.

#### The Root Cause

The listening-now query already computed a cutoff correctly (`cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`) and filtered with `ListeningEvent.listened_at >= cutoff`. The problem was the value of the `RECENT_THRESHOLD` constant in `feed_service.py`:

RECENT_THRESHOLD = timedelta(hours=24)

A 24-hour window meant any friend who had listened at any point in the past day was treated as "listening now," so stale events from yesterday or many hours ago showed up in the feed.

#### My Fix and Side-Effect Check

I changed the single `RECENT_THRESHOLD` constant from 24 hours to 5 minutes:

RECENT_THRESHOLD = timedelta(minutes=5)

The existing cutoff computation and filter were already correct, so no other logic needed to change. This narrows the window so only friends who listened in the last 5 minutes appear.

After the fix, I reran the seed script and checked:

GET /feed/<user_id>/listening-now

The response only included recent listening events and no longer included stale events from hours ago or yesterday. I also confirmed `get_activity_feed()` was left untouched, since that feed is intentionally not filtered by recency, and that recent seed events still appeared so the feed was not over-filtered.

### Issue #5: The last song in a playlist never shows up

#### How I Reproduced It

I reproduced this bug after running `python seed_data.py`, starting the Flask app, and getting the seeded playlist IDs from the database.

I used this command to list the playlists:

`python -c "from app import create_app; from models import Playlist; app=create_app(); app.app_context().push(); [print(p.name, p.id) for p in Playlist.query.all()]"`

Then I opened the playlist songs endpoint:

`GET /playlists/<playlist_id>/songs`

The seed data creates playlists with multiple songs. I expected the playlist endpoint to return all songs in the playlist.

Instead, the endpoint returned one fewer song than expected. The final song in the playlist order was missing. This confirmed Issue #5 before I changed any code.

#### How I Found the Root Cause

I started from the route `GET /playlists/<playlist_id>/songs` in `routes/playlists.py`, which calls `playlist_service.get_playlist_songs()`. The README lists Issue #5 as belonging to `playlist_service.py`.

The query itself looked correct: it joins `Song` to the `playlist_entries` association table, filters by `playlist_id`, and orders by `position` ascending. Since the data and ordering were right but the count was short by exactly one, I focused on how the result list was returned rather than how it was queried.

#### The Root Cause

The service sliced the ordered result with `songs[:-1]` before serializing:

`return [song.to_dict() for song in songs[:-1]]`

`[:-1]` drops the last element of the list. Because the songs were ordered by ascending `position`, the dropped element was always the final song in the playlist. This is why every playlist returned exactly one fewer song, with the last one missing, even though the query returned the complete, correctly ordered set.

#### My Fix and Side-Effect Check

I removed the slice so all songs are returned:

`return [song.to_dict() for song in songs]`

The query, join, filter, and ordering were already correct, so no other logic needed to change.

After the fix, I reran `python seed_data.py`, restarted the Flask app, and checked `GET /playlists/<playlist_id>/songs` for the seeded playlists (which have 5–7 songs each). The endpoint now returns all songs in ascending position order, including the final song. I also confirmed that an empty playlist returns an empty list rather than erroring, since `[:-1]` on an empty list and the fixed version both yield `[]`.
