# Postman Scripts — LLH Fresher Core

**Project:** Language Learning Hub — DOC Flashcards, AI, Sharing, and Export  
**Execution model:** Manual Postman execution, one tester, sequential runs only  
**Test case source:** `test-case-API-fresher.tsv`  
**Matrix sources:** `TD-MATRIX-010/012/014/015.tsv` and their matching Runner CSV files  
**Status:** Implementation guide for configuring the physical Postman Collection

## 1. Script inventory

| Script ID | Scope | Purpose |
|---|---|---|
| `SCR-COM-PRE-01` | Collection / Before request | Create `docRunId` when it is missing |
| `SCR-MATRIX-PRE-01` | Matrix request / Before request | Load the complete row requestBody and resolve fixture variables |
| `SCR-JSON-ENV-TEST-01` | JSON request / Post-response | Assert status, JSON envelope, error code, and safe error response |
| `SCR-SETUP-TEST-01` | Setup request / Post-response | Capture approved fixture IDs and parent names |
| `SCR-FLASHCARD-TEST-01` | Create/PATCH / Post-response | Assert Flashcard content and capture a requested handoff ID |
| `SCR-LIST-TEST-01` | List / Post-response | Assert list shape, optional count, and optional deleted-ID absence |
| `SCR-DELETE-TEST-01` | Flashcard DELETE / Post-response | Assert DeleteResult and update the lifecycle handoff |
| `SCR-AI-PREVIEW-TEST-01` | AI Preview / Post-response | Assert the exact generated/failed input partition |
| `SCR-AI-SAVE-TEST-01` | AI Save / Before + Post-response | Assert the saved batch and capture child IDs or a new Deck ID |
| `SCR-SHARE-TEST-01` | Share lifecycle / Post-response | Assert state transitions and capture/revoke the share token |
| `SCR-PUBLIC-TEST-01` | Public shared Deck / Post-response | Assert the public allow-list and privacy boundary |
| `SCR-MATRIX-ASSERT-01` | Matrix request / Post-response | Assert status, envelope, materialized body, and expected field length |
| `SCR-MATRIX-FLASHCARD-TEST-01` | Matrix 010/012 / Post-response | Assert Flashcard response fields and capture Flashcard/PATCH handoffs |
| `SCR-MATRIX-AI-PREVIEW-TEST-01` | Matrix 014 / Post-response | Assert the exact AI Preview partition |
| `SCR-MATRIX-AI-SAVE-TEST-01` | Matrix 015 / Post-response | Assert saved batch/target and capture card/Deck handoffs |
| `SCR-EXPORT-TEST-01` | Export success / Post-response | Assert the binary XLSX HTTP contract |

## 2. Postman placement and order

### 2.1 Normal JSON requests

```text
Collection Before request: SCR-COM-PRE-01
Request Before request: request-local initializer, when required
Request Post-response: SCR-JSON-ENV-TEST-01
Request Post-response: one specialized response script, when required
```

### 2.2 Matrix requests

```text
Collection Before request: SCR-COM-PRE-01
Request Before request: matrixId/requestKey initializer
Request Before request: SCR-MATRIX-PRE-01
Request Body / raw / JSON: {{matrixRequestBody}}
Request Post-response: SCR-MATRIX-ASSERT-01
Request Post-response: the one API-specific Matrix script for this request
```

Use this exact mapping:

| Matrix | API-specific response script |
|---|---|
| `TD-MATRIX-010` | `SCR-MATRIX-FLASHCARD-TEST-01` |
| `TD-MATRIX-012` | `SCR-MATRIX-FLASHCARD-TEST-01` |
| `TD-MATRIX-014` | `SCR-MATRIX-AI-PREVIEW-TEST-01` |
| `TD-MATRIX-015` | `SCR-MATRIX-AI-SAVE-TEST-01` |

Do not attach `SCR-JSON-ENV-TEST-01` or the non-Matrix specialized scripts to a Matrix mutation request. `SCR-MATRIX-ASSERT-01` and the selected API-specific Matrix script own those assertions.

The raw Matrix request body must be exactly `{{matrixRequestBody}}`, without quotes or surrounding braces. Each Runner row already contains the complete JSON payload. `SCR-MATRIX-PRE-01` validates that payload and resolves fixture variables; it does not generate or mutate business data.

### 2.3 Export requests

- Successful XLSX response: use `SCR-EXPORT-TEST-01` only.
- JSON error response: use `SCR-JSON-ENV-TEST-01` only.

## 3. Native Postman authorization

| Actor or case | Authorization type | Token value |
|---|---|---|
| Learner A | Bearer Token | `{{learnerAAccessToken}}` |
| Learner B | Bearer Token | `{{learnerBAccessToken}}` |
| Learner C | Bearer Token | `{{learnerCAccessToken}}` |
| Admin | Bearer Token | `{{adminAccessToken}}` |
| Deactivated Learner | Bearer Token | `{{deactivatedLearnerToken}}` |
| Missing authorization | No Auth | None |
| Public endpoint | No Auth | None |

Prefer folder-level authorization with **Inherit auth from parent** for requests that use the same actor.

## 4. Run start and final cleanup

### Run start

1. Select the intended local or non-production Environment.
2. Confirm no other run is active.
3. Inspect residual Collection Variables from the previous run.
4. Verify and reconcile any referenced resource before clearing its variable.
5. Keep tokens and environment configuration unchanged.
6. Send the first request so `SCR-COM-PRE-01` creates `docRunId`.
7. Record `docRunId` in the execution evidence.

### Final cleanup

1. Capture all required evidence.
2. Delete case-local Flashcards referenced by stored ID variables.
3. Delete returned/new Matrix Decks and the disposable Deck when applicable.
4. Delete the owner and foreign namespaced Folders with `confirmed=true` when non-empty.
5. Verify the deleted resources and cascaded children no longer exist.
6. Clear variables only after cleanup has been verified.
7. If cleanup fails, retain all identifying variables for retry or reconciliation.

## 5. Setup fixture order

