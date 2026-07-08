## Codebase Map

### Main Files and Their Roles

**app.py** - Application factory. Creates the Flask app, configures the
SQLite database URI, registers the four route blueprints, and calls
db.create_all() to initialize tables. All other modules import `db` from
here.

**models.py** - Defines all 6 SQLAlchemy models and 3 association tables:
- `User` - has username, email, listening_streak, last_listened_at
- `Song` - has title, artist, genre, shared_by (FK to User), tags (M2M)
- `Tag` - simple name label; linked to songs via song_tags association table
- `Playlist` - has name, created_by; songs linked via playlist_entries
- `ListeningEvent` - records each time a user listens to a song
- `Rating` - stores a user's 1–5 score for a song (unique per user+song)
- `Notification` - stores messages sent to users when friends interact
  with their shared songs
- `friendships` - M2M association table linking users to their friends
- `song_tags` - M2M association table linking songs to tags
- `playlist_entries` - M2M association table with extra columns:
  position (int), added_by (FK), added_at (datetime)

**routes/** - Four Blueprint files. Each route does input parsing and
response formatting only - all business logic is delegated immediately
to a service function. No business logic lives in routes.
- `songs.py` - /songs/search, /songs/<id>, /songs/<id>/rate,
  /songs/<id>/listen
- `users.py` - /users/<id>, /users/<id>/streak,
  /users/<id>/notifications
- `feed.py` - /feed/<id>/listening-now, /feed/<id>/activity
- `playlists.py` - /playlists/, /playlists/<id>,
  /playlists/<id>/songs (GET and POST)

**services/** - Five service files containing all business logic:
- `streak_service.py` - records listening events and updates streak
- `feed_service.py` - returns friends' recent listening activity
- `search_service.py` - searches songs by title/artist with tag join
- `playlist_service.py` - creates playlists and retrieves ordered songs
- `notification_service.py` - creates notifications, handles ratings
  and playlist additions

### Data Flow: User Rates a Song

1. Client sends POST /songs/<song_id>/rate with user_id and score
2. `routes/songs.py` → `rate()` parses JSON, calls
   `notification_service.rate_song(user_id, song_id, score)`
3. `rate_song()` validates score (1–5), looks up Song and User,
   creates or updates a Rating record, commits to DB - but does NOT
   create a notification (this is Issue #4)
4. Returns the Rating dict to the route, which returns 201

### Data Flow: Get Playlist Songs

1. Client sends GET /playlists/<playlist_id>/songs
2. `routes/playlists.py` → `get_songs()` calls
   `playlist_service.get_playlist_songs(playlist_id)`
3. Queries Song joined to playlist_entries filtered by playlist_id,
   ordered by position ascending
4. Returns `songs[:-1]` - slicing off the last song (this is Issue #5)

### Pattern I Noticed

Every route immediately delegates to a service function - routes never
touch the database directly. This makes bugs easy to locate: find the
service function for the affected feature and the bug is almost always
there. The notification service doubles as a business logic handler for
both ratings and playlist additions, which is why Issue #4 lives there
despite being triggered from the songs route.


## Root Cause Analysis

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:**
Fetched GET /playlists/<playlist_id>/songs for all three seeded playlists.
Each returned count: 6 despite the seed data inserting 7 songs per playlist.
Confirmed the most recently added song was always missing.

**How I found the root cause:**
Went directly to services/playlist_service.py → get_playlist_songs() since
the route delegates immediately to that function. Read the return statement
and spotted songs[:-1] — Python slice notation that drops the last element.

**The root cause:**
The return statement in get_playlist_songs() used songs[:-1] instead of
songs. In Python, [:-1] slices a list from the beginning up to but not
including the last element. This meant every call to this function
silently dropped the last song regardless of playlist size. Adding a new
song moved the previous last song into a visible position while hiding
the new one — exactly matching the reported behavior.

**The fix and side-effect check:**
Changed songs[:-1] to songs on the return line. Verified all three
playlists now return the correct count (7 songs each). Checked that
ordering is still correct (songs still sorted by position ascending).
No other functions in playlist_service.py call this slice pattern.


### Issue #4 — No notification when a song is rated

**How I reproduced it:**
Had darius rate "Midnight Drive" (shared by nova) via
POST /songs/<song_id>/rate with score 5. The rating saved successfully
(returned a valid Rating dict). Then checked nova's notifications via
GET /users/<nova_id>/notifications — only the existing playlist
notification appeared, no rating notification.

**How I found the root cause:**
Went to services/notification_service.py since that's where both
ratings and playlist notifications are handled. Compared rate_song()
to add_to_playlist() line by line. add_to_playlist() ends with a
create_notification() call that notifies the song's original sharer.
rate_song() saves the rating and commits but then just returns —
no call to create_notification() anywhere in the function.

**The root cause:**
rate_song() in notification_service.py never called create_notification().
The function correctly saved the Rating record and committed it to the
database, but the notification block that exists in add_to_playlist()
was never implemented in rate_song(). The notification infrastructure
was already in place — create_notification() exists and works, and the
notification_type field supports "song_rated" — the call was simply
missing.

**The fix and side-effect check:**
Added a create_notification() call at the end of rate_song(), mirroring
the pattern in add_to_playlist(): check that the rater is not the
original sharer (song.shared_by != user_id), then notify the sharer
with a "song_rated" notification body showing the rater's username,
song title, and score. Verified nova received a "song_rated"
notification after darius rated "Still Waters" 4/5. Confirmed that
rating your own song (shared_by == user_id) correctly skips the
notification by the guard condition.

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it:**
Searched GET /songs/search?q=Anthem and GET /songs/search?q=a.
Results came back with correct counts in this SQLite environment, but
running the original query directly in a Python shell confirmed the
structural cause: the outerjoin multiplies rows by tag count before
SQLAlchemy's identity map deduplicates them.

**How I found the root cause:**
Went to services/search_service.py → search_songs(). The query used
.outerjoin(song_tags, Song.id == song_tags.c.song_id) before filtering.
Recognized that joining Song to its tag association table produces one
row per tag per song — a song with 3 tags produces 3 rows in the join
result. SQLAlchemy's identity map collapses these back into one Song
object in many cases, but this deduplication is not guaranteed and does
not work reliably across all database backends or query configurations.

**The root cause:**
search_songs() joined Song to the song_tags association table in order
to filter or include tag data, but Song.to_dict() already loads tags
automatically via the tags relationship defined in models.py with
lazy="subquery". The join was entirely unnecessary and produced one
result row per tag per song before SQLAlchemy's deduplication, causing
multi-tag songs to appear multiple times in search results depending on
the database backend and query context.

**The fix and side-effect check:**
Removed the .outerjoin(song_tags, ...) line entirely from the query.
Tags still load correctly on all songs because the Song model's tags
relationship with lazy="subquery" handles tag loading automatically
without needing an explicit join in the query. Verified that Anthem
returns 1 result, broad search returns 13 unique songs, and all
multi-tag songs (Crown Heights Anthem with 3 tags, Harlem Renaissance
with 3 tags, etc.) still show their full tag lists correctly.


## AI Usage

I used Claude as an AI tool throughout this project for codebase navigation,
bug investigation, and understanding unfamiliar code patterns.

**Instance 1 — Codebase orientation:**
I gave Claude the contents of all service files and route files and asked it
to help me understand the overall architecture and identify the responsibility
of each file. It produced a clear summary of how routes delegate to services
and how the notification pattern works. I verified this by reading the files
myself and confirmed the description was accurate before writing my codebase
map.

**Instance 2 — Bug #5 (playlist) investigation:**
After spotting songs[:-1] in playlist_service.py, I asked Claude to confirm
what Python's [:-1] slice notation does. It explained that [:-1] returns all
elements except the last one, which confirmed my reading of the bug. The fix
(changing to songs) was something I identified myself from reading the code,
Claude confirmed the behavior of the slice syntax I had already found.

**Instance 3 — Bug #4 (notification) investigation:**
I asked Claude to help me compare rate_song() and add_to_playlist() side by
side to identify what was structurally different. It pointed out that
add_to_playlist() ends with a create_notification() call while rate_song()
does not. I verified this myself by reading both functions and confirmed the
missing call was the root cause. I wrote the fix myself, modelling it on the
existing add_to_playlist() pattern.

**Instance 4 — Bug #3 (search duplicates) investigation:**
I asked Claude to explain why an outerjoin on a many-to-many association
table could produce duplicate rows. It explained that joining Song to
song_tags produces one row per tag per song, and that SQLAlchemy's identity
map may or may not deduplicate these depending on the backend. I verified
this by running the original query directly in a Python shell and confirmed
the structural issue. I identified that the join was unnecessary because
tags already load via the Song.tags relationship with lazy="subquery".


