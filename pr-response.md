# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude Code (AI CLI assistant) throughout this project, in these specific ways:

- **Codebase orientation (Milestone 1):** Before reading the review comments, I had it summarize `models.py`, `services/collection_service.py`, and `tests/test_collection.py` — what each file is responsible for and what patterns it establishes (the `verb_to_noun` naming, the custom-exception-per-error-case pattern, the fixture structure in tests). I verified the summaries against the code before relying on them.
- **Pattern verification (Comments 1–2):** I used a project-wide AI-assisted search to enumerate all `save_to_watchlist` call sites before renaming (3 references found, 0 after), and had it walk through `add_to_collection()`'s deduplication check step by step before writing the watchlist version.
- **Design decisions (Comments 4–5):** The positions are mine — I chose to keep `public=True` and to adopt date-added sort after weighing the tradeoffs. I used AI as a devil's advocate and to pressure-test the arguments against the actual codebase; that process surfaced two pieces of evidence I hadn't articulated: that `GET /collection/<user_id>` already exposes watch history and ratings publicly with no visibility flag (which became the backbone of my Comment 4 consistency argument), and that the `public` flag isn't yet *enforced* in `get_watchlist()` (which I documented as follow-up work). The final reasoning differs from a generic AI answer in that it's anchored to CineLog's existing behavior rather than abstract "public is better for discovery" claims.
- **Commit hygiene (Milestone 4):** I had it check my `git log --oneline` output against conventional commit format and the one-logical-change rule before finalizing the history, and verified the result myself against CONTRIBUTING.md.

Every AI-produced explanation and edit was verified by running the test suite (`pytest tests/ -v`) and, where relevant, manual end-to-end checks against an in-memory database.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` so it follows the project's `verb_to_noun` naming convention (matching `add_to_collection()`, `remove_from_collection()`, and `get_collection()` in `services/collection_service.py`, as listed in CONTRIBUTING.md). I also updated the function's docstring ("Save a film" → "Add a film") so the documentation matches the new name.

**How I verified:** Before editing, I ran a project-wide search for `save_to_watchlist` (excluding `.venv/`) to enumerate every reference. There were exactly three: the definition in `services/watchlist_service.py`, and an import plus one call site in `routes/watchlist/watchlist.py`. I updated all three, then re-ran the same search and confirmed zero matches remained. Finally I ran `pytest tests/ -v` — all 4 tests pass, and the route module imports cleanly (a missed import would have failed at collection time).

## Comment 2 — Deduplication
**What I did:** Before implementing anything I read how `add_to_collection()` in `services/collection_service.py` handles the same case: it queries `CollectionEntry.query.filter_by(user_id=..., film_id=...).first()` and, if an entry exists, raises a custom `AlreadyInCollectionError` — the route layer then translates that exception into an HTTP 409 Conflict. I mirrored that pattern in the watchlist:

- Added an `AlreadyInWatchlistError` exception class to `services/watchlist_service.py`.
- In `add_to_watchlist()`, after the existing film-exists check, I query `WatchlistEntry.filter_by(user_id=user_id, film_id=film_id).first()`; if a row comes back, the function raises `AlreadyInWatchlistError` instead of inserting a duplicate.
- In `routes/watchlist/watchlist.py`, the `POST /watchlist/<user_id>/add` handler now catches `AlreadyInWatchlistError` and returns `409` with an error message — the same status the collection endpoint uses — instead of letting a duplicate insert through (or a 500 escape).
- Updated the docstring's `Raises:` section to document the new exception.

**How I verified:** I ran a manual check against an in-memory database: created a user and a film, called `add_to_watchlist()` once (succeeded), then called it again with the same user/film and confirmed it raised `AlreadyInWatchlistError` rather than creating a second row. Then I ran the full suite (`pytest tests/ -v`) — all 4 tests still pass.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, which asserts that calling `add_to_watchlist()` with a `film_id` that doesn't exist in the database raises `FilmNotFoundError` (a clean domain exception) rather than a database integrity error. I modeled it on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` and reused that file's structure: the same `app` fixture (isolated app with an in-memory SQLite database, `db.create_all()` on setup and `drop_all()` on teardown), the same `sample_user` fixture, and the same `pytest.raises(...)` assertion style with a deliberately fake film ID.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` — the new test passes. Then ran the full suite (`pytest tests/ -v`) — all 5 tests pass, so the new file doesn't interfere with the existing collection tests.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for watchlist entries.

