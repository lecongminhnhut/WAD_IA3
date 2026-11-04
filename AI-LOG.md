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

> **Retrospective documentation note:** Entry 1 was recorded immediately after performing the independent review. Entries 2–4 were documented retrospectively at commit `4776883` as the workflow evolved, based on terminal commands, shell history, and git diffs.

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
- **Version A:** Identified that stock requests inside `products.forEach(async (p) => ...)` at `a.js:52-56` are launched asynchronously without being awaited. As a result, `stockValue.textContent = formatVnd(total)` at line 56 executes immediately before any stock response is received, leaving the displayed value at `0 đ`. It also claimed that unawaited errors from `getStock` would escape the outer `try...catch` and crash the application.
- **Version B:** Identified that `{ timeout: 5000 }` passed to `fetch()` at `b.js:16` is not a standard `RequestInit` property and is silently ignored by the browser Fetch API, preventing the request from aborting after 5 seconds in hang mode. It also noted this would trigger a TypeScript type error `TS2353` under `jsconfig.json`.
- **Version C:** Found no requirement violations; observed that `AbortSignal.timeout(5000)`, `Promise.all`, and `textContent` were correctly implemented.

### What I Accepted, Rejected, or Modified
- **Accepted:** I saved the AI's verbatim response directly into `ASSISTANT-REVIEW.md` to preserve an untouched record. I accepted the findings for Version A (unawaited stock promises) and Version B (ineffective timeout option) as valid hypotheses to investigate.
- **Rejected / Qualified:** I rejected the AI's claim that Version A's stock rejection error would crash standard testing (Claim 2). When I ran the standard test suite (Rows 1–7), stock requests all returned HTTP 200, so this issue never manifested in standard server modes. I only treated it as a theoretical source defect until specifically simulating stock endpoint failures later. I also rejected accepting any code modifications before completing the 27 original test runs.

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

The README explicitly specifies the skill being practiced: *"The one sentence I would tell the assistant. A good sentence names the row, the evidence and the constraint."*

The AI assistant initially suggested a verbose, multi-paragraph prompt with code templates and persona instructions ("Act as me, the student completing this assignment..."). I rejected that verbose approach and instead used the precise one-sentence prompt:

```text
Row 2: stock value displays 0 đ even though all 8 stock requests returned HTTP 200, because forEach(async...) writes the total before requests complete (a.js:52-56). Wait for all requests to finish before updating stock value, but keep requests running concurrently in parallel.
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
- **Rejected:** 
  1. I rejected the AI's verbose multi-paragraph prompt suggestion in favor of the single-sentence prompt naming row, evidence, and constraint as mandated by `README.md`.
  2. I rejected the AI's suggestion to refactor working functions (`formatVnd`, card cloning, status clearing, retry button logic) to keep the commit minimal (+2/-5 lines).

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

Following the required "one sentence I would tell the assistant" format naming row, evidence, and constraint:

```text
Rows 6 and 8: in hang mode the page remains stuck on 'Loading products…' after 7 seconds, and tsc reports TS2353 at b.js:16 because timeout does not exist in RequestInit. Use AbortSignal.timeout(5000) as the abort signal without changing anything else.
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
- **Rejected:** 
  1. I rejected the AI's draft of a bulky retry wrapper with custom setTimeout and AbortController handles, choosing the single-line replacement `signal: AbortSignal.timeout(5000)` (+1/-1 lines).
  2. I rejected touching any other file or starter code.

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
   - 3 Row 8 TypeScript tests verified with genuine compiler output.
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
- For Row 8, initially reported that `npx tsc -p jsconfig.json --checkJs --noEmit` was blocked by PowerShell's execution policy on `npx.ps1`.
- Captured 27 screenshots saved to `evidence/fixed/a-row1.png` through `c-row9.png`.
- Added the Post-fix regression testing section to `VERDICTS.md` without modifying any of the original predictions or historical test observations.

### What I Accepted, Rejected, or Modified
- **Accepted:** I accepted the 24 runtime PASS results and the comprehensive regression verification across all 3 versions.
- **Rejected / Corrected:**
  1. The AI initially assumed `npx tsc -p jsconfig.json --checkJs --noEmit` would run out of the box in the terminal. However, PowerShell blocked `npx.ps1` under the system execution policy (`PSSecurityException`). When the AI suggested modifying PowerShell execution policy or installing global packages, I rejected modifying the student environment. Instead, I resolved the check using `npx.cmd` and the local Node/TypeScript runtime, confirming 0 errors in `a.js`, `b.js`, and `c.js`.
  2. In the initial evidence capture, the AI generated cropped viewport images that produced identical SHA-256 hashes for Rows 1, 2, and 3. I rejected these duplicate files and recaptured genuine, full-page browser screenshots at 1440x1050 showing all 8 cards, VND formatting, and the Scarf card, ensuring all 27 evidence files have unique hashes.

### How I Verified the Result
- I verified that all 27 images in `evidence/fixed/` existed, were readable, and had 27 distinct SHA-256 hashes.
- I verified that `VERDICTS.md` preserved lines 1–160 intact, included the complete original 27-row results table, and documented the post-fix results.
- I checked `git diff --staged` to confirm that only `VERDICTS.md` and `evidence/fixed/` were staged before committing with `test: add post-fix regression evidence for all versions`.
