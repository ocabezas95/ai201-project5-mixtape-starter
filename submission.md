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

## AI Usage

_(We will fill this section out at the very end of the project!)_
