# Hugging Face Open Source Contributions

## Overview of Merged Pull Requests

| PR    | Repository     | Type          | Core Area               | Impact                                                  | Link                                                               |
| ----- | -------------- | ------------- | ----------------------- | ------------------------------------------------------- | ------------------------------------------------------------------ |
| #7831 | datasets       | Bug Fix       | NumPy 2.x Compatibility | Restored stratified splits for NumPy ≥ 2.0              | [View PR](https://github.com/huggingface/datasets/pull/7831)       |
| #3218 | dataset-viewer | Tests         | Cache Layer             | Added unit tests for cache step retrieval               | [View PR](https://github.com/huggingface/dataset-viewer/pull/3218) |
| #3206 | dataset-viewer | Refactor      | Test Infrastructure     | Removed duplicate repo settings logic (−222 LOC)        | [View PR](https://github.com/huggingface/dataset-viewer/pull/3206) |
| #7648 | datasets       | Documentation | Dataset API             | Clarified non in-place transformation behavior          | [View PR](https://github.com/huggingface/datasets/pull/7648)       |
| #7623 | datasets       | Bug Fix       | Dataset Loading         | Prevented unintended fallback to CWD in folder builders | [View PR](https://github.com/huggingface/datasets/pull/7623)       |

---

## Quick Breakdown

* **2 Bug Fixes** (user-facing behavior improvements)
* **1 Documentation Fix** (API clarity)
* **1 Test Coverage Addition**
* **1 Infrastructure Refactor (Large cleanup)**

---
---

## PR: Fix for NumPy 2.0+ ValueError in Stratified Splits
Repository: Hugging Face – datasets

PR: #7831 [Check this out!](https://github.com/huggingface/datasets/pull/7831)

Merged into: main (Oct 28, 2025)

---

### What Was the Issue?

When users called train_test_split() with the stratify_by_column parameter, it failed on NumPy 2.0+ with:

```ValueError: Unable to avoid copy while creating an array```

This happened because NumPy 2.x enforces stricter rules around memory laIt. The stratification column was coming from an Arrow-backed array that is often non-contiguous in memory, and NumPy 2.x refuses implicit unsafe conversions that previously worked in 1.x.

---

### As a result:

- Stratified splitting broke entirely on NumPy ≥ 2.0.

- Existing workflows using class-balanced splits stopped working.

---

### What I Changed

I wrapped the stratification column access in:

```np.asarray(...)```

inside arrow_dataset.py.

---

### Why This Works

- np.asarray() explicitly allows NumPy to create a copy when required.

- This satisfies NumPy 2.x’s stricter memory constraints.

- It maintains compatibility with NumPy 1.x.

---
---
## PR: Add Unit Tests for `get_previous_step_or_raise`

Repository: Hugging Face – dataset-viewer

PR: #3218 [Check this out!](https://github.com/huggingface/dataset-viewer/pull/3218)

Merged into: main (Jul 12, 2025)

---

### What Was the Issue?

Issue #1908 requested proper unit test coverage for the `get_previous_step_or_raise` function.

The function is responsible for retrieving cached step artifacts or raising appropriate exceptions when:

* No cache entry exists
* The cached response indicates an error
* A valid cached response is available

However, it previously lacked direct unit tests validating these behaviors.

---

### As a Result:

* Core cache-handling logic was not explicitly tested
* Error pathways were unverified
* Regression risk existed around cache state handling

---

### What I Changed

I added dedicated unit tests covering:

* Successful cache hit → returns response
* No cache found → raises `CachedArtifactNotFoundError`
* Error status in cache → raises `CachedArtifactError`

The tests use official helper utilities:

* `upsert_response`
* `delete_response`

This ensures consistency with the existing cache infrastructure.

---

### Why This Works

* Validates both happy-path and failure-path behavior
* Ensures correct exception semantics
* Reduces regression risk in cache-layer logic
* Aligns with the project’s testing conventions

Although I couldn’t execute the tests locally due to `libcommon` import resolution constraints, the test logic is clean, deterministic, and fully compatible with the CI environment.

---
---

## PR: Refactor Tests to Use `HfApi.update_repo_settings` for Gated Dataset Setup

