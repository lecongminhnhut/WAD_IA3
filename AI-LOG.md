# AI Usage Log — IA#3: Three Answers to One Brief

**Course:** CSC13008 – Web Programming  
**Student:** Lê Công Minh Nhựt  
**Project:** IA#3 – Three Answers to One Brief  

This log is my personal record of using AI assistants during this assignment, documenting each interaction in chronological order.

| # | Date | Purpose | Affected Component |
|---|------|---------|---------------------|
| 1 | 2026-11-04 | Independent review of original implementations A, B, and C against `brief.md` and `README.md` | Versions A, B, C |
| 2 | 2026-11-04 | Assisted fix of the stock calculation race condition in Version A | Version A |
| 3 | 2026-11-04 | Assisted fix of the invalid fetch timeout mechanism in Version B | Version B |
| 4 | 2026-11-04 | Post-fix regression verification across all 27 rows and evidence collection | Versions A, B, C |

---

## Entry 1: Independent Review of Original Versions A, B, and C

- **Date:** 2026-11-04
- **Tool / Model:** Codex (GPT-5 Codex)
- **What I used the AI for:**  
  Before inspecting my own test results or writing predictions in `VERDICTS.md`, I asked Codex to perform an independent source code inspection of `public/versions/a.js`, `b.js`, and `c.js` against the requirements in `brief.md` and `README.md`.

### Prompt I Used

```text
Please independently review the three original JavaScript implementations A, B, and C against the requirements in brief.md and README.md.

For each implementation, identify every potential violation of the requirements.

Pay particular attention to:
- Product loading and rendering.
- Product ordering and formatting.
- Loading and error handling.
- Stock retrieval and total calculation.
- Asynchronous execution and Promise handling.
- Request concurrency and parallel execution.
- Timeout behavior.
- Empty-state handling.
- Retry behavior.
- TypeScript compatibility using the original jsconfig.json.
- Any other requirement in the brief.

For every issue, provide:
1. The affected version.
2. The exact source code location.
3. The violated requirement.
4. Why the implementation is incorrect.
5. The expected correct behavior.
6. A suggested correction.

Also identify implementations or requirements that appear correct.

Do not modify any code. Do not assume that an implementation passes merely because it looks reasonable. Base your review on the actual source code and supplied requirements.
```

### What the AI Suggested
- **Version A:** Identified that stock requests inside `products.forEach(async (p) => ...)` at `a.js:52-56` are launched asynchronously without being awaited. As a result, `stockValue.textContent = formatVnd(total)` at line 56 executes immediately before any stock response is received, leaving the displayed value at `0 đ`. It also noted that errors from unawaited promises would escape the outer `try...catch`.
- **Version B:** Identified that `{ timeout: 5000 }` passed to `fetch()` at `b.js:16` is not a standard `RequestInit` property and is silently ignored by the browser Fetch API, preventing the request from aborting after 5 seconds in hang mode. It also noted this would trigger a TypeScript type error under `jsconfig.json`.
- **Version C:** Found no requirement violations; observed that `AbortSignal.timeout(5000)`, `Promise.all`, and `textContent` were correctly implemented.

### What I Accepted, Rejected, or Modified
- **Accepted:** I saved the AI's verbatim response directly into `ASSISTANT-REVIEW.md` to preserve an untouched record. I accepted the findings for Version A (unawaited stock promises) and Version B (ineffective timeout option) as valid hypotheses to investigate.
- **Rejected / Qualified:** I treated the speculative claim about Version A's stock error handling as unconfirmed, because standard test modes exercise `/api/products` failures rather than stock endpoint errors. I did not accept any code modifications at this stage.

### How I Verified the Result
After preserving the review, I tested the three original versions in the browser and compared the review claims against my actual test evidence:
- Confirmed Version A Row 2 failure: `evidence/a-row2.png` showed the stock total permanently stuck at `0 đ`.
- Confirmed Version B Row 6 failure: `evidence/b-row6.png` showed the page still loading after 7+ seconds in hang mode.
- Confirmed Version B Row 8 failure: `evidence/b-row8.png` showed TypeScript error `TS2353` for `timeout` on line 16.
- Confirmed Version C: all rows 1–9 passed without issues (`evidence/c-row1.png` to `c-row9.png`).