**Reasoning:** CineLog is a community film tracking app — the product's core loop is social: users log films, rate them, and browse what other people are watching. A watchlist that's visible by default feeds that loop ("what are my friends planning to watch?") and is what makes the feature social rather than a private todo list. More importantly, `public=True` is consistent with how the rest of CineLog already behaves: a user's collection — their actual watch history *including their ratings* — is openly readable today via `GET /collection/<user_id>`, with no visibility flag on `CollectionEntry` at all. It would be incoherent for the platform to treat what someone *intends* to watch as private by default while what they've *actually watched and how they rated it* is unconditionally public. If anything, the watchlist is the first feature in CineLog that has a per-entry visibility control at all, so users get an opt-out lever no other part of the app offers.

**Tradeoff acknowledged:** Private-by-default is the safer choice for user trust — it protects people who assume a watchlist is a personal notes feature, and it follows the "privacy by default" principle. I think that protection matters less here because (a) nothing else on the platform is private, so public watchlists don't expose a user beyond what CineLog already exposes, and (b) the per-entry `public` flag gives users an explicit opt-out. If CineLog later adds account-level privacy settings, the watchlist default should follow that setting rather than a hardcoded `True`. One follow-up worth its own issue: `get_watchlist()` doesn't yet filter out `public=False` entries when someone else views the list — the flag is stored but not enforced, and it should be before visibility is a real promise.

*How this decision is surfaced:* documented here and called out explicitly in the PR description, as requested.

## Comment 5 — Sort order
**My position:** Agree with the maintainer — I changed `get_watchlist()` to sort by `date_added` descending (newest first) and dropped the `join(Film)` that only existed to support the title sort.

**Reasoning:** Two things convinced me beyond the maintainer's point. First, consistency: `get_collection()` already returns newest-first, and the README documents the collection endpoint as "(newest first)". Users shouldn't have to learn two different mental models for the two lists CineLog shows them. Second, the way a watchlist actually gets used is queue-like: you add a film the moment you hear about it, and the most common reason to reopen the list is "what did I just add?" — recency correlates with intent. Alphabetical order optimizes for looking up a specific title, but that's a search problem, not a sort problem (the films endpoint already has filter params for lookup), and as a watchlist grows, alphabetical order buries every new addition in the middle of the list, making the list feel static.

**Engagement with reviewer's point:** The maintainer said "Most users want to see what they added recently." I agree, and the consistency argument reinforces it: newest-first is already the platform's established behavior for user lists, so date-added isn't just the better guess about user intent — it's the only choice that doesn't fragment the UX. If users later want a library-style browse, a `?sort=title` query param on `GET /watchlist/<user_id>` would serve both audiences without changing the default; I'd rather add that when someone asks than speculatively now.

*Implementation note:* while verifying the new sort I found that `get_watchlist()` crashed with `AttributeError` on any non-empty watchlist — `WatchlistEntry` had no `film` relationship (the `Film` model only declared a backref for `CollectionEntry`), so `entry.film.to_dict()` never worked. I added the missing `watchlist_entries` relationship on `Film` (mirroring the existing `collection_entries` pattern) as its own `fix:` commit, and verified newest-first ordering against an in-memory database with two entries added five days apart.

## Comment 6 — Rebase
**What conflicted:** While this PR was open, `main` merged a refactor that migrated film IDs from auto-incrementing integers to UUIDs (`Film.id` became `db.String(36)`, and `CollectionEntry.film_id` changed to match). My watchlist code was written against integer film IDs: `WatchlistEntry.film_id` was `db.Column(db.Integer, db.ForeignKey("film.id"))`, and the service/route docstrings documented `film_id` as an int. The conflict was semantic rather than a textual merge marker: `git rebase origin/main` replayed all my commits cleanly, but afterwards `models.py` was main's post-refactor version — which meant the watchlist feature referenced a `WatchlistEntry` model whose integer-ID definition no longer existed, and the test suite failed at import time (`ImportError: cannot import name 'WatchlistEntry'`).

**How I resolved it:** I ran `git fetch origin` then `git rebase origin/main` (no merge commit — rebase replays my commits on top of main, keeping history linear). Then, in a `fix:` commit on top, I restored the `WatchlistEntry` model in `models.py` updated for the new schema: `film_id` is now `db.String(36)` with the same `ForeignKey("film.id")`, matching how `CollectionEntry.film_id` was migrated. I also updated the two places that still documented integer IDs — the `add_to_watchlist()` docstring (`film_id (int)` → `film_id (str): UUID of the film`) and the route docstring (`Body: { "film_id": <int> }` → `"<uuid>"`).

**How I verified no conflict remains:** (1) `git log --merges origin/main..HEAD` returns nothing and `git log --graph` shows a straight line — no merge commits. (2) `grep` for integer film ID references in the watchlist code returns none. (3) `pytest tests/ -v` passes all 5 tests on top of the rebased main. (4) I ran an end-to-end check against an in-memory database: created films (whose IDs now come back as UUID strings), added them to a watchlist, confirmed dedup still raises `AlreadyInWatchlistError`, and confirmed `get_watchlist()` returns newest-first.

