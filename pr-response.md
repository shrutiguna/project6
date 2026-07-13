# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

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
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## Stretch Features

### remove_from_watchlist()
**What I did:**

### Second test
**What I did:**

### Visibility toggle endpoint
**What I did:**

## Commit History

<!-- git log --oneline screenshot goes here -->

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