---

## Entry 2: Fixing Version A Stock Calculation Race Condition

- **Date:** 2026-11-04
- **Tool / Model:** Codex (GPT-5 Codex workspace inspection)
- **What I used the AI for:**  
  I used Codex to fix the confirmed stock calculation race condition in `public/versions/a.js`.

### Prompt I Used

```text
Act as me, the student completing this assignment. I want you to independently review and fix Version A based on the assignment requirements and the previous AI review.

Before modifying any code, carefully read brief.md, ASSISTANT-REVIEW.md, VERDICTS.md, AI-LOG.md, README.md and relevant project configuration files, and public/versions/a.js. Identify every confirmed issue in Version A and distinguish confirmed failures from potential issues that have not been demonstrated.

Only modify public/versions/a.js. Fix the confirmed stock calculation race condition so stock requests remain concurrent, all requests are awaited, and the displayed total is correct. Handle request failures appropriately, preserve existing correct behavior, and avoid unnecessary refactoring. Inspect the other potential issue and only fix it if it is established as a requirement violation.

After modifying Version A, run the relevant tests, reproduce row 2, confirm 12.360.000 đ, verify concurrency, check error and loading behavior, run npx tsc -p jsconfig.json --checkJs --noEmit, and recheck the other Version A cases. Keep historical screenshots unchanged.

Add a new AI-LOG entry for this actual modification. Before committing, inspect git diff and git status. Commit only public/versions/a.js with:

fix: resolve version A stock calculation race condition

Do not modify Version B or Version C and do not begin the final full regression test.
```

### What the AI Suggested
Codex proposed replacing the unawaited `products.forEach(async (p) => ...)` loop with `Promise.all`:
```javascript
const stocks = await Promise.all(products.map((p) => getStock(p.id)));
const total = products.reduce((sum, p, i) => sum + p.price * stocks[i], 0);
stockValue.textContent = formatVnd(total);
```

### What I Accepted, Rejected, or Modified
- **Accepted:** I accepted the `Promise.all` solution because it keeps stock requests concurrent (dispatched in parallel) while ensuring all responses resolve before computing and displaying the total. Because `Promise.all` is awaited inside the existing `try` block, any network failure is automatically caught by the existing catch handler.
- **Rejected:** I rejected any broader refactoring of working parts (card rendering, price formatting, or retry handling). I restricted changes strictly to `public/versions/a.js`.

### How I Verified the Result
- I re-tested Version A under normal mode: all eight stock requests launched concurrently (<0.5 ms dispatch time), and the stock total displayed `12.360.000 đ` correctly.
- I re-tested slow, error, hang, and empty modes to confirm no regressions were introduced.
- I committed the fix with `git commit -m "fix: resolve version A stock calculation race condition"`.

---

## Entry 3: Fixing Version B Timeout Handling

- **Date:** 2026-11-04
- **Tool / Model:** Codex (GPT-5 Codex workspace inspection)
- **What I used the AI for:**  
  I used Codex to fix the invalid `{ timeout: 5000 }` fetch option in `public/versions/b.js`.

### Prompt I Used

```text
Continue fixing Version B.

First, read ASSISTANT-REVIEW.md, VERDICTS.md, brief.md, jsconfig.json, and public/versions/b.js to understand the confirmed issues.

You are allowed to modify exactly one file: public/versions/b.js. Do not modify, create, delete, rename, or reformat any other file, including AI-LOG.md, VERDICTS.md, ASSISTANT-REVIEW.md, screenshots, evidence, configuration files, Version A, or Version C.

Fix the confirmed Version B issues:
1. Replace the invalid { timeout: 5000 } fetch option with a valid timeout mechanism such as AbortSignal.timeout(5000), and ensure a hanging request is aborted and handled correctly.
2. Remove the invalid timeout property from RequestInit and ensure the implementation passes the project TypeScript check.

Preserve product rendering, ordering, price formatting, loading and error states, empty state, retry behavior, concurrent stock requests, and stock calculation. Do not rewrite unrelated code.

Run the relevant Version B tests, especially Row 6 and Row 8, check regressions, inspect git status and git diff, and commit only public/versions/b.js with:

fix: implement valid timeout handling in version B
```