| Order | Setup request | Stored output | Used by |
|---:|---|---|---|
| 1 | Create Owner Folder | `docFolderId`, `docFolderIdName` | Owner Deck fixtures and new-Deck save |
| 2 | Create Owner Main Deck | `docDeckId`, `docDeckIdName` | Manual Flashcards, sharing, export |
| 3 | Create Owner Empty Deck | `docEmptyDeckId`, `docEmptyDeckIdName` | Empty list/export |
| 4 | Create Disposable Deck | `docDisposableDeckId`, `docDisposableDeckIdName` | Mutation Matrix fixtures (010, 012, 015) |
| 5 | Create Owner PATCH Flashcard | `docPatchFlashcardId` | Non-Matrix PATCH, duplicate, and RBAC cases |
| 6 | Create Foreign Folder | `docForeignFolderId`, `docForeignFolderIdName` | Foreign hierarchy |
| 7 | Create Foreign Deck | `docForeignDeckId`, `docForeignDeckIdName` | BOLA target |
| 8 | Create Foreign Flashcard | `docForeignFlashcardId` | Foreign/BOLA fixture |

Use this disposable-fixture cycle for mutation Matrices. The same setup request and variable name are reused, but each cycle creates a new physical Deck:

1. Before Matrix 010, create an empty `docDisposableDeckId`; run the whole Matrix; delete the Deck through the API after evidence capture.
2. Before Matrix 012, create a new empty `docDisposableDeckId`, then create `docMatrixPatchFlashcardId` inside it; run the whole Matrix; delete the Deck through the API.
3. Before Matrix 015, create another empty `docDisposableDeckId`; run the whole Matrix; delete both the returned new Deck and the disposable Deck through the API.

## 6. Request-local initializers

Only add variables used by that request.

### 6.1 JSON success and error

```javascript
{
  pm.variables.set('expectedStatus', '200');
}
```

```javascript
{
  pm.variables.set('expectedStatus', '404');
  pm.variables.set('expectedErrorCode', 'RESOURCE_NOT_FOUND');
}
```

### 6.2 Named parent setup

```javascript
{
  pm.variables.set('expectedStatus', '201');
  pm.variables.set('storeAs', 'docDeckId');
}
```

### 6.3 Flashcard fixture setup

```javascript
{
  pm.variables.set('expectedStatus', '201');
  pm.variables.set('storeAs', 'docPatchFlashcardId');
  pm.variables.set('expectedDeckId', pm.collectionVariables.get('docDeckId'));
}
```

Matrix 012 fixture:

```javascript
{
  pm.variables.set('expectedStatus', '201');
  pm.variables.set('storeAs', 'docMatrixPatchFlashcardId');
  pm.variables.set('expectedDeckId', pm.collectionVariables.get('docDisposableDeckId'));
}
```

### 6.4 Lifecycle or case-local Flashcard create

Lifecycle create:

```javascript
{
  pm.variables.set('expectedStatus', '201');
  pm.variables.set('expectedDeckId', pm.collectionVariables.get('docDeckId'));
  pm.variables.set('flashcardStoreAs', 'docFlashcardId');
}
```

Independent positive create:

```javascript
{
  pm.variables.set('expectedStatus', '201');
  pm.variables.set('expectedDeckId', pm.collectionVariables.get('docDeckId'));
  pm.variables.set('flashcardStoreAs', 'docCaseFlashcardId');
}
```

### 6.5 PATCH

Copy the exact owner GET/list response immediately before PATCH. Do not guess omitted-field values.

```javascript
{
  pm.variables.set('expectedStatus', '200');
  pm.variables.set('expectedDeckId', pm.collectionVariables.get('docDeckId'));
  pm.variables.set('baselineFlashcardJson', '<exact Flashcard JSON from the latest owner GET/list>');
}
```

### 6.6 List

`expectedListCount` and `expectedAbsentFlashcardId` are optional.

```javascript
{
  pm.variables.set('expectedStatus', '200');
  pm.variables.set('expectedListCount', '1');
  pm.variables.set('expectedAbsentFlashcardId', pm.collectionVariables.get('docDeletedFlashcardId'));
}
```

### 6.7 Flashcard DELETE

```javascript
{
  pm.variables.set('expectedStatus', '200');
  pm.variables.set('deleteIdVariable', 'docFlashcardId');
  pm.variables.set('deletedIdVariable', 'docDeletedFlashcardId');
}
```

For a case-local cleanup request, use `docCaseFlashcardId` and omit `deletedIdVariable`.

### 6.8 AI Preview

The arrays must match the request input after normalization.

```javascript
{
  pm.variables.set('expectedStatus', '200');
  pm.variables.set('expectedNormalizedInputsJson', JSON.stringify(['cat', 'future']));
  pm.variables.set('requiredFailedInputsJson', JSON.stringify([]));
}
```

### 6.9 AI Save

Save to an existing Deck and retain child IDs:

```javascript
{
  pm.variables.set('expectedStatus', '201');
  pm.variables.set('storeSavedFlashcardIds', 'true');
  pm.variables.set('storeSavedDeck', 'false');
}
```

Save to a new Deck and retain the returned Deck ID/name:

```javascript
{
  pm.variables.set('expectedStatus', '201');
  pm.variables.set('storeSavedFlashcardIds', 'false');
  pm.variables.set('storeSavedDeck', 'true');
}
```

### 6.10 Share lifecycle

Use `firstEnable`, `idempotent`, or `reEnable`.

```javascript
{
  pm.variables.set('expectedStatus', '201');
  pm.variables.set('shareMode', 'firstEnable');
}
```

### 6.11 Matrix request

Set the request Body to **raw / JSON** with this exact content:

```text
{{matrixRequestBody}}
```

Then place the following initializer before `SCR-MATRIX-PRE-01`:

```javascript
{
  pm.variables.set('matrixId', 'TD-MATRIX-010');
  pm.variables.set('requestKey', 'PM-REQUEST-010-03');
}
```

## 7. Reusable scripts

