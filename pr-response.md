# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude (Claude Code) throughout this project, primarily for orientation and hygiene rather than for the design decisions themselves:

- **Codebase orientation:** Before touching any review comment, I had it summarize `models.py`, `services/collection_service.py`, and `tests/test_collection.py` — what each function does, the `verb_to_noun` naming convention, and the fixture/assertion pattern used across tests. This is what made it obvious that Comment 1's rename should follow `add_to_collection()`'s pattern, and that Comment 2's deduplication should follow `add_to_collection()`'s check-then-raise-then-insert structure, rather than inventing a new approach.
- **Rebase debugging:** After `git rebase origin/main` reported success with no conflicts, I used it to help trace through *why* `get_watchlist()` started raising `AttributeError: 'WatchlistEntry' object has no attribute 'film'` once the new sort-order test ran. That surfaced the real problem: main's UUID refactor commit had silently dropped the entire `WatchlistEntry` class from `models.py` (since the watchlist feature hadn't merged into main yet), and none of my commits' diffs touched that class directly, so git's line-based merge never flagged it as a conflict. Without that investigation I might have assumed the rebase completed cleanly, when in fact it produced a `models.py` that would break the whole feature at runtime.
- **Stress-testing Comment 4 and Comment 5:** After drafting my own position on both (public=True default, date-added sort order), I asked it to argue the counterposition as a careful reviewer would. For Comment 4, it pushed back that "matching an existing bad pattern (CollectionEntry having no privacy control at all) isn't necessarily a good reason to repeat it for a new feature" — which is a fair point, and I addressed it directly in my Tradeoff section rather than ignoring it: I acknowledge that a `public=False` default is the safer, more privacy-conscious choice in isolation, and only prefer `public=True` because of the specific inconsistency it would create against `CollectionEntry`'s existing unconditional visibility. For Comment 5, it noted that "most users want to see recent additions" doesn't automatically mean date-added should be the *only* view — a real product might want both — but since Comment 5 only asks for a single default sort order for `get_watchlist()`, I kept my response focused on defending that one choice rather than scope-creeping into a multi-sort feature that wasn't requested.
- **Commit hygiene:** Before finalizing, I ran `git log --oneline` past it and asked whether every message matched conventional commit format and whether any commit bundled unrelated changes. It flagged that my original watchlist rename and dedup work were fine as separate commits, but the very first commit inherited from the starter repo ("added watchlist model and endpoint fixed a bug more changes") was not conventional — I used `git rebase -i` to reword it to `feat: add watchlist service and endpoint`.

In every case, the final reasoning in Comments 4 and 5 is my own — grounded in specifics from this codebase (the absence of a visibility field on `CollectionEntry`, the existing `date_added.desc()` convention in `get_collection()`) that a generic AI answer wouldn't have surfaced without being pointed at the actual files.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention used elsewhere (e.g. `add_to_collection()`, `remove_from_collection()` in `services/collection_service.py`). Also updated the docstring's first line from "Save a film..." to "Add a film..." so the docs stay consistent with the new name.

**How I verified:** Ran a project-wide search (`grep -rn "save_to_watchlist"`) across the repo before renaming to find every call site. It turned up exactly two: the function definition in `services/watchlist_service.py` and the import + call in `routes/watchlist/watchlist.py` (`from services.watchlist_service import save_to_watchlist, get_watchlist` and `entry = save_to_watchlist(...)`). Updated both, then re-ran the same search afterward and got zero matches, confirming no call sites were missed. Ran `pytest tests/ -v` — all 4 existing tests still pass since the rename didn't touch behavior.

## Comment 2 — Deduplication
**What I did:** Before reading the reviewer's comment closely, I read `add_to_collection()` in `services/collection_service.py` to see how the project already solves this exact problem for a nearly identical entity (`CollectionEntry` vs `WatchlistEntry`). That function queries for an existing `CollectionEntry` with the same `(user_id, film_id)` pair and raises a dedicated `AlreadyInCollectionError` if one is found, before ever touching the database with an insert. I followed the same pattern in `add_to_watchlist()`: added an `AlreadyInWatchlistError` exception class, and before creating the new `WatchlistEntry` I query `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`. If a matching entry already exists, the function raises `AlreadyInWatchlistError` instead of silently inserting a second row. I also updated `routes/watchlist/watchlist.py` to catch this new exception and return it as a 409 Conflict, mirroring exactly how `routes/collection.py` handles `AlreadyInCollectionError`.