### What the AI Suggested
Codex suggested replacing line 16:
```javascript
// Before
const response = await fetch('/api/products', { timeout: 5000 });

// After
const response = await fetch('/api/products', { signal: AbortSignal.timeout(5000) });
```

### What I Accepted, Rejected, or Modified
- **Accepted:** I accepted `AbortSignal.timeout(5000)` because it is the web-standard way to time out fetch requests in modern browsers. It resolves the Row 6 hang failure (aborts after 5 seconds) and eliminates the invalid property from `RequestInit`, fixing the Row 8 TypeScript error `TS2353`.
- **Rejected:** I rejected touching any other file or refactoring existing rendering/calculation logic in `b.js`.

### How I Verified the Result
- I tested Version B in hang mode: after approximately 5 seconds, the request was canceled, status changed to `Could not load products.`, and the Retry button appeared.
- I checked TypeScript diagnostics: line 16 no longer showed any type error in VS Code.
- I verified normal, slow, error, and empty modes to confirm no regressions.
- I committed the fix with `git commit -m "fix: implement valid timeout handling in version B"`.

---

## Entry 4: Post-Fix Regression Testing and Evidence Capture

- **Date:** 2026-11-04
- **Tool / Model:** Codex (GPT-5 Codex workspace inspection)
- **What I used the AI for:**  
  After fixing Versions A and B, I used Codex to run a comprehensive post-fix regression test across all 27 rows (9 rows each for Version A, B, and C), record the observations in `VERDICTS.md`, and capture screenshots into `evidence/fixed/`.

### Prompt I Used

```text
Proceed with the documentation and evidence commit.

Before committing:
1. Verify that VERDICTS.md preserves all original predictions and historical test results.
2. Verify that all 27 post-fix test rows are documented:
   - 24 runtime tests marked PASS.
   - 3 Row 8 TypeScript tests marked INCOMPLETE, with the environment limitation explained.
3. Verify that the new screenshots exist under evidence/fixed/.
4. Inspect git status and git diff.
5. Confirm that no source code, AI logs, or unrelated files are included.

Stage ONLY:
- VERDICTS.md
- evidence/fixed/

Commit using:
test: add post-fix regression evidence for all versions
```

### What the AI Produced
- Tested all 27 rows across the three versions against `http://localhost:5050`.
- Confirmed that all 24 runtime tests passed (normal loading, stock calculation, escaping, slow mode, error/retry recovery, timeout, console cleanliness, and empty state).
- For Row 8, reported that `npx tsc -p jsconfig.json --checkJs --noEmit` could not be executed because the local PowerShell execution policy blocked `npx.ps1`, and offline environment restrictions prevented downloading `tsc`.
- Captured 27 screenshots saved to `evidence/fixed/a-row1.png` through `c-row9.png`.
- Added the Post-fix regression testing section to `VERDICTS.md` without modifying any of the original predictions or historical test observations.

### What I Accepted, Rejected, or Modified
- **Accepted:** I accepted the 24 runtime PASS results and the generated screenshots under `evidence/fixed/`.
- **Preserved Limitation:** For Row 8, I insisted on keeping the verdict marked as **INCOMPLETE** rather than claiming PASS, because the project-level TypeScript command could not run to completion in this terminal environment.
- **Protected History:** I ensured the original test evidence in `evidence/` remained untouched so that historical test records were preserved.

### How I Verified the Result
- I verified that all 27 images in `evidence/fixed/` existed and were readable.
- I verified that `VERDICTS.md` preserved lines 1–160 intact.
- I checked `git diff --staged` to confirm that only `VERDICTS.md` and `evidence/fixed/` were staged before committing with `test: add post-fix regression evidence for all versions`.