Each block has its own lexical scope. Keep the outer braces when multiple blocks share the same Postman tab.

### SCR-COM-PRE-01 — Run namespace

```javascript
{
  if (!pm.collectionVariables.get('docRunId')) {
    const timestamp = Date.now();
    const random = Math.random().toString(36).slice(2, 8);
    pm.collectionVariables.set('docRunId', `AUTO_DOC_${timestamp}_${random}`);
  }
}
```

### SCR-MATRIX-PRE-01 — Complete Matrix body loader

This script does not generate test data. It only validates the selected Matrix and resolves fixture variables inside the row's complete `requestBody`.

```javascript
{
  const rowId = String(pm.iterationData.get('rowId') || 'missing-row');
  const matrixId = String(pm.iterationData.get('matrixId') || '');
  const requestKey = String(pm.iterationData.get('requestKey') || '');

  if (matrixId !== String(pm.variables.get('matrixId') || '')
      || requestKey !== String(pm.variables.get('requestKey') || '')) {
    throw new Error(`[${rowId}] Wrong Matrix file or request.`);
  }

  const resolvedBody = pm.variables.replaceIn(
    String(pm.iterationData.get('requestBody') || '')
  );
  try { JSON.parse(resolvedBody); }
  catch (_) { throw new Error(`[${rowId}] requestBody is not valid JSON.`); }
  pm.variables.set('matrixRequestBody', resolvedBody);
}
```

### SCR-JSON-ENV-TEST-01 — Common JSON contract

```javascript
{
  const expectedStatus = Number(pm.variables.get('expectedStatus'));
  let json = null;
  try { json = pm.response.json(); } catch (_) {}

  pm.test('Exact HTTP status', () => {
    pm.expect(Number.isInteger(expectedStatus)).to.eql(true);
    pm.expect(pm.response.code).to.eql(expectedStatus);
  });

  pm.test('JSON Content-Type', () => {
    pm.expect(pm.response.headers.get('Content-Type') || '').to.match(/^application\/json\b/i);
  });

  pm.test('JSON envelope', () => {
    pm.expect(json).to.be.an('object');
    if (expectedStatus >= 400) {
      const expectedCode = String(pm.variables.get('expectedErrorCode') || '');
      pm.expect(expectedCode).not.to.eql('');
      pm.expect(json).to.have.all.keys('error');
      pm.expect(json.error).to.include.all.keys('code', 'message', 'requestId');
      pm.expect(json.error.code).to.eql(expectedCode);
      pm.expect(json.error.requestId).to.be.a('string').and.not.empty;
      pm.expect(JSON.stringify(json)).not.to.match(
        /Bearer\s|password|secretKey|stack|SQL|filesystem|api[_-]?key/i
      );
    } else {
      pm.expect(json).to.have.property('data');
      pm.expect(json).not.to.have.property('error');
    }
  });
}
```

### SCR-SETUP-TEST-01 — Fixture handoff

```javascript
{
  const parentVariables = new Set([
    'docFolderId', 'docDeckId', 'docEmptyDeckId',
    'docDisposableDeckId', 'docForeignFolderId', 'docForeignDeckId'
  ]);
  const flashcardVariables = new Set([
    'docPatchFlashcardId', 'docMatrixPatchFlashcardId', 'docForeignFlashcardId'
  ]);
  const storeAs = String(pm.variables.get('storeAs') || '');
  const expectedStatus = Number(pm.variables.get('expectedStatus'));
  const runId = String(pm.collectionVariables.get('docRunId') || '');
  const expectedDeckId = String(pm.variables.get('expectedDeckId') || '');
  let data = null;
  try { data = pm.response.json().data; } catch (_) {}

  const allowed = parentVariables.has(storeAs) || flashcardVariables.has(storeAs);
  const hasId = data && typeof data.id === 'string' && data.id.length > 0;
  const parentSafe = parentVariables.has(storeAs)
    && typeof data?.name === 'string'
    && runId.length > 0
    && data.name.includes(runId);
  const flashcardSafe = flashcardVariables.has(storeAs)
    && expectedDeckId.length > 0
    && data?.deckId === expectedDeckId;
  const canStore = pm.response.code === expectedStatus
    && allowed
    && hasId
    && (parentSafe || flashcardSafe);

  pm.test('Fixture is safe to store', () => pm.expect(canStore).to.eql(true));

  if (canStore) {
    pm.collectionVariables.set(storeAs, data.id);
    if (parentVariables.has(storeAs)) {
      pm.collectionVariables.set(`${storeAs}Name`, data.name);
    }
  }
}
```

### SCR-FLASHCARD-TEST-01 — Flashcard content and ID handoff

```javascript
{
  const expectedDeckId = String(pm.variables.get('expectedDeckId') || '');
  const storeAs = String(pm.variables.get('flashcardStoreAs') || '');
  const allowedStores = new Set(['', 'docFlashcardId', 'docCaseFlashcardId']);
  const card = pm.response.json().data;
  let submitted;
  let baseline = null;

  if (!allowedStores.has(storeAs)) throw new Error('[FLASHCARD] Unsupported flashcardStoreAs.');

  try {
    submitted = JSON.parse(pm.variables.replaceIn(pm.request.body.raw || '{}'));
  } catch (_) {
    throw new Error('[FLASHCARD] Invalid submitted JSON.');
  }

  if (pm.request.method === 'PATCH') {
    try { baseline = JSON.parse(String(pm.variables.get('baselineFlashcardJson') || '')); }
    catch (_) { throw new Error('[FLASHCARD] Exact PATCH baseline is required.'); }
  }

  const safeId = typeof card?.id === 'string'
    && card.id.length > 0
    && card.deckId === expectedDeckId;
  const projection = [
    'id', 'deckId', 'term', 'meaning', 'exampleSentence',
    'isMastered', 'createdAt', 'updatedAt'
  ].sort();

  if (storeAs && safeId) pm.collectionVariables.set(storeAs, card.id);

  pm.test('Exact Flashcard projection and target Deck', () => {
    pm.expect(safeId).to.eql(true);
    pm.expect(Object.keys(card).sort()).to.eql(projection);
  });

  pm.test('Submitted content is returned after trim', () => {
    ['term', 'meaning', 'exampleSentence'].forEach((key) => {
      if (Object.prototype.hasOwnProperty.call(submitted, key)) {
        pm.expect(card[key]).to.eql(String(submitted[key]).trim());
      }
    });
  });

  if (baseline) {
    pm.test('PATCH preserves omitted content and Deck', () => {
      ['term', 'meaning', 'exampleSentence'].forEach((key) => {
        if (!Object.prototype.hasOwnProperty.call(submitted, key)) {
          pm.expect(card[key]).to.eql(baseline[key]);
        }
      });
      pm.expect(card.deckId).to.eql(baseline.deckId);
    });
  }

  pm.test('Response excludes raw SRS fields', () => {
    ['reviewLevel', 'nextReview', 'lastRecallRating'].forEach((key) => {
      pm.expect(card).not.to.have.property(key);
    });
  });
}
```