**How I verified:** Compared side-by-side with `services/collection_service.py`'s `add_to_collection()` to confirm the check-then-raise-then-insert order matches. Ran the full test suite (`pytest tests/ -v`) to confirm no regressions, and manually traced through the new code path: call `add_to_watchlist(user, film)` twice in a row — first call creates the entry and returns it, second call hits the `existing` check and raises `AlreadyInWatchlistError` instead of reaching `db.session.add()`. This mirrors `test_add_to_collection_duplicate_raises` in `test_collection.py`, which I used as the model for the equivalent watchlist test I added for Comment 3.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on `tests/test_collection.py`. I reused the same `app`, `sample_user`, and `sample_film` fixtures verbatim (same in-memory SQLite setup, same teardown). The specific test the reviewer asked for, `test_add_to_watchlist_nonexistent_film_raises`, is a line-for-line adaptation of `test_add_to_collection_nonexistent_film_raises`: it calls `add_to_watchlist()` with a fake UUID (`"00000000-0000-0000-0000-000000000000"`) that doesn't correspond to any row in the `Film` table and asserts that `FilmNotFoundError` is raised via `pytest.raises(FilmNotFoundError)`. While I was at it, I also added `test_add_to_watchlist_creates_entry` and `test_add_to_watchlist_duplicate_raises`, mirroring the equivalent collection tests, so the new service has the same baseline coverage as `collection_service.py` rather than just the one test that was explicitly requested.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` — all 3 tests pass. Then ran the full suite `pytest tests/ -v` to confirm the new test file doesn't interfere with the existing collection tests (they use isolated in-memory DBs per test via the `app` fixture, so there's no shared state).

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for new `WatchlistEntry` rows.

**Reasoning:** The key thing I noticed while reading `models.py` is that `CollectionEntry` — the entity for films a user has already watched and rated — has *no* visibility field at all. It's implicitly, unconditionally public: there is no code path anywhere in the app that hides a user's watch history or ratings from anyone else. `WatchlistEntry` is the very first entity in CineLog to introduce a privacy concept. Given that, defaulting a *new* entity to the same effective visibility as every *existing* entity in the app is the behavior that best matches the existing user mental model. A CineLog user has no reason today to believe any part of their activity is private by default — their full watch history and ratings are already exposed with zero opt-out. If `WatchlistEntry` defaulted to `public=False`, we'd be introducing an inconsistency where the thing users have already watched (arguably more revealing) is public, but the thing they merely intend to watch is hidden — that's backwards from a coherence standpoint, and would likely confuse users who assume "this app is public" based on how collections already behave.

Practically, `public=True` also supports the core value CineLog is building toward as a *community* tracking app: watchlists are useful social signal (friends can see what you're planning to watch and coordinate, or get recommendations from what others intend to watch) and that value only exists if lists are visible by default rather than requiring an opt-in most users won't take (defaults are sticky — very few users change a boolean they don't know exists).

**Tradeoff acknowledged:** The real cost of `public=True` is that watchlists can be more personally revealing than a collection entry in one specific way: a watchlist exposes *intent* before the fact (e.g., "I'm about to watch this," which can telegraph things like being behind on a popular/topical film, or aspirational picks a user might feel more exposed by than something they've already watched and closed the loop on). A `public=False` default would optimize for user safety-by-default and data minimization — nothing is shared until a user explicitly chooses to share it, which is generally the safer default when you're not sure how sensitive a new field is. I think that's a reasonable position too, but in CineLog's case it's outweighed by the inconsistency problem above: we'd be protecting the newer, arguably less sensitive data (want-to-watch) while leaving the more sensitive data (already-watched, rated) fully exposed with no equivalent protection. If CineLog later adds privacy controls to `CollectionEntry` as well, I'd revisit this default for both entities together rather than let them diverge.

## Comment 5 — Sort order
**My position:** I agree with the maintainer — I changed `get_watchlist()` to sort by `date_added` descending (most recently added first), replacing the original `Film.title.asc()` alphabetical sort.

**Reasoning:** Beyond agreeing in principle, there's a concrete consistency argument for this in CineLog's own codebase: `get_collection()` in `services/collection_service.py` *already* sorts by `CollectionEntry.date_added.desc()` — "newest first" is the established convention for how this app presents any list of user-film relationships, and there's a test (`test_get_collection_returns_newest_first`) enforcing it. Having the watchlist sort alphabetically while the collection sorts newest-first would mean the two most similar features in the app behave differently for no functional reason, which is exactly the kind of inconsistency a reviewer should catch. Making `get_watchlist()` match `get_collection()`'s ordering isn't just aesthetically consistent — it means a developer who already understands one list endpoint's behavior can correctly predict the other's without re-reading the code.

**Engagement with reviewer's point:** The maintainer's argument was "most users want to see what they added recently," and I think that's correct specifically for a watchlist (more so than it might be for a generic list). A watchlist is an action-oriented, in-progress list — items get added because a user decided "I want to watch this soon," and the most recent additions are the most likely to reflect current intent (something they just heard about or were recommended). Alphabetical order actively works against that: as the list grows, a film added five minutes ago gets buried under everything from A–whatever-comes-before-its-title, forcing the user to scan the whole list to find what's fresh. Newest-first surfaces exactly the thing the user is most likely looking for when they open their watchlist. I added `test_get_watchlist_returns_newest_first` (modeled on `test_get_collection_returns_newest_first`) to lock this behavior in.

## Comment 6 — Rebase
**What conflicted:** I ran `git fetch origin` then `git rebase origin/main`. Git flagged one textual conflict in `.gitignore` (both `main`, via a merged `chore/add-gitignore` PR, and my branch added a `.gitignore` independently) — I merged the two lists together (kept `.pytest_cache/` from my side plus everything from main's version). The rebase then reported "Successfully rebased" with no further conflicts — but that was misleading. `main`'s `refactor: migrate film IDs from integer to UUID` commit had changed `Film.id` from `db.Column(db.Integer, ...)` to `db.Column(db.String(36), default=generate_uuid)`, and — because the watchlist feature hadn't merged yet at the time of that refactor — it had also *removed* the `WatchlistEntry` class from `models.py` entirely (it only existed pre-refactor as leftover scaffolding on the shared root commit). None of my later commits touch the `WatchlistEntry` class definition itself (my commits only ever added a relationship line and reused the class), so git's line-based 3-way merge had nothing to conflict on — it silently produced a `models.py` with no `WatchlistEntry` model at all, which would have broken the entire feature without raising an error.

**How I resolved it:** I re-added the `WatchlistEntry` class to `models.py`, this time matching the post-refactor convention: both `id` and `film_id` are now `db.Column(db.String(36), ...)` (UUID) instead of the old integer `film_id`. I also added a `UniqueConstraint("user_id", "film_id")` to `WatchlistEntry`, matching the same pattern already used on `CollectionEntry` — this gives the deduplication logic from Comment 2 a database-level backstop, not just an application-level check. I then updated `services/watchlist_service.py` and `routes/watchlist/watchlist.py` docstrings/comments that still said `film_id (int)` or `<int>` in the request body example, changing them to `film_id (str): UUID` and `"<uuid>"` to match reality.

**How I verified no conflict remains:** Ran `git log --oneline --merges origin/main..HEAD`, which returned nothing — confirming the branch history is fully linear with no merge commits. Ran the full test suite (`pytest tests/ -v`) — all 8 tests pass, including `test_add_to_watchlist_nonexistent_film_raises`, which passes a fake UUID string (not an integer) as `film_id`, exercising the new UUID column type directly. I also manually re-read the fully reconstructed `models.py` end to end to confirm `WatchlistEntry` matches the shape and conventions of `CollectionEntry` (UUID primary key, UUID foreign key, `date_added`, unique constraint).

## Stretch Features

### remove_from_watchlist()
**What I did:** Added `remove_from_watchlist(user_id, film_id)` to `services/watchlist_service.py`, following the exact pattern of `remove_from_collection()` in `services/collection_service.py`: it looks up the `WatchlistEntry` by `(user_id, film_id)`, and if none exists it raises a new `NotInWatchlistError` (mirroring `NotInCollectionError`) rather than silently no-op'ing — this matches the project's convention of raising a specific, named exception for every "this operation doesn't make sense" case instead of returning `None`/`False` ambiguously. If the entry exists, it's deleted and the function returns `True`, again matching `remove_from_collection()`'s return contract. I also wired up a `DELETE /watchlist/<user_id>/remove` route in `routes/watchlist/watchlist.py`, copying the error-to-status-code mapping used by `routes/collection.py`'s equivalent endpoint (`NotInWatchlistError` → 404). Added two tests: `test_remove_from_watchlist_deletes_entry` (happy path) and `test_remove_from_watchlist_not_present_raises` (the "isn't on the watchlist" case), both modeled on the collection service's test structure.

### Second test
**What I did:** Added `test_get_watchlist_empty_for_new_user`, which asserts `get_watchlist()` returns `[]` for a user who has never added anything. I chose this edge case because it's the actual first-run state every real user starts in — before Comments 1–3 were addressed, nothing in the test suite touched `get_watchlist()` at all, and the sort-order test I added for Comment 5 only ever exercises it with existing entries. An empty-list case is easy to get wrong in subtly different ways (returning `None`, raising on an empty query, or a `.join(Film)` silently excluding a user with zero entries) — asserting the exact `== []` return value pins down the expected "nothing here yet" behavior explicitly rather than leaving it implicit.

### Visibility toggle endpoint
**What I did:** Added a `public` parameter to `add_to_watchlist(user_id, film_id, public=True)`, which is passed straight through to the new `WatchlistEntry(..., public=public)`. The default stays `True`, consistent with the Comment 4 decision above — this isn't a change in default behavior, just an explicit way for callers to opt a specific entry into `public=False` without needing a separate update call. Updated the `POST /watchlist/<user_id>/add` route to accept an optional `"public"` key in the JSON body (`data.get("public", True)`), so a caller can send `{"film_id": "...", "public": false}` to add a private entry directly. Added `test_add_to_watchlist_defaults_to_public` and `test_add_to_watchlist_respects_explicit_public_false` to cover both the default and the override.

## Commit History

`git log --oneline origin/main..HEAD` (branch commits, newest first — linear history, no merge commits):

```
5c35d46 docs: finalize pr-response.md with AI usage, commit log, and PR description
42293f7 feat: add public parameter to add_to_watchlist for explicit visibility control
2462849 test: add edge case for get_watchlist on an empty watchlist
cc0fd3b feat: add remove_from_watchlist with test coverage
ce5a353 docs: document rebase and UUID conflict resolution
0c31add fix: update watchlist model and code to UUID film IDs after main refactor
b5f4b86 docs: document rename, dedup, test, and design decision responses
5edbb04 fix: sort watchlist by date added, newest first
cee5b25 fix: add missing Film-WatchlistEntry relationship for get_watchlist
1d90bf3 test: add test for nonexistent film_id in add_to_watchlist
069e929 fix: add deduplication check to prevent duplicate watchlist entries
4a6b664 fix: rename save_to_watchlist to add_to_watchlist per naming convention
34af857 fix: update film retrieval method to use db.session.get in collection and watchlist services
322b7d2 feat: add watchlist service and endpoint
```

## PR Description

### What this feature does
This PR adds the watchlist feature to CineLog: users can save films they intend to watch (as opposed to `collection`, which tracks films already watched). It adds:

- `POST /watchlist/<user_id>/add` — add a film to a user's watchlist. Body: `{"film_id": "<uuid>", "public": true}` (`public` is optional, defaults to `true`). Returns 404 if the film doesn't exist, 409 if it's already on the watchlist.
- `GET /watchlist/<user_id>` — return a user's watchlist, sorted by date added (newest first).
- `DELETE /watchlist/<user_id>/remove` — remove a film from a user's watchlist. Body: `{"film_id": "<uuid>"}`. Returns 404 if the film isn't on the watchlist.

### Design decisions
- **Default visibility (`public=True`):** New watchlist entries default to public. `CollectionEntry` (watched films) has no privacy control at all in this codebase — it's implicitly, unconditionally public — so defaulting the new `WatchlistEntry.public` field to `True` keeps the watchlist consistent with how every other entity in the app already behaves, rather than introducing an inconsistent, surprising privacy model for just one feature. Callers who want a private entry can pass `public=False` explicitly. Full reasoning and the tradeoff acknowledged in Comment 4 above.
- **Sort order (date-added, newest first):** `get_watchlist()` sorts by `date_added` descending. This matches `get_collection()`'s existing sort order exactly, and better serves what a watchlist is actually for — surfacing what a user most recently decided they want to watch, rather than burying it alphabetically. Full reasoning in Comment 5 above.

### How to manually test
1. Set up the environment and run the app:
   ```
   python -m venv .venv
   source .venv/Scripts/activate   # or .venv/bin/activate on macOS/Linux
   pip install -r requirements.txt
   python app.py
   ```
2. Run the automated test suite: `pytest tests/ -v` — expect 13 passed.
3. Manually exercise the endpoints with curl (replace `<user_id>` / `<film_id>` with real UUIDs from your database, e.g. via `GET /films`):
   - Add a film to the watchlist: `curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add -H "Content-Type: application/json" -d '{"film_id": "<film_id>"}'` — expect `201` with the new entry, `"public": true`.
   - Add the same film again: expect `409` with an "already on this user's watchlist" error.
   - Add a film with an unknown UUID: expect `404` with a "no film found" error.
   - Add a second film explicitly private: `curl -X POST .../add -d '{"film_id": "<film_id_2>", "public": false}'` — expect `"public": false` in the response.
   - View the watchlist: `curl http://127.0.0.1:5000/watchlist/<user_id>` — expect the films returned with the most recently added one first.
   - Remove a film: `curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove -H "Content-Type: application/json" -d '{"film_id": "<film_id>"}'` — expect `200`, then re-run the GET and confirm it's gone.
   - Remove a film that isn't on the watchlist: expect `404`.