## Beyond the review comments

**404 on unknown film (small fix):** The add endpoint imported `FilmNotFoundError` but never caught it, so `POST /watchlist/<user_id>/add` with an unknown `film_id` returned a 500. The collection route returns 404 for the same case, so I added the matching `except FilmNotFoundError → 404` handler.

**Stretch — `remove_from_watchlist()`:** Implemented `remove_from_watchlist(user_id, film_id)` following the project's existing patterns: it mirrors `remove_from_collection()` exactly — look up the entry with `filter_by(user_id, film_id)`, raise a new `NotInWatchlistError` if the film isn't on the watchlist (rather than failing silently), otherwise delete and return `True`. Exposed as `DELETE /watchlist/<user_id>/remove` with the same body shape and status codes as the collection version (200 on success, 404 when not on the list, 400 when `film_id` is missing). Two tests cover it: the happy path (entry actually deleted from the DB) and the not-on-watchlist case (exception raised).

**Stretch — second test (edge case choice):** I added `test_add_to_watchlist_creates_entry` (happy path, including asserting the `public=True` default) and `test_add_to_watchlist_duplicate_raises` (duplicate add raises and leaves exactly one row). I chose these because CONTRIBUTING.md requires all three cases — happy path, duplicate/conflict, nonexistent ID — for any new service function, and the review only asked for the nonexistent-ID case. This completes the required coverage for `add_to_watchlist()` and directly regression-tests the Comment 2 deduplication fix.

**Stretch — visibility toggle:** `add_to_watchlist()` now takes a `public` parameter (default `True`, per the Comment 4 decision), and the endpoint accepts it in the request body: `POST /watchlist/<user_id>/add` with `{"film_id": "<uuid>", "public": false}` creates a private entry; omitting the key keeps the public default. Verified both paths manually and via the happy-path test's default assertion.

## PR Description

### What this PR does
Adds a **watchlist** feature to CineLog: users can save films they want to watch later, separate from their collection of films they've already logged. It introduces a `WatchlistEntry` model (user ↔ film link with `date_added` and a `public` visibility flag), service functions (`add_to_watchlist`, `remove_from_watchlist`, `get_watchlist`) following the project's `verb_to_noun` convention, and three endpoints:

| Method | Endpoint | Behavior |
|--------|----------|----------|
| GET | `/watchlist/<user_id>` | User's watchlist, newest first |
| POST | `/watchlist/<user_id>/add` | Add a film (`film_id` required, `public` optional) — 201, 404 unknown film, 409 duplicate |
| DELETE | `/watchlist/<user_id>/remove` | Remove a film — 200, 404 if not on the list |

Film IDs are UUIDs throughout, matching the recent `main` refactor (branch is rebased on it, linear history).

### Design decisions
- **Default visibility is `public=True`:** consistent with the rest of CineLog, where collections (watch history + ratings) are already openly readable per user; the per-entry `public` flag — settable via the API — is the platform's first privacy control, so users have an explicit opt-out. Full reasoning in `pr-response.md` (Comment 4).
- **Sort order is date-added, newest first:** matches `get_collection()`'s documented behavior and the queue-like way watchlists are used; alphabetical lookup is better served by search/filter than by default ordering. Full reasoning in `pr-response.md` (Comment 5).

### How to test manually
1. `pip install -r requirements.txt`
2. Seed a user and a film, and note the printed UUIDs — save this as `seed.py` and run `python seed.py`:
   ```python
   from app import create_app, db
   from models import User, Film

   app = create_app()
   with app.app_context():
       db.create_all()
       user = User(username="demo", email="demo@example.com")
       film = Film(title="Paddington 2", year=2017, genre="Comedy")
       db.session.add_all([user, film])
       db.session.commit()
       print("user_id:", user.id)
       print("film_id:", film.id)
   ```
3. Start the server: `flask --app app run`
4. Empty watchlist: `curl http://127.0.0.1:5000/watchlist/<user_id>` → `[]`
5. Add the film:
   `curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add -H "Content-Type: application/json" -d "{\"film_id\": \"<film_id>\"}"` → 201 with the entry JSON, `"public": true`
6. Add it again (duplicate): same command → **409** with an error message
7. Unknown film: same command with `"film_id": "00000000-0000-0000-0000-000000000000"` → **404**
8. View the list: `curl http://127.0.0.1:5000/watchlist/<user_id>` → the film with `date_added` attached (add a second film to see newest-first ordering)
9. Private entry: step 5's command with `-d "{\"film_id\": \"<other_film_id>\", \"public\": false}"` → 201 with `"public": false`
10. Remove: `curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove -H "Content-Type: application/json" -d "{\"film_id\": \"<film_id>\"}"` → 200; repeat → **404**
11. Run the suite: `pytest tests/ -v` → 9 tests pass