### SCR-LIST-TEST-01 — Lightweight list assertions

```javascript
{
  if (pm.response.code === 200) {
    const data = pm.response.json().data;
    const expectedCountText = String(pm.variables.get('expectedListCount') || '').trim();
    const absentId = String(pm.variables.get('expectedAbsentFlashcardId') || '');

    pm.test('Flashcard list shape', () => {
      pm.expect(data).to.be.an('array');
      const projection = [
        'id', 'deckId', 'term', 'meaning', 'exampleSentence',
        'isMastered', 'createdAt', 'updatedAt'
      ].sort();
      data.forEach((card) => {
        pm.expect(Object.keys(card).sort()).to.eql(projection);
      });
    });

    if (expectedCountText) {
      pm.test('Exact list count', () => {
        pm.expect(data).to.have.length(Number(expectedCountText));
      });
    }

    if (absentId) {
      pm.test('Expected deleted Flashcard is absent', () => {
        pm.expect(data.some((card) => card.id === absentId)).to.eql(false);
      });
    }
  }
}
```

Full ID/content review remains manual unless the Test Case supplies a deterministic expected value.

### SCR-DELETE-TEST-01 — Flashcard DeleteResult handoff

```javascript
{
  const idVariable = String(pm.variables.get('deleteIdVariable') || 'docFlashcardId');
  const deletedIdVariable = String(pm.variables.get('deletedIdVariable') || '');
  const allowedIds = new Set(['docFlashcardId', 'docCaseFlashcardId']);
  const targetId = String(pm.collectionVariables.get(idVariable) || '');
  const data = pm.response.json().data;

  if (!allowedIds.has(idVariable)) throw new Error('[DELETE] Unsupported deleteIdVariable.');

  const deletedSafely = pm.response.code === 200
    && targetId.length > 0
    && data?.deleted === true
    && data?.resourceId === targetId
    && data?.deletedFlashcardCount === 1
    && data?.deletedDeckCount === 0
    && data?.invalidatedShareCount === 0;

  pm.test('DeleteResult confirms the requested Flashcard', () => {
    pm.expect(deletedSafely).to.eql(true);
  });

  if (deletedSafely) {
    if (deletedIdVariable) pm.collectionVariables.set(deletedIdVariable, targetId);
    pm.collectionVariables.unset(idVariable);
  }
}
```

### SCR-AI-PREVIEW-TEST-01 — Exact input partition

```javascript
{
  const dependencyCodes = new Set([
    'AI_DEPENDENCY_FAILED', 'AI_RATE_LIMITED', 'AI_TIMEOUT'
  ]);

  if ([502, 503, 504].includes(pm.response.code)) {
    const error = pm.response.json().error;
    pm.test('Mapped AI dependency failure', () => {
      pm.expect(dependencyCodes.has(error.code)).to.eql(true);
    });
  } else if (pm.response.code === 200) {
    const data = pm.response.json().data;
    const normalize = (value) => String(value).normalize('NFKC').trim().toLowerCase();
    let expectedInputs;
    let requiredFailed;

    try {
      expectedInputs = JSON.parse(
        String(pm.variables.get('expectedNormalizedInputsJson') || '[]')
      ).map(normalize);
      requiredFailed = JSON.parse(
        String(pm.variables.get('requiredFailedInputsJson') || '[]')
      ).map(normalize);
    } catch (_) {
      throw new Error('[AI PREVIEW] Invalid expected input arrays.');
    }

    const generated = data.generatedFlashcards.map((card) => normalize(card.term.split('\n')[0]));
    const failed = data.failedItems.map((item) => normalize(item.input));
    const partition = [...generated, ...failed];

    pm.test('Generated and failed items form the exact input partition', () => {
      pm.expect(partition.slice().sort()).to.eql(expectedInputs.slice().sort());
      pm.expect(new Set(partition).size).to.eql(partition.length);
      requiredFailed.forEach((item) => pm.expect(failed).to.include(item));
    });

    pm.test('generationStatus matches the failed item count', () => {
      pm.expect(data.generationStatus).to.eql(failed.length ? 'partial' : 'completed');
    });
  }
}
```

Review translation quality and naturalness manually. This script checks deterministic contract behavior only.

### SCR-AI-SAVE-TEST-01 — Exact batch and safe handoff

Before request:

```javascript
{
  const storeChildren = String(pm.variables.get('storeSavedFlashcardIds') || 'false') === 'true';
  const storeDeck = String(pm.variables.get('storeSavedDeck') || 'false') === 'true';

  if (storeChildren && pm.collectionVariables.get('docSavedFlashcardIds')) {
    throw new Error('[AI SAVE] Cleanup previous docSavedFlashcardIds before this request.');
  }
  if (storeDeck && pm.collectionVariables.get('docSavedDeckId')) {
    throw new Error('[AI SAVE] Cleanup previous docSavedDeckId before this request.');
  }
}
```

Post-response:

