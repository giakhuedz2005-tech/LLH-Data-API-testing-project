# Optimized API Test Run Flow — DOC Flashcards, AI, Sharing & Export

## 1. Objective

This document defines the optimized execution order for the 47 test cases in `test-case-API-fresher.tsv`.
The execution order prioritizes dependencies, fixture state, and recoverability rather than adhering rigidly to line order in the test sheet.

Core principles:

- Execute read-only and denied mutations before running operations that can alter or delete data.
- Retain `docPatchFlashcardId`, `docDeckId`, `docEmptyDeckId`, and foreign fixtures until all relevant RBAC/BOLA cases are completed.
- Only create `docFlashcardId` immediately before the create → delete → verify omission workflow.
- Each Matrix mutation uses a fresh `docDisposableDeckId` and is cleaned up immediately after collecting sufficient evidence.
- The two workflows `WF-AI-PARTIAL-SAVE` and `WF-SHARE-PUBLIC-LIFECYCLE` must be executed seamlessly according to their internal sequences.
- Do not delete shared parent Folders/Decks before Final Cleanup.
- Do not automatically retry mutations when the commit state is uncertain; reconcile with API/DB first before proceeding.

## 2. Mandatory Run Gates

### Gate A — Run Start

1. Select the correct local/non-production Environment.
2. Verify tokens for Learner A, Learner B, deactivated Learner C, Admin, and invalid token.
3. Check for leftover variables from previous runs; verify resources before clearing variables.
4. Execute the first request for `SCR-COM-PRE-01` to generate `docRunId`, then record `docRunId` into test evidence.
5. Confirm that no other tester/run is concurrently sharing the same fixtures.

### Gate B — Shared Fixture Setup

Create in the following order:

1. Owner Folder → `docFolderId`, `docFolderIdName`.
2. Owner Main Deck → `docDeckId`, `docDeckIdName`.
3. Owner Empty Deck → `docEmptyDeckId`, `docEmptyDeckIdName`.
4. Disposable Deck → `docDisposableDeckId`, `docDisposableDeckIdName` (for mutation Matrix fixtures).
5. Owner PATCH Flashcard → `docPatchFlashcardId` (and another card in `docDeckId` as the source for `normalizedDuplicateTerm`).
6. Foreign Folder for Learner B → `docForeignFolderId`, `docForeignFolderIdName`.
7. Foreign Deck for Learner B → `docForeignDeckId`, `docForeignDeckIdName`.
8. Foreign Flashcard for Learner B → `docForeignFlashcardId`.

Do not create `docFlashcardId` here; `TC-API-DOC-010-01` will create it immediately before the DELETE workflow.

### Gate C — After Every Denied Mutation

After every case expecting `400/401/403/404/409`:

1. Use the correct owner to call GET/list or perform a read-only DB verification.
2. Strictly compare against the pre-request baseline.
3. If state has changed: preserve evidence, mark FAIL/log a defect, and restore via public API before proceeding to the next case.
4. Do not let unexpected mutations pollute fixtures for subsequent test cases.

### Gate D — After Each Data-Creating Case

- `docCaseFlashcardId`: delete by ID after evidence capture and confirm baseline restoration.
- `docSavedFlashcardIds`: delete all exact saved IDs and confirm baseline.
- `docSavedDeckId`: delete the exact returned Deck with `confirmed=true`.
- Matrix: delete parent disposable Deck after evidence capture; do not bulk delete children by ID.
- Only clear variables after cleanup has been verified.

## 3. Optimized Execution Order

### Phase 1 — Read-only and Authentication on Stable Fixtures

| # | Test Case | Objective | Notes |
|---:|---|---|---|
| 1 | `TC-API-DOC-011-01` | List owned empty Deck | List `docEmptyDeckId`; must return an empty array. |
| 2 | `TC-API-DOC-011-02` | List current populated Deck | Capture main Deck baseline and confirm current projection. |
| 3 | `TC-API-DOC-011-05` | B cannot list A Deck | Learner B cannot list Deck of A; verify owner baseline after request. |
| 4 | `TC-API-DOC-011-06` | List Flashcards rejects missing Authorization | Missing auth; no data leakage permitted. |
| 5 | `TC-API-DOC-011-07` | List Flashcards rejects invalid token | Invalid token; no data leakage permitted. |
| 6 | `TC-API-DOC-019-02` | Reject empty Deck | Export empty Deck must be rejected and must not generate a valid file. |
| 7 | `TC-API-DOC-019-05` | B cannot export A Deck | Learner B cannot export Deck of A. |
| 8 | `TC-API-DOC-019-06` | Export rejects missing Authorization | Missing auth cannot export. |
| 9 | `TC-API-DOC-019-07` | Export rejects invalid token | Invalid token cannot export. |
| 10 | `TC-API-DOC-019-01` | Correct read-only export | Successful owner export; verify HTTP, open XLSX manually, and confirm API state is unchanged. |

Rationale: This entire phase must not alter business state, thereby establishing early baselines and detecting auth/read issues prior to mutations.