Repository: Hugging Face – dataset-viewer

PR: #3206 [Check this out!](https://github.com/huggingface/dataset-viewer/pull/3206)

Merged into: main (Jul 17, 2025)

---

### What Was the Issue?

The test suite contained custom implementations of `update_repo_settings()` inside internal utilities to configure gated datasets during testing.

With `huggingface_hub>=0.25.0`, the official `HfApi.update_repo_settings()` now supports the `gated` parameter directly.

This made the internal re-implementations redundant.

---

### As a Result:

* Duplicate logic existed across multiple test utilities
* Maintenance overhead increased
* Unnecessary imports and helper code cluttered the test setup
* Risk of divergence from official hub behavior

---

### What I Changed

I removed the custom `update_repo_settings()` implementations from:

* `jobs/cache_maintenance/tests/utils.py`
* `services/admin/tests/fixtures/hub.py`
* `services/worker/tests/fixtures/hub.py`

Then I replaced all usages with:

```
HfApi.update_repo_settings(...)
```

Additionally, I cleaned up unused imports:

* `hf_raise_for_status`
* `REPO_TYPES`
* `REPO_TYPES_URL_PREFIXES`

Total impact: **26 additions / 222 deletions**, a net simplification.

Closes: #3063

---

### Why This Works

* Leverages the official API instead of maintaining parallel logic
* Reduces code duplication
* Aligns tests with current `huggingface_hub` capabilities
* Improves long-term maintainability
* Introduces zero functional or behavioral changes

---
---

## PR: Fix Misleading `add_column()` Usage Example in Docstring

Repository: Hugging Face – datasets

PR: #7648 [Check this out!](https://github.com/huggingface/datasets/pull/7648)

Merged into: main (Jul 17, 2025)

---

### What Was the Issue?

The docstring example for `Dataset.add_column()` implied that the method modifies the dataset **in-place**.

In reality, the method returns a **new dataset** with the additional column.
Users must assign the result to a variable to preserve the change.

This mismatch between documentation and behavior caused confusion.

---

### As a Result:

* Users could mistakenly believe their dataset was modified
* Silent logical errors could occur if the returned dataset was not reassigned
* The documentation did not accurately reflect functional semantics

---

### What I Changed

I updated the `add_column()` docstring example to clearly show that:

```
dataset = dataset.add_column(...)
```

is required.

During review, it was pointed out that similar misleading examples existed in other transformation methods. I extended the fix to:

* `select_columns`
* `select`
* `filter`
* `shard`
* `flatten`

All updated to clarify that these methods return new datasets rather than modifying in-place.

Fixes: #7611

---

### Why This Works

* Aligns documentation with actual functional behavior
* Prevents user misunderstanding
* Reduces silent logic bugs in downstream code
* Improves overall API clarity

---
---

## PR: Raise Error in `FolderBasedBuilder` When `data_dir` and `data_files` Are Missing

Repository: Hugging Face – datasets

PR: #7623 [Check this out!](https://github.com/huggingface/datasets/pull/7623)

Merged into: main (Jun 18, 2025)

---

### What Was the Issue?

When calling:

```
load_dataset("audiofolder")
```

without specifying either `data_dir` or `data_files`, the loader would silently fall back to the **current working directory**.

This behavior applied to folder-based builders such as:

* `audiofolder`
* `imagefolder`

Instead of failing early, the system would scan the working directory.

---

### As a Result:

* Long and unnecessary loading times
* Accidental scanning of unrelated files
* Confusing and unpredictable behavior
* No clear feedback to the user

This issue was discussed in #6152.

---

### What I Changed

I added a dedicated validation check inside:

```
FolderBasedBuilder._info()
```

Now, if neither `data_dir` nor `data_files` is provided, a `ValueError` is raised immediately.

The validation is localized within the specific builder class rather than implemented in a generic loader layer.

---

### Why This Works

* Fails fast with a clear, explicit error
* Prevents unintended fallback to the current working directory
* Keeps validation logic scoped to folder-based builders
* Does not affect valid usage paths

This is a user-facing bug fix that improves predictability and prevents silent misconfiguration.