```javascript
{
  if (pm.response.code === 201) {
    const data = pm.response.json().data;
    const storeChildren = String(pm.variables.get('storeSavedFlashcardIds') || 'false') === 'true';
    const storeDeck = String(pm.variables.get('storeSavedDeck') || 'false') === 'true';
    let submitted;

    try {
      submitted = JSON.parse(pm.variables.replaceIn(pm.request.body.raw || '{}'));
    } catch (_) {
      throw new Error('[AI SAVE] Invalid submitted JSON.');
    }

    const safeCards = Array.isArray(data?.flashcards)
      && data.flashcards.length > 0
      && data.flashcards.every((card) =>
        typeof card.id === 'string'
        && card.id.length > 0
        && card.deckId === data.deck?.id
      );
    const safeNewDeck = typeof data?.deck?.id === 'string'
      && data.deck.id.length > 0
      && submitted.newDeckName
      && data.deck.name === submitted.newDeckName;

    if (storeChildren && safeCards) {
      pm.collectionVariables.set(
        'docSavedFlashcardIds',
        JSON.stringify(data.flashcards.map((card) => card.id))
      );
    }
    if (storeDeck && safeNewDeck) {
      pm.collectionVariables.set('docSavedDeckId', data.deck.id);
      pm.collectionVariables.set('docSavedDeckIdName', data.deck.name);
    }

    const signature = (card) => JSON.stringify([
      String(card.term).trim(),
      String(card.meaning).trim(),
      String(card.exampleSentence).trim()
    ]);

    pm.test('Exact saved batch and target Deck', () => {
      pm.expect(safeCards).to.eql(true);
      pm.expect(data.flashcards).to.have.length(submitted.flashcards.length);
      pm.expect(data.flashcards.map(signature).sort())
        .to.eql(submitted.flashcards.map(signature).sort());
      if (submitted.targetDeckId) pm.expect(data.deck.id).to.eql(submitted.targetDeckId);
      if (submitted.newDeckName) pm.expect(data.deck.name).to.eql(submitted.newDeckName);
    });

    pm.test('Saved cards exclude raw SRS fields', () => {
      data.flashcards.forEach((card) => {
        ['reviewLevel', 'nextReview', 'lastRecallRating'].forEach((key) => {
          pm.expect(card).not.to.have.property(key);
        });
      });
    });
  }
}
```

### SCR-SHARE-TEST-01 — Share token lifecycle

```javascript
{
  if ([200, 201].includes(pm.response.code)) {
    const data = pm.response.json().data;
    const mode = String(pm.variables.get('shareMode') || '');
    const previousToken = String(pm.collectionVariables.get('docShareToken') || '');
    const oldToken = String(pm.collectionVariables.get('docOldShareToken') || '');
    const disabling = pm.request.method === 'DELETE';

    pm.test('Exact ShareResult projection', () => {
      pm.expect(data).to.have.all.keys('sharingStatus', 'sharedDeckUrl', 'created');
    });

    if (disabling) {
      const safeDisable = data.sharingStatus === 'Disabled'
        && data.sharedDeckUrl === null
        && data.created === false
        && /^[0-9a-f]{64}$/.test(previousToken);

      pm.test('Valid disable transition', () => pm.expect(safeDisable).to.eql(true));
      if (safeDisable) {
        pm.collectionVariables.set('docOldShareToken', previousToken);
        pm.collectionVariables.unset('docShareToken');
      }
    } else {
      const match = String(data.sharedDeckUrl || '').match(/\/shared-decks\/([0-9a-f]{64})$/);
      const token = match ? match[1] : '';
      const validMode = mode === 'firstEnable'
        ? data.created === true
        : mode === 'idempotent'
          ? data.created === false && token === previousToken
          : mode === 'reEnable'
            ? data.created === true && token !== oldToken
            : false;
      const safeEnable = data.sharingStatus === 'Enabled'
        && /^[0-9a-f]{64}$/.test(token)
        && validMode;

      pm.test('Valid declared enable transition', () => pm.expect(safeEnable).to.eql(true));
      if (safeEnable) pm.collectionVariables.set('docShareToken', token);
    }
  }
}
```

### SCR-PUBLIC-TEST-01 — Anonymous allow-list

```javascript
{
  const data = pm.response.json().data;

  pm.test('Request has no Authorization header', () => {
    pm.expect(pm.request.headers.has('Authorization')).to.eql(false);
  });

  pm.test('Exact PublicDeck projection', () => {
    pm.expect(data).to.have.all.keys('folderName', 'deckName', 'flashcardCount', 'flashcards');
    pm.expect(data.flashcards).to.be.an('array').and.not.empty;
    pm.expect(data.flashcardCount).to.eql(data.flashcards.length);
    data.flashcards.forEach((card) => {
      pm.expect(card).to.have.all.keys('term', 'meaning', 'exampleSentence');
    });
  });

  pm.test('Public response excludes private and internal fields', () => {
    pm.expect(JSON.stringify(data)).not.to.match(
      /ownerId|email|userId|deckId|folderId|flashcardId|reviewLevel|nextReview|lastRecallRating|sharingStatus|shareToken|createdAt|updatedAt/i
    );
  });
}
```

### SCR-MATRIX-ASSERT-01 — Common Matrix assertions