### Phase 2 — Denied Mutations on Shared Fixtures

| # | Test Case | Objective | Notes |
|---:|---|---|---|
| 11 | `TC-API-DOC-010-07` | B cannot create in A Deck | B cannot create card in Deck A; apply Gate C. |
| 12 | `TC-API-DOC-010-08` | Create Flashcard rejects missing Authorization | Missing auth cannot create card; apply Gate C. |
| 13 | `TC-API-DOC-010-09` | Create Flashcard rejects invalid token | Invalid token cannot create card; apply Gate C. |
| 14 | `TC-API-DOC-012-07` | B cannot edit A card | B cannot edit `docPatchFlashcardId`; capture exact target baseline. |
| 15 | `TC-API-DOC-012-10` | Admin cannot edit Learner Flashcard | Admin cannot edit Learner card; apply Gate C. |
| 16 | `TC-API-DOC-012-11` | Deactivated Learner C cannot edit Flashcard | Deactivated Learner cannot edit card; apply Gate C. |
| 17 | `TC-API-DOC-013-03` | B cannot delete A card | B cannot delete `docPatchFlashcardId`; confirm card remains intact. |
| 18 | `TC-API-DOC-016-06` | B cannot enable A share | Run while sharing is Disabled to demonstrate B cannot enable Deck A. |
| 19 | `TC-API-DOC-010-05` | Reject normalized duplicate in same Deck | Duplicate create; if backend unexpectedly creates card, store returned ID and clean up after evidence. |
| 20 | `TC-API-DOC-012-04` | Reject duplicate term edit | Duplicate PATCH; placed at end of phase due to risk of polluting shared card. If defect recurs, restore exact baseline before next phase. |

### Phase 3 — Positive Mutations with Immediate Restoration

| # | Test Case | Objective | Notes |
|---:|---|---|---|
| 21 | `TC-API-DOC-010-02` | Create valid Chinese Flashcard | Create Chinese card; store `docCaseFlashcardId`, collect evidence, and delete immediately. |
| 22 | `TC-API-DOC-012-01` | Update meaning only | PATCH meaning only; capture actual GET/list baseline, execute, verify omitted fields/SRS, then restore. |
| 23 | `TC-API-DOC-012-02` | Update all three fields | PATCH all 3 fields; use unique term, verify SRS, then restore exact baseline. |

End of phase: `docPatchFlashcardId` must exist and match the designated shared baseline.

### Phase 4 — Isolated Matrix Mutations

| # | Test Case | Objective | Notes |
|---:|---|---|---|
| 24 | `TC-API-DOC-010-03` | Create Flashcard boundary and validation matrix | Create fresh disposable Deck, run complete Matrix 010, collect DB evidence, delete parent Deck. |
| 25 | `TC-API-DOC-012-03` | Update Flashcard boundary and validation matrix | Create disposable Deck + `docMatrixPatchFlashcardId`, run complete Matrix 012, collect content/SRS evidence, delete parent Deck. |

Do not reuse the same physical `docDisposableDeckId` between Matrix 010 and 012.

### Phase 5 — Flashcard Lifecycle Destructive Workflow

These three cases must run consecutively:

| # | Test Case | Objective | Notes |
|---:|---|---|---|
| 26 | `TC-API-DOC-010-01` | Create valid Japanese card | Create lifecycle card and store `docFlashcardId`; do not clean up yet. |
| 27 | `TC-API-DOC-013-01` | Permanently delete owned card | Capture survivor/downstream baseline, then delete exact `docFlashcardId`; script transitions ID to `docDeletedFlashcardId`. |
| 28 | `TC-API-DOC-011-04` | Deleted card omitted from DOC and approved downstream | Confirm deleted ID is omitted from DOC/downstream and survivors remain intact; only then clear `docDeletedFlashcardId`. |

Placing DELETE here prevents premature deletion of `docFlashcardId`. Do not insert other tests between #26–#28.

### Phase 6 — AI Preview and AI Save

