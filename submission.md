# Mixtape Bug Hunt Submission

## Codebase Map

### Main Files & Responsibilities

- **models.py**: Defines the SQLAlchemy database models (entities like User, Song, Playlist, Notification) that represent the application's core state.
- **routes/**: Contains the API controller layer (`songs.py`, `playlists.py`, `users.py`, `feed.py`). These files handle incoming HTTP requests, parse inputs, and format HTTP responses, but delegate all core business logic to the services layer.
- **services/**: Contains the core business logic of the application (`streak_service.py`, `feed_service.py`, `search_service.py`, etc.). This is where data is processed, calculated, and where the 5 open bugs live.

### Architectural Patterns

- **Route-to-Service Delegation**: Every route delegates immediately to a service function. The routes do input parsing and response formatting, while all business logic lives in `services/`.

### Sample Data Flow Trace

- **Feature**: User views a playlist's songs.
- **Flow**: `GET /playlists/<id>/songs` in `routes/playlists.py` is triggered -> It invokes `playlist_service.get_playlist_songs()` -> The service queries the database using models from `models.py` and returns the dataset back to the route to be served as a response.

---

## Root Cause Analysis

### Issue 1: My listening streak keeps resetting

- **How you reproduced it**: To simulate a streak update on a Sunday, I set up a scenario where a user had a consecutive listening record from Friday to Saturday (`days_since_last == 1`). When triggering an update where the current simulation time fell on a Sunday, `today.weekday()` returned `6`. This caused the condition `elif days_since_last == 1 and today.weekday() != 6:` to evaluate to `False`, forcing the logic into the `else` block and resetting the streak to 1 instead of incrementing it.
- **How you found the root cause**: I navigated to `services/streak_service.py` based on the affected service tracking table in the README. Tracing the execution flow within `update_listening_streak`, I isolated the date comparison logic and evaluated the behavior of Python's built-in `datetime.weekday()` method on weekend boundary transitions.
- **The root cause**: Python's `datetime.weekday()` maps Monday through Sunday as integers from `0` to `6` (where Sunday is `6`). The codebase explicitly included a check `today.weekday() != 6` within the day-increment conditional branch. This design logic mistakenly treated Sunday as an invalid day for a consecutive daily streak rollover, forcing a streak reset on Sundays even if the user listened consecutively on Saturday.
- **Your fix and side-effect check**: I modified the conditional statement in `services/streak_service.py` to remove the incorrect `and today.weekday() != 6` evaluation block, allowing consecutive daily listens (`days_since_last == 1`) to successfully increment the streak regardless of the day of the week. After implementing the fix, I verified that updating streaks on both regular weekdays and Sundays increments correctly without resetting using the `pytest` test suite.

### Issue 5: The last song in a playlist never shows up

- **How you reproduced it**: Using the `flask shell`, I queried the `playlist_entries` join table directly for a sample playlist and counted its total rows (7 entries). I then invoked the service-layer function `get_playlist_songs()` using that same playlist ID and found that it returned an array containing only 6 entries, completely omitting the final item.
- **How you found the root cause**: Based on the project structure map, I traced the `GET /playlists/<id>/songs` controller route to its service delegation inside `services/playlist_service.py`. Looking at the return statement of `get_playlist_songs()`, I identified a Python list slicing operation that truncates the final element of the array.
- **The root cause**: The function fetched the correct array of database records but returned them using the Python slice syntax `songs[:-1]`. In Python, negative indexing on a slice operates up to, but explicitly excludes, the element at the specified end index. This forced the application to drop the last sequence track from the dataset immediately before formatting the HTTP response.
- **Your fix and side-effect check**: I modified the return statement to evaluate as `return [song.to_dict() for song in songs]`, entirely stripping away the truncating slice so that the complete collection is mapped to dictionary representations. I then verified the change by executing `pytest tests/test_playlists.py` to ensure all tests passed successfully without causing regressions in ordering or response structures.

### Issue 2: Friends Listening Now shows people from yesterday

- **How you reproduced it**: By running the `flask shell` and inspecting the service layer constants, I printed out the value of `RECENT_THRESHOLD`. It was configured as `1 day, 0:00:00`, confirming that any song listening events that occurred up to 24 hours ago were actively being flagged by the query window and pushed into the user's active "Listening Now" feed.
- **How you found the root cause**: Guided by the bug tracker map, I opened `services/feed_service.py` to inspect how the dynamic datetime window boundary was calculated. I immediately isolated the declaration `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` and traced `RECENT_THRESHOLD` back to its global declaration at the top of the file.
- **The root cause**: The variable `RECENT_THRESHOLD` was statically defined as `timedelta(hours=24)`. While syntactically valid, a 24-hour delta causes the active query filter to include listening historical events from the previous calendar day. For an instantaneous feed feature designed to show friends listening "now," this time threshold was far too wide.
- **Your fix and side-effect check**: I modified the declaration at the top of `services/feed_service.py` to read `RECENT_THRESHOLD = timedelta(minutes=5)`. This restricts the listening event lookup query to a tight five-minute buffer, ensuring the feed exclusively returns contemporary activity. After saving, I executed the full `pytest` suite to verify that no secondary service features or shared logic dependencies were broken by narrowing the threshold delta.

## AI Usage

I utilized an AI assistant as an interactive engineering co-pilot throughout this debugging sprint. Rather than relying on automated code-generation or guessing solutions blindly, the AI assisted with high-level codebase navigation, tracing data flow architectures (Route-to-Service delegation chains), and breaking down Python standard library structures (such as evaluating `datetime.weekday()`'s `0-6` indexing limits and assessing `timedelta` durations). Every single bug was isolated, structurally evaluated, and verified by me using manual environment tests (`flask shell`) and unit testing tools (`pytest`) prior to finalizing commits.