```javascript
{
  const rowId = String(pm.iterationData.get('rowId') || 'missing-row');
  const expectedStatus = Number(pm.iterationData.get('expectedStatus'));
  const expectedCode = String(pm.iterationData.get('expectedErrorCode') || '');
  const assertField = String(pm.iterationData.get('assertField') || '');
  const expectedLength = String(pm.iterationData.get('expectedFieldLength') || '');
  let json = null;
  let body = null;
  let sentBody = null;

  try { json = pm.response.json(); } catch (_) {}
  try { body = JSON.parse(String(pm.variables.get('matrixRequestBody') || '')); } catch (_) {}
  try { sentBody = JSON.parse(pm.variables.replaceIn(String(pm.request.body?.raw || ''))); } catch (_) {}

  pm.test(`[${rowId}] exact HTTP status`, () => {
    pm.expect(pm.response.code).to.eql(expectedStatus);
  });
  pm.test(`[${rowId}] JSON Content-Type`, () => {
    pm.expect(pm.response.headers.get('Content-Type') || '').to.match(/^application\/json\b/i);
  });
  pm.test(`[${rowId}] exact materialized request body`, () => {
    pm.expect(sentBody).to.deep.eql(body);
  });

  if (assertField && expectedLength) {
    pm.test(`[${rowId}] exact ${assertField} length`, () => {
      pm.expect(body?.[assertField]).to.be.a('string')
        .and.have.length(Number(expectedLength));
    });
  }

  if (expectedStatus >= 400) {
    pm.test(`[${rowId}] exact error contract`, () => {
      pm.expect(json).to.have.all.keys('error');
      pm.expect(json.error).to.include.all.keys('code', 'message', 'requestId');
      pm.expect(json.error.code).to.eql(expectedCode);
      pm.expect(json.error.requestId).to.be.a('string').and.not.empty;
      pm.expect(JSON.stringify(json)).not.to.match(
        /Bearer\s|password|secretKey|stack|SQL|filesystem|api[_-]?key/i
      );
    });
  } else if (pm.response.code < 400) {
    pm.test(`[${rowId}] success envelope`, () => {
      pm.expect(json).to.have.property('data');
      pm.expect(json).not.to.have.property('error');
    });
  } else {
    const providerCodes = new Set([
      'AI_DEPENDENCY_FAILED', 'AI_RATE_LIMITED', 'AI_TIMEOUT'
    ]);
    pm.test(`[${rowId}] mapped provider blocker`, () => {
      pm.expect(providerCodes.has(json?.error?.code)).to.eql(true);
    });
  }
}
```

### SCR-MATRIX-FLASHCARD-TEST-01 — Matrix 010/012 response

```javascript
{
  const rowId = String(pm.iterationData.get('rowId') || 'missing-row');
  const apiId = String(pm.iterationData.get('contractApiId') || '');
  const expectedStatus = Number(pm.iterationData.get('expectedStatus'));
  const expectedCount = String(pm.iterationData.get('expectedReturnedCount') || '').trim();
  const expectedDeckId = pm.variables.replaceIn(
    String(pm.iterationData.get('expectedTargetDeckId') || '')
  );
  const body = JSON.parse(String(pm.variables.get('matrixRequestBody') || '{}'));
  let data = null;
  try { data = pm.response.json().data; } catch (_) {}

  function appendId(id) {
    let ids = [];
    try { ids = JSON.parse(String(pm.collectionVariables.get('docMatrixFlashcardIds') || '[]')); }
    catch (_) {}
    if (!ids.includes(id)) ids.push(id);
    pm.collectionVariables.set('docMatrixFlashcardIds', JSON.stringify(ids));
  }

  if (apiId === 'API-DOC-010'
      && pm.response.code === 201
      && typeof data?.id === 'string'
      && data.deckId === expectedDeckId) {
    appendId(data.id);
    pm.collectionVariables.set('docLastMatrixCreatedIdsJson', JSON.stringify([data.id]));
  }

  if (expectedStatus < 400 && [200, 201].includes(pm.response.code)) {
    const projection = [
      'id', 'deckId', 'term', 'meaning', 'exampleSentence',
      'isMastered', 'createdAt', 'updatedAt'
    ].sort();

    pm.test(`[${rowId}] exact Flashcard response`, () => {
      pm.expect(Object.keys(data).sort()).to.eql(projection);
      pm.expect(data.id).to.be.a('string').and.not.empty;
      pm.expect(data.deckId).to.eql(expectedDeckId);
      if (apiId === 'API-DOC-012') {
        pm.expect(data.id).to.eql(
          pm.collectionVariables.get('docMatrixPatchFlashcardId')
        );
      }
      ['term', 'meaning', 'exampleSentence'].forEach((key) => {
        if (Object.prototype.hasOwnProperty.call(body, key)) {
          pm.expect(data[key]).to.eql(String(body[key]).trim());
        }
      });
      if (expectedCount) pm.expect(Number(expectedCount)).to.eql(1);
    });
  }
}
```

### SCR-MATRIX-AI-PREVIEW-TEST-01 — Matrix 014 response

```javascript
{
  const rowId = String(pm.iterationData.get('rowId') || 'missing-row');
  const expectedStatus = Number(pm.iterationData.get('expectedStatus'));
  const expectedCount = String(pm.iterationData.get('expectedPartitionCount') || '').trim();

  if (expectedStatus === 200 && pm.response.code === 200) {
    const body = JSON.parse(String(pm.variables.get('matrixRequestBody') || '{}'));
    const data = pm.response.json().data;
    const normalize = (value) => String(value).normalize('NFKC').trim().toLowerCase();
    const expectedInputs = String(body.vocabularyList || '')
      .split(',').map(normalize).filter(Boolean);
    const generated = data.generatedFlashcards
      .map((card) => normalize(card.term.split('\n')[0]));
    const failed = data.failedItems.map((item) => normalize(item.input));
    const partition = [...generated, ...failed];

    pm.test(`[${rowId}] exact AI Preview result`, () => {
      pm.expect(data).to.have.all.keys(
        'generatedFlashcards', 'failedItems', 'generationStatus'
      );
      pm.expect(partition.slice().sort()).to.eql(expectedInputs.slice().sort());
      pm.expect(new Set(partition).size).to.eql(partition.length);
      if (expectedCount) pm.expect(partition).to.have.length(Number(expectedCount));
      pm.expect(data.generationStatus).to.eql(failed.length ? 'partial' : 'completed');
      data.generatedFlashcards.forEach((card) => {
        pm.expect(card).to.have.all.keys('term', 'meaning', 'exampleSentence');
      });
    });
  }
}
```

### SCR-MATRIX-AI-SAVE-TEST-01 — Matrix 015 response