| # | Test Case | Objective | Notes |
|---:|---|---|---|
| 29 | `TC-API-DOC-014-01` | Generate Japanese preview | Japanese preview; verify zero-write. Legitimate provider failures are recorded as BLOCKED, not PASS. |
| 30 | `TC-API-DOC-014-02` | Generate Chinese preview | Chinese preview; verify zero-write. |
| 31 | `TC-API-DOC-014-03` | Parse mixed delimiters, trim/ignore blanks/deduplicate | Parsing, trim, blank filtering, and deduplication; verify zero-write. |
| 32 | `TC-API-DOC-014-11` | Successful/partial preview never auto-saves | Preview never auto-saves; compare owner snapshot before/after. |
| 33 | `TC-API-DOC-014-05` | AI Preview boundary and validation matrix | Matrix Preview 014; select only this request in Runner; no business cleanup needed. |
| 34 | `TC-API-DOC-015-01` | Atomic save to existing owned Deck | Save into existing Deck; delete exact `docSavedFlashcardIds` immediately after evidence capture. |
| 35 | `TC-API-DOC-015-02` | Atomic new Deck + cards | Save into new Deck; delete `docSavedDeckId` after evidence capture and verify no orphans remain. |
| 36 | `TC-API-DOC-015-06` | One duplicate rolls back batch | A single duplicate must roll back the entire batch; apply Gate C/no-orphan rule. |
| 37 | `TC-API-DOC-015-08` | Learner A cannot save generated cards to Learner B Deck | A cannot save into Deck B; Learner B verifies foreign baseline remains unchanged. |
| 38 | `TC-API-DOC-015-05` | Generated Save boundary and validation matrix | Matrix Save 015 with fresh disposable Deck; clean up both returned new Deck and disposable Deck. |
| 39 | `TC-API-DOC-014-06` | Continue matching items and report mismatches | Start `WF-AI-PARTIAL-SAVE`; review valid subset and store `selectedFlashcardsJson`. |
| 40 | `TC-API-DOC-015-03` | Save valid selection from partial preview | Run immediately after #39; save exact valid selection, clean up IDs, then clear `selectedFlashcardsJson`. |

#39 and #40 must run consecutively because `TC-API-DOC-015-03` consumes data produced by `TC-API-DOC-014-06`.

### Phase 7 — Sharing State Machine and RBAC Disable

Prior to this phase, confirm `docDeckId` is still populated and sharing is Disabled.

| # | Test Case | Objective | Notes |
|---:|---|---|---|
| 41 | `TC-API-DOC-016-01` | First enable | First enable; store `docShareToken`. |
| 42 | `TC-API-DOC-016-02` | Enable already-enabled Deck | Second enable must be idempotent and retain token. |
| 43 | `TC-API-DOC-018-01` | View active populated share anonymously | Anonymous read of active share; no Authorization and public projection only. |
| 44 | `TC-API-DOC-017-04` | B cannot disable A share | B attempts to disable while share is genuinely active; owner verifies share/token remain unchanged. |
| 45 | `TC-API-DOC-017-01` | Disable active share | Owner disables; move old token to `docOldShareToken`. |
| 46 | `TC-API-DOC-018-04` | Disabled old token | Anonymous access using old token must return 404. |
| 47 | `TC-API-DOC-016-05` | Re-enable creates new token | Re-enable; new token must differ from `docOldShareToken`. |

Do not insert tests that alter sharing state between #41–#47.

## 4. Quick Script Placement Reference

| Request Type | Before Request | Post-Response |
|---|---|---|
| Standard JSON | Request-local initializer if needed | `SCR-JSON-ENV-TEST-01`, then at most one specialized script |
| Matrix 010/012 | matrix guard, then `SCR-MATRIX-PRE-01` | `SCR-MATRIX-ASSERT-01`, then `SCR-MATRIX-FLASHCARD-TEST-01` |
| Matrix 014 | matrix guard, then `SCR-MATRIX-PRE-01` | `SCR-MATRIX-ASSERT-01`, then `SCR-MATRIX-AI-PREVIEW-TEST-01` |
| Matrix 015 | matrix guard, then `SCR-MATRIX-PRE-01` | `SCR-MATRIX-ASSERT-01`, then `SCR-MATRIX-AI-SAVE-TEST-01` |
| List success | initializer | `SCR-JSON-ENV-TEST-01`, then `SCR-LIST-TEST-01` |
| Flashcard create/PATCH success | initializer | `SCR-JSON-ENV-TEST-01`, then `SCR-FLASHCARD-TEST-01` |
| Flashcard DELETE success | initializer | `SCR-JSON-ENV-TEST-01`, then `SCR-DELETE-TEST-01` |
| Standard AI Preview | initializer | `SCR-JSON-ENV-TEST-01`, then `SCR-AI-PREVIEW-TEST-01` |
| Standard AI Save | initializer/request standard script | `SCR-JSON-ENV-TEST-01`, then `SCR-AI-SAVE-TEST-01` |
| Sharing success | initializer | `SCR-JSON-ENV-TEST-01`, then `SCR-SHARE-TEST-01` |
| Public success | initializer | `SCR-JSON-ENV-TEST-01`, then `SCR-PUBLIC-TEST-01` |
| Export XLSX success | initializer | `SCR-EXPORT-TEST-01` only |
| Negative JSON | initializer status/error code | `SCR-JSON-ENV-TEST-01` only |

`SCR-COM-PRE-01` is placed in Collection Pre-request and applies to the entire run.

## 5. Final Cleanup

Execute only after #47 and all required evidence have been completed:

1. If share is Enabled, reconcile/disable according to the cleanup procedure prior to deleting parent.
2. Delete all case-local child records still stored by exact ID.
3. Delete all remaining returned/new Matrix Decks and disposable Decks.
4. Delete owner/foreign namespaced Folders with `confirmed=true` after verifying exact ID/name/run namespace.
5. Confirm that parent and cascaded children no longer exist.
6. Only then clear Collection Variables.
7. If cleanup fails, retain identifying variables for retry/reconciliation.
