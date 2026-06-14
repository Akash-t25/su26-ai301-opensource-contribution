# Contribution #1: Implementing ZipStore's `__delitem__` via Overwrite in zarr-python

---

| Field | Details |
|---|---|
| **Contribution #** | 1 |
| **Student** | Akash Tiloda |
| **Project** | [zarr-developers/zarr-python](https://github.com/zarr-developers/zarr-python) |
| **My Fork** | [Akash-t25/zarr-python](https://github.com/Akash-t25/zarr-python) |
| **Issue** | [#828 — Implementing ZipStore's \_\_delitem\_\_ via overwrite](https://github.com/zarr-developers/zarr-python/issues/828) |
| **Status** | Phase III — Complete |

---

## Why I Chose This Issue

My original issue for this contribution was PyLabRobot issue #719 — adding a `preferred_pickup_distance_from_top` attribute to the `Resource` base class. I completed Phase 1 and Phase 2 with that issue, but during Phase 2 I discovered it had been closed by the maintainer: the problem was addressed in PR #872 before I had a chance to contribute. That was a real lesson in open source timing — issues can close between when you claim them and when you're ready to write code. Rather than treating it as a setback, I used it as signal to look more carefully at issue age and PR history before committing to a new one.

For my replacement issue, I landed on zarr-python issue #828. Zarr is a Python library for storing chunked, compressed N-dimensional arrays — the kind of data infrastructure that shows up in climate science, genomics, and ML data pipelines, used by organizations like NASA and the Pangeo project. The issue is a well-scoped enhancement: `ZipStore.delete()` currently raises `NotImplementedError` because ZIP files don't natively support deleting members. The proposed fix rewrites the archive without the deleted entries. Two previous PRs (#1184 and #2838) attempted this but went stale. That history tells me the problem is real and the maintainers want it solved, but it needs someone to pick it up and carry it through.

---

## Understanding the Issue

### Problem Description

Zarr arrays can be backed by different storage backends. One of them is `ZipStore`, which stores array chunks inside a ZIP archive. ZIP files do not natively support deleting individual members, so `ZipStore.delete()` raises `NotImplementedError`. This makes `ZipStore` a second-class backend — any Zarr operation that needs to delete a chunk (e.g., resizing an array) will fail at runtime when backed by a ZIP store.

### Expected Behavior

`ZipStore.delete()` and `ZipStore.delete_dir()` should succeed without raising `NotImplementedError`. The fix rewrites the ZIP archive, copying only the surviving members into a fresh archive and atomically replacing the original. Callers see a consistent view with no trace of the deleted key.

### Current Behavior

- `ZipStore.delete()` raises `NotImplementedError`.
- `supports_deletes` is set to `False`, causing the entire deletion path to be skipped.
- Any Zarr operation requiring chunk deletion fails when using a ZIP-backed store.

### Affected Components

| Component | Role |
|---|---|
| `src/zarr/storage/_zip.py` — `supports_deletes` | Flip from `False` to `True` |
| `src/zarr/storage/_zip.py` — `_rewrite_without()` | New private helper: rewrites archive excluding matched members |
| `src/zarr/storage/_zip.py` — `delete(key)` | Was `NotImplementedError`; now removes the key via archive rewrite |
| `src/zarr/storage/_zip.py` — `delete_dir(prefix)` | Rewrites archive dropping everything under the prefix (single pass) |
| `tests/test_store/test_zip.py` | New tests for delete behavior; updated existing integration test |
| `tests/test_codecs/test_sharding.py` | Added `filterwarnings` marker for now-active `zip` sharding delete test |
| `changes/828.feature.md` | Towncrier changelog entry |

---

## Reproduction Process

### Environment Setup

```bash
git clone https://github.com/Akash-t25/zarr-python.git
cd zarr-python
python3 -m venv env
source env/bin/activate
pip install -e ".[dev]"
pip install pytest hypothesis pytest-asyncio  # not bundled in env by default
```

### Steps to Reproduce

1. In a Python shell with the env active, create a `ZipStore` and attempt to delete a key:
   ```python
   import zarr
   store = zarr.storage.ZipStore("test.zip", mode="w")
   store["chunk"] = b"data"
   del store["chunk"]   # raises NotImplementedError
   ```
2. Observe `NotImplementedError` raised from `src/zarr/storage/_zip.py`.
3. Check `ZipStore.supports_deletes` — it is `False`, confirming deletion is entirely disabled.

### Reproduction Evidence

Screenshots below show `delete()` raising `NotImplementedError` and `supports_deletes = False` in the original source:

![ZipStore delete method - part 1](phase2_a.png)
![ZipStore delete method - part 2](phase2_b.png)

---

## Solution Approach

### Analysis

After reviewing stale PRs #1184 and #2838, the core obstacle is that the `zipfile` module provides no API to remove a member in place. Both prior PRs tried writing `b""` as a soft-delete sentinel, but the maintainers pushed back on sentinel-based approaches because they leak implementation details to callers and complicate the key-listing logic.

The approach that makes the contract clean is an archive rewrite: copy every surviving member into a temporary ZIP in the same directory, then `os.replace()` it over the original. This is atomic on POSIX and keeps the public API straightforward — deleted keys simply do not exist.

Key findings from tracing the codebase:
- `supports_deletes = False` at line 58 causes the base `Store` to skip the deletion path entirely before even calling `delete()`.
- `delete_dir(prefix)` on the base class loops and calls `delete()` per key — for a ZIP that means one full rewrite per key. Overriding it to do a single-pass rewrite is a necessary optimization.
- The archive can accumulate duplicate entries (same name written twice). The rewrite helper must keep only the last entry per name to compact these.
- `ZipStore` must be reopened in `"a"` mode after the rewrite; the original open mode could be `"w"` or `"x"`, which would truncate or fail on next write.

**Files modified:**

| File | Change |
|---|---|
| `src/zarr/storage/_zip.py` | `supports_deletes = True`; added `tempfile`, `Callable` imports; added `_rewrite_without()`; rewrote `delete()` and `delete_dir()` |
| `tests/test_store/test_zip.py` | Updated `test_api_integration`; added `test_store_supports_deletes`, `test_delete_compacts_duplicates`, `test_delete_then_set` |
| `tests/test_codecs/test_sharding.py` | Added `filterwarnings("ignore:Duplicate name")` marker for now-active `zip` sharding delete test |
| `changes/828.feature.md` | Towncrier changelog entry |

### Proposed Solution (High Level)

The fix is built around a private `_rewrite_without(should_delete)` helper in `src/zarr/storage/_zip.py`:

1. **`_rewrite_without(should_delete)`** — collects archive members (keeping only the last entry per name to compact duplicates), writes survivors into a temp ZIP in the same directory, calls `os.replace()` for an atomic swap, then reopens the archive in `"a"` mode. No-ops if no members match `should_delete`.

2. **`delete(key)`** — calls `_rewrite_without(lambda name: name == key)`. Missing key is a no-op, matching the contract of other stores.

3. **`delete_dir(prefix)`** — calls `_rewrite_without(lambda name: name.startswith(prefix))`. Single archive rewrite for the entire prefix, instead of one rewrite per key.

4. **`supports_deletes = True`** — flips the flag so the base `Store` routes deletion to our new methods.

### Implementation Plan (UMPIRE Framework)

| Phase | Step | Description |
|---|---|---|
| **U — Understand** | Read the issue and stale PRs | Reviewed #1184 and #2838; understood why sentinel approach was rejected |
| **M — Match** | Identify the pattern | Archive-rewrite pattern matches maintainer feedback from prior PRs |
| **P — Plan** | Map the change set | Identified 4 files; scoped `_rewrite_without` as the core abstraction |
| **I — Implement** | Write the code | Updated `_zip.py`, updated and added tests, added changelog entry |
| **R — Review** | Self-review and test | 74 passed, 5 skipped in `test_zip.py`; sharding and core tests pass |
| **E — Evaluate** | Open the PR | To be completed in Phase IV |

---

## Testing Strategy

### Unit Tests

All implemented and passing in `tests/test_store/test_zip.py`:

- `test_store_supports_deletes` — verifies `ZipStore.supports_deletes` is `True`.
- `test_delete_compacts_duplicates` — verifies that after writing the same key twice and deleting, the rewrite compacts duplicates and the key is gone.
- `test_delete_then_set` — verifies that a key can be deleted and then re-set successfully.
- `test_zipstore_delitem_removes_key` — verifies `key not in store` after deletion.
- `test_zipstore_delitem_raises_keyerror_for_missing_key` — verifies missing key is a no-op (no exception).
- `test_zipstore_contains_false_after_delete` — verifies `key in store` returns `False` after deletion.
- `test_zipstore_keys_excludes_deleted` — verifies deleted keys do not appear in `store.keys()`.

The inherited `StoreTests.test_delete`, `test_delete_dir`, and `test_delete_nonexistent_key_does_not_raise` now run (and pass) since `supports_deletes` is `True`.

### Integration Tests

- `test_api_integration` in `test_zip.py` — the "assign full chunk to fill value" path now succeeds (chunk gets deleted); `del root["bar"]` now works. Both previously asserted `NotImplementedError`.
- `test_delete_empty_shards[zip]` in `test_sharding.py` — was previously skipped; now active and passing with a `filterwarnings` marker for the expected `"Duplicate name"` ZIP warning.

### Manual Testing

Test results after implementation:

- `tests/test_store/test_zip.py`: **74 passed, 5 skipped** (skips are `*_sync` delete tests — `ZipStore` does not implement `SupportsDeleteSync`, which is out of scope for #828).
- `tests/test_codecs/test_sharding.py` (zip): **49 passed**.
- `test_core.py` + `test_api.py` (zip): **28 passed, 3 skipped**.

Note: `test_group.py` fails to collect with a pre-existing "duplicate parametrization" error unrelated to this change, triggered by the newer pytest version in the environment.

---

## Implementation Notes

### Week 1 Progress

- [x] Identified and researched the issue
- [x] Read zarr documentation and understood the `ZipStore` backend
- [x] Reviewed prior PRs #1184 and #2838 for context
- [x] Confirmed the issue is open and unclaimed

### Week 2 Progress

- [x] Forked the repository
- [x] Cloned fork locally
- [x] Set up virtual environment and installed with `pip install -e ".[dev]"`
- [x] Reproduced the `NotImplementedError` locally
- [x] Traced `ZipStore` class and confirmed `supports_deletes = False`
- [x] Analyzed stale PRs to understand why sentinel approach was rejected

### Week 3 Progress

- [x] Implemented `_rewrite_without()` helper in `src/zarr/storage/_zip.py`
- [x] Flipped `supports_deletes = True`
- [x] Rewrote `delete(key)` using archive rewrite
- [x] Rewrote `delete_dir(prefix)` for single-pass prefix deletion
- [x] Updated `tests/test_store/test_zip.py` — new and updated tests
- [x] Fixed `tests/test_codecs/test_sharding.py` for now-active zip delete test
- [x] Added `changes/828.feature.md` changelog entry
- [x] All relevant tests passing (74 passed, 5 skipped in `test_zip.py`)

### Code Changes

**`src/zarr/storage/_zip.py`**
- `supports_deletes: bool = False` → `True`
- Added `tempfile` import and `Callable` under `TYPE_CHECKING`
- Added `_rewrite_without(should_delete: Callable[[str], bool])` private helper
- Rewrote `delete(key)` — no longer raises `NotImplementedError`
- Rewrote `delete_dir(prefix)` — single-pass archive rewrite instead of per-key loop

**`tests/test_store/test_zip.py`**
- Updated `test_api_integration` to reflect deletion now succeeding
- Added `test_store_supports_deletes`, `test_delete_compacts_duplicates`, `test_delete_then_set`

**`tests/test_codecs/test_sharding.py`**
- Added `filterwarnings("ignore:Duplicate name")` marker for `test_delete_empty_shards[zip]`

**`changes/828.feature.md`**
- Towncrier changelog entry for the feature

### Pull Request

> To be opened in Phase IV after implementation and testing are complete.

---

## Learnings & Reflections

The biggest surprise was that the "obvious" soft-delete approach — writing `b""` as a sentinel — had already been tried twice and rejected. Reading the stale PRs before writing a single line of code saved me from going down the same dead end. That's the real lesson: in open source, the issue comments and closed PRs are as important as the issue itself.

The archive-rewrite approach forced me to think carefully about atomicity (`os.replace()`), compaction (duplicate entries in the same ZIP), and state management (reopening in `"a"` mode after the swap). Each of those was a small but non-obvious detail that only surfaced by tracing the actual `zipfile` module behavior.

---

## Resources Used

- [zarr-python GitHub Repository](https://github.com/zarr-developers/zarr-python)
- [My zarr-python Fork](https://github.com/Akash-t25/zarr-python)
- [Issue #828 — Implementing ZipStore's \_\_delitem\_\_ via overwrite](https://github.com/zarr-developers/zarr-python/issues/828)
- [Prior PR #1184](https://github.com/zarr-developers/zarr-python/pull/1184)
- [Prior PR #2838](https://github.com/zarr-developers/zarr-python/pull/2838)
- [zarr documentation](https://zarr.readthedocs.io)
- Python `zipfile` module documentation
- CodePath AI301 course materials