```javascript
{
  const rowId = String(pm.iterationData.get('rowId') || 'missing-row');
  const expectedStatus = Number(pm.iterationData.get('expectedStatus'));
  const expectedCount = String(pm.iterationData.get('expectedReturnedCount') || '').trim();
  const body = JSON.parse(String(pm.variables.get('matrixRequestBody') || '{}'));
  let data = null;
  try { data = pm.response.json().data; } catch (_) {}

  if (pm.response.code === 201) {
    const cards = Array.isArray(data?.flashcards) ? data.flashcards : [];
    const ids = cards
      .filter((card) => typeof card.id === 'string' && card.id.length > 0)
      .map((card) => card.id);

    if (ids.length === cards.length && ids.length > 0) {
      pm.collectionVariables.set('docLastMatrixCreatedIdsJson', JSON.stringify(ids));
    }
    if (body.newDeckName
        && typeof data?.deck?.id === 'string'
        && data.deck.name === body.newDeckName) {
      pm.collectionVariables.set('docMatrixDeckId', data.deck.id);
      pm.collectionVariables.set('docMatrixDeckIdName', data.deck.name);
    }
  }

  if (expectedStatus === 201 && pm.response.code === 201) {
    const cards = data.flashcards;
    const signature = (card) => JSON.stringify([
      String(card.term).trim(),
      String(card.meaning).trim(),
      String(card.exampleSentence).trim()
    ]);

    pm.test(`[${rowId}] exact AI Save result`, () => {
      pm.expect(data.deck).to.be.an('object');
      pm.expect(cards).to.be.an('array');
      if (expectedCount) pm.expect(cards).to.have.length(Number(expectedCount));
      pm.expect(cards.map(signature).sort()).to.eql(body.flashcards.map(signature).sort());
      cards.forEach((card) => pm.expect(card.deckId).to.eql(data.deck.id));
      if (body.targetDeckId) pm.expect(data.deck.id).to.eql(body.targetDeckId);
      if (body.newDeckName) pm.expect(data.deck.name).to.eql(body.newDeckName);
    });

    pm.test(`[${rowId}] saved cards exclude raw SRS fields`, () => {
      cards.forEach((card) => {
        ['reviewLevel', 'nextReview', 'lastRecallRating'].forEach((key) => {
          pm.expect(card).not.to.have.property(key);
        });
      });
    });
  }
}
```

A mapped AI provider failure still fails `SCR-MATRIX-ASSERT-01` exact status. Record the execution as `BLOCKED`, not PASS.

### SCR-EXPORT-TEST-01 — XLSX HTTP contract

```javascript
{
  const expectedStatus = Number(pm.variables.get('expectedStatus'));
  const disposition = pm.response.headers.get('Content-Disposition') || '';

  pm.test('Exact export status', () => {
    pm.expect(pm.response.code).to.eql(expectedStatus);
  });

  if (expectedStatus === 200) {
    pm.test('Non-empty XLSX with exact headers', () => {
      pm.expect(pm.response.size().body).to.be.above(0);
      pm.expect(pm.response.headers.get('Content-Type')).to.eql(
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
      );
      pm.expect(disposition.startsWith("attachment; filename*=UTF-8''")).to.eql(true);
    });

    pm.test('Stable filename prefix, UTC+7 date, and extension', () => {
      const filename = decodeURIComponent(
        disposition.slice("attachment; filename*=UTF-8''".length)
      );
      const date = new Date(Date.now() + 7 * 60 * 60 * 1000)
        .toISOString()
        .slice(0, 10)
        .replaceAll('-', '');
      pm.expect(filename).to.match(new RegExp(`^deck_.+_${date}\\.xlsx$`));
    });
  }
}
```

Open the workbook manually and verify the worksheet name, column order, numbering, multilingual/multiline content, and absence of private/internal fields. `TBD-EXPORT-NAME-01` remains scoped to hyphen-versus-underscore filename normalization.

## 9. Matrix execution

Each Matrix has two files with different roles:

- `*.tsv` is the readable Test Design source. Do not import it into Postman.
- `*.runner.csv` is the actual 18-column data file for Postman Collection Runner.

To import a Matrix:

1. Confirm the selected Matrix request uses raw JSON body `{{matrixRequestBody}}`.
2. Open the collection or folder in Postman and select **Run collection**.
3. Clear every request except the request named for that Matrix: `PM-REQUEST-010-03`, `PM-REQUEST-012-03`, `PM-REQUEST-014-05`, or `PM-REQUEST-015-05`.
4. In the Runner **Data** area, select **Select File** and choose the matching `*.runner.csv`.
5. Preview the file and confirm 18 columns, the expected row count from section 12, and readable values.
6. Select the correct Postman Environment, then run. Each CSV row becomes one Runner iteration; it is not a separate saved request.

All four validated Runner CSV files can now run in one pass. Matrix 010 and 012 use row-unique, self-contained payloads in a fresh disposable fixture; Matrix 014 is zero-write; Matrix 015 starts from a fresh empty disposable Deck.

Postman automatically checks the response. Persistence, zero-write, no-orphan, and atomicity are checked manually with read-only database evidence. Database access must not be used to alter test state; restore and cleanup through the public API.

### 9.1 Matrix 010

1. Create a new empty `docDisposableDeckId`; configure `PM-REQUEST-010-03` as `POST /decks/{{docDisposableDeckId}}/flashcards`.
2. Capture the empty Deck baseline in the read-only database.
3. Import and run all nine rows of `TD-MATRIX-010.runner.csv` against only `PM-REQUEST-010-03`.
4. `SCR-MATRIX-ASSERT-01` checks the common contract; `SCR-MATRIX-FLASHCARD-TEST-01` checks exact content/Deck and appends every successful returned ID to `docMatrixFlashcardIds`.
5. Confirm the disposable Deck contains exactly the three IDs returned by rows 04, 06, and 08. The other six rows must create nothing.
6. After evidence capture, delete `docDisposableDeckId` once with `confirmed=true` and confirm cascade/non-existence. Do not issue child DELETE requests.

### 9.2 Matrix 012

