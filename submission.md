# Mixtape Bug Hunt Submission
**Author:** Elaheh Baharlouei

## Codebase Map
### Issue #1 — My listening streak keeps resetting

* **How I reproduced it:** I opened the database using a Flask shell and manually set a test user's `last_listened_at` timestamp to a Saturday, and their `listening_streak` to 12. I then changed my system clock to Sunday and triggered a new listening event for that user. I verified via `GET /users/<id>/streak` that the streak reset to 1 instead of incrementing to 13.
* **How I found the root cause:** I traced the execution top-down. I started at the `GET /<user_id>/streak` route in `routes/users.py`, which delegates directly to the `streak_service.py` file. I read the `update_listening_streak` function to follow the conditional logic and noticed a hardcoded condition checking the day of the week. I used an AI tool to clarify what edge cases could cause `today.weekday() != 6` to fail, and it confirmed that `6` represents Sunday in Python. The moment of confidence was seeing that Sunday listens explicitly forced the code to skip the increment block and fall into the default reset block.
* **The root cause:** Python's `datetime.weekday()` assigns Monday as 0 and Sunday as 6. The `update_listening_streak` function contained a hardcoded condition (`elif days_since_last == 1 and today.weekday() != 6:`) that actively prevented the streak from incrementing on Sundays. When a user listened on a Sunday, this condition evaluated to `False`, causing the execution to fall through to the `else:` block, which erroneously reset their streak to 1. 
* **My fix and side-effect check:** I removed the `and today.weekday() != 6` condition so the logic purely checks if `days_since_last == 1`. To ensure no side effects, I verified the surrounding boundary conditions: same-day listens (`days_since_last == 0`) still exit early without changing the streak, and skipped days (`days_since_last > 1`) correctly trigger the `else` block to reset the streak to 1 as intended.


### Main Files & Responsibilities
* **`models.py`:** Defines the core SQLAlchemy database schema (`User`, `Song`, `Playlist`, `ListeningEvent`, etc.). Notably, it utilizes association tables for complex relationships, such as `playlist_entries`, which explicitly tracks a song's `position` in a playlist using an integer rather than relying on insertion order.
* **`routes/users.py`:** Handles incoming HTTP requests for user-related endpoints. It strictly focuses on extracting request parameters, delegating logic to the service layer, and formatting JSON responses.
* **`services/streak_service.py`:** Contains the business logic for calculating user listening streaks. It evaluates date deltas to determine whether a streak should increment, remain unchanged, or reset to 1.

### Data Flow Trace: Updating a Listening Streak
When a user listens to a song, the system executes `record_listening_event(user_id, song_id)` inside `services/streak_service.py`. This initiates a two-step flow:
1. A new `ListeningEvent` record is committed to the database with the current UTC timestamp.
2. The service immediately calls `update_listening_streak()`, which calculates the `days_since_last` listen by comparing today's date against the user's `last_listened_at` timestamp. Based on this delta, the user's `listening_streak` attribute is mutated on the `User` model.

### Architectural Pattern Notice
The application strictly adheres to a separation of concerns pattern. The route files (like `routes/users.py`) act purely as traffic controllers doing input parsing and response formatting. Every route delegates immediately to a service function. All actual business logic and database manipulations live exclusively in the `services/` layer.