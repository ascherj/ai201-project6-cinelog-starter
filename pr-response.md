# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Used Claude Code for codebase orientation (summarizing what `add_to_collection()` does and how tests are structured) and to stress-test the design arguments for Comments 4 and 5. All code changes and final design positions are my own. For Comments 4 and 5, I wrote my draft first, then asked the AI what counterarguments a careful reviewer might raise — this surfaced the "privacy-first default" angle for Comment 4, which I addressed directly in my final response.

---

## Comment 1 — Rename

**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` (function definition) and updated the one call site in `routes/watchlist/watchlist.py` (import and the call inside `add_film()`). Used a project-wide search to confirm no other references remained.

**How I verified:**
Ran `grep -rn "save_to_watchlist" .` across the repo — only references remaining were in `instruct.md` (the instructions doc, not production code). Full test suite still passed after the rename.

---

## Comment 2 — Deduplication

**What I did:**
Added an `AlreadyInWatchlistError` exception class to `services/watchlist_service.py`, mirroring the `AlreadyInCollectionError` pattern in `collection_service.py`. Inside `add_to_watchlist()`, I added a query for an existing `WatchlistEntry` with the same `user_id` and `film_id` — if found, it raises `AlreadyInWatchlistError` before attempting the insert. I also added a `UniqueConstraint("user_id", "film_id")` to the `WatchlistEntry` model to enforce deduplication at the database level as a second line of defense. Updated the route to catch and return a 409 for this error, matching the collection endpoint's pattern.

**How I verified:**
The `test_add_to_watchlist_duplicate_raises` test confirms only one entry exists after two add attempts. Full test suite passes.

---

## Comment 3 — Missing test

**What I did:**
Created `tests/test_watchlist.py` following the same fixture structure as `tests/test_collection.py` — separate `app`, `sample_user`, and `sample_film` fixtures using an in-memory SQLite database. Wrote `test_add_to_watchlist_nonexistent_film_raises` using a fake UUID as `film_id` and asserting `FilmNotFoundError` is raised, directly mirroring `test_add_to_collection_nonexistent_film_raises`. Also included a duplicate test (`test_add_to_watchlist_duplicate_raises`) to cover the deduplication logic added in Comment 2.

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` — both tests pass. Full suite (`pytest tests/ -v`) shows 6/6 passing.

---

## Comment 4 — Default visibility

**My position:**
Keep `public=True` as the default.

**Reasoning:**
CineLog is described as a *community* film tracking app — its value proposition is social discovery, not private journaling. A watchlist that defaults to private defeats the community purpose: users would need to opt in to share, and most won't bother changing a default. The result would be a ghost platform where nobody can see what anyone else wants to watch.

The `public=True` default optimizes for the primary use case — users want to share what they're planning to watch and discover what their community is saving. This is consistent with how similar platforms (Letterboxd, Goodreads) work: lists are public by default, with the option to make specific entries private.

**Tradeoff acknowledged:**
The risk is that users who add films they're embarrassed about (guilty pleasures, guilty rewatches) may not realize their list is public by default. The mitigation is clear UI labeling at add-time — the API already accepts a `public` field, so callers can override it explicitly. A future improvement would be a user-level privacy preference that overrides the default for all new entries.

---

## Comment 5 — Sort order

**My position:**
Change `get_watchlist()` to sort by `date_added` descending (newest first), matching the reviewer's preference and the existing `get_collection()` pattern.

**Reasoning:**
A watchlist is a queue of intent — films a user is actively planning to watch. The most recently saved films are the ones most likely to be top of mind. Alphabetical sort is useful for a static reference list, but a watchlist is dynamic: users add films after seeing a trailer, a recommendation, or a review, and they want those recent additions visible immediately at the top.

More importantly, `get_collection()` already sorts by `date_added` descending. Keeping both endpoints consistent means the mental model is the same across the app — "newest first" — rather than requiring users to remember that the collection and watchlist sort differently.

**Engagement with reviewer's point:**
The reviewer's point about consistency with `get_collection()` is the strongest argument here — not just aesthetics, but cognitive consistency across the app. I initially kept alphabetical because it makes a specific film easier to find by name, but that problem is better solved with search/filter than by changing the default sort. The default should optimize for browsing behavior, not lookup behavior.

---

## Comment 6 — Rebase

**What conflicted:**
After rebasing `feature/watchlist` onto the updated `main`, the conflict was in `models.py`. The `main` branch had migrated `Film.id` from `db.Column(db.Integer, ...)` to `db.Column(db.String(36), ...)` with UUID generation, and `CollectionEntry.film_id` from `Integer` to `String(36)`. The `feature/watchlist` branch had added `WatchlistEntry` with `film_id = db.Column(db.Integer, ...)` — a pre-refactor type that was now inconsistent with the rest of the schema.

**How I resolved it:**
Updated `WatchlistEntry.film_id` to `db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False)` to match the post-refactor UUID type used by `CollectionEntry`. Also updated the docstring in `watchlist_service.py` that still referenced `film_id (int)` to correctly document it as a UUID string.

**How I verified no conflict remains:**
Ran `git log --oneline` to confirm no merge commits in the branch history. Ran the full test suite (`pytest tests/ -v`) — 6/6 passing after the rebase.

---

## PR Description

### Watchlist Feature — Add films to a personal watchlist

This PR adds a watchlist feature to CineLog, allowing users to save films they want to watch later. Unlike the collection (films already watched), the watchlist is a forward-looking queue.

**Endpoints added:**
- `GET /watchlist/<user_id>` — returns the user's watchlist, sorted by date added (newest first)
- `POST /watchlist/<user_id>/add` — adds a film to the watchlist; returns 409 if already present, 404 if film doesn't exist

**Design decisions made:**

1. **Default visibility (`public=True`):** Watchlist entries default to public, consistent with CineLog's community-first purpose. Callers can pass `"public": false` in the request body to override per-entry.

2. **Sort order (date-added, descending):** `get_watchlist()` sorts by `date_added` descending, matching `get_collection()` behavior and optimizing for browsing recently saved films.

**How to manually test:**

```bash
# 1. Start the app
python app.py

# 2. You'll need a user_id and film_id from the database.
#    Seed a film directly via the SQLite shell, or use existing IDs.

# 3. Add a film to the watchlist
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'
# Expected: 201 with the new WatchlistEntry

# 4. Add the same film again (deduplication check)
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_uuid>"}'
# Expected: 409 with error message

# 5. Add a nonexistent film
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
# Expected: 404 with error message

# 6. View the watchlist
curl http://127.0.0.1:5000/watchlist/<user_id>
# Expected: 200 with list of films, newest first

# 7. Run the test suite
pytest tests/ -v
# Expected: 6/6 passing
```

<!-- Screenshot of git log --oneline to be added after interactive rebase in Milestone 4 -->