1. Create a new empty `docDisposableDeckId`, then create one Matrix card inside it and store its ID as `docMatrixPatchFlashcardId`.
2. Configure `PM-REQUEST-012-03` as `PATCH /flashcards/{{docMatrixPatchFlashcardId}}` and capture the card/SRS baseline in the read-only database.
3. Import and run all ten rows of `TD-MATRIX-012.runner.csv` against only `PM-REQUEST-012-03`.
4. Every non-empty Matrix body contains all three content fields with a row-unique valid term, so successful rows are independent from earlier content and no between-row restore is required.
5. `SCR-MATRIX-ASSERT-01` checks the common contract; `SCR-MATRIX-FLASHCARD-TEST-01` checks the exact submitted response, target Deck, and Matrix fixture ID.
6. Confirm only `docMatrixPatchFlashcardId` exists in the disposable Deck, its final content matches the last successful row (09), row 10 made no change, and raw SRS fields match the initial baseline.
7. After evidence capture, delete `docDisposableDeckId` once with `confirmed=true` and confirm cascade/non-existence. No PATCH restore is needed.

### 9.3 Matrix 014

- Run `TD-MATRIX-014.runner.csv` only against `PM-REQUEST-014-05`.
- Use `SCR-MATRIX-ASSERT-01` for the common contract and `SCR-MATRIX-AI-PREVIEW-TEST-01` for the exact generated/failed input partition.
- Capture read-only database evidence before and after the run to confirm no Folder, Deck, Flashcard, share, or SRS record was created or changed for the owner/run scope.
- A mapped provider failure on an accepted row is `BLOCKED`.
- No business-resource cleanup is required.

### 9.4 Matrix 015

- Start with a newly created empty `docDisposableDeckId`, capture its and `docFolderId`'s read-only database baselines, then run all seven rows of `TD-MATRIX-015.runner.csv` against only `PM-REQUEST-015-05`.
- Use `SCR-MATRIX-ASSERT-01` for the common contract and `SCR-MATRIX-AI-SAVE-TEST-01` for batch/target response assertions and safe handoffs.
- For rejected rows, confirm no partial Flashcards or orphan Deck exists in the database.
- New-Deck success stores `docMatrixDeckId` and `docMatrixDeckIdName`.
- Existing-target bulk stores returned IDs temporarily in `docLastMatrixCreatedIdsJson`; confirm the exact returned batch exists in the target Deck.
- Do not issue 50 child cleanup requests. After evidence capture, delete the disposable Deck through the API and confirm cascade/non-existence in the database.

## 10. Matrix verification coverage

| Matrix expectation | Verification |
|---|---|
| `expectedStatus` | Automatic: exact HTTP status |
| `expectedErrorCode` | Automatic: exact error code and safe error envelope |
| `expectedFieldLength` | Automatic: exact materialized field length for accepted and rejected rows |
| `expectedReturnedCount` | Automatic: one returned resource for APIs 010/012; exact batch length for API 015 |
| `expectedPartitionCount` | Automatic: exact AI generated+failed partition |
| `expectedTargetDeckId` | Automatic: exact Deck association |
| Materialized Flashcard content | Automatic: exact trimmed content |
| AI Save batch | Automatic: exact content signatures, count, and target mode |
| Rejected mutation state | Manual: read-only database evidence before/after |
| Positive create/save persistence | Manual: read-only database evidence matched to returned IDs |
| PATCH persistence and SRS preservation | Manual: read-only database field comparison |

## 11. Variable lifecycle

| Variable | Producer | Clear only after |
|---|---|---|
| `docRunId` | `SCR-COM-PRE-01` | Final cleanup is verified |
| Parent ID and Name pairs | `SCR-SETUP-TEST-01` | Parent deletion is verified |
| `docPatchFlashcardId`, `docMatrixPatchFlashcardId`, `docForeignFlashcardId` | `SCR-SETUP-TEST-01` | Owning parent cleanup is verified |
| `docFlashcardId` | `SCR-FLASHCARD-TEST-01` | Lifecycle DELETE succeeds |
| `docCaseFlashcardId` | `SCR-FLASHCARD-TEST-01` | Case-local cleanup succeeds |
| `docDeletedFlashcardId` | `SCR-DELETE-TEST-01` | Absence is verified |
| `docSavedFlashcardIds` | `SCR-AI-SAVE-TEST-01` | Every returned child is deleted and the baseline is restored |
| `docSavedDeckId`, `docSavedDeckIdName` | `SCR-AI-SAVE-TEST-01` | Returned Deck deletion is verified |
| `docMatrixFlashcardIds` | `SCR-MATRIX-FLASHCARD-TEST-01` | Disposable Deck cascade deletion is verified |
| `docMatrixDeckId`, `docMatrixDeckIdName` | `SCR-MATRIX-AI-SAVE-TEST-01` | Returned Matrix Deck deletion is verified |
| `docShareToken`, `docOldShareToken` | `SCR-SHARE-TEST-01` | Share workflow/final cleanup is verified |

## 12. Runner assets

| Matrix | Rows | Runner CSV SHA-256 |
|---|---:|---|
| `TD-MATRIX-010` | 9 | `ab62e03b5a320255c233d59167ad36a993993d18bdb24ca62f93e9d3ac688b4f` |
| `TD-MATRIX-012` | 10 | `16ed2b503c815dae61ded4d7cb0def3ffeaf93b961d434df0575e88216aad363` |
| `TD-MATRIX-014` | 8 | `c0024a906b57dd8c0ea3f206ea6bdef26b34977c23bdbc037cf21f8c86739a35` |
| `TD-MATRIX-015` | 7 | `912f4617a57ef788d25f54a2ad731e34dbb4e8546b79fc4acecf21732fe52ba4` |

Each Runner CSV row contains its literal final request body in `requestBody`. `SCR-MATRIX-PRE-01` only validates JSON and resolves fixture variables such as `{{docDisposableDeckId}}`; it does not generate field content. Before execution, calculate the selected Runner file hash and compare it with this table. Do not merge Matrix CSV files into a single run.
