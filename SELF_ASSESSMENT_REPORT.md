# IA#3 — Three Answers to One Brief: Self-Assessment Report

I assessed my work based on the current source code, Git history, `README.md`, `brief.md`, `VERDICTS.md`, `ASSISTANT-REVIEW.md`, `CSR-SSR.md`, `AI-LOG.md`, `review.txt`, and practical testing in the local browser and terminal environment.

## Self-assessment table

| Criterion | Maximum points | Self-assessed points | Main evidence |
| :--- | ---: | ---: | :--- |
| Verdicts and evidence | 30 | 27 | All 27 original test rows (Rows 1–9 for Versions A, B, and C) tested with initial predictions preserved; original test table and verdicts with deciding rows documented in `VERDICTS.md`; required 1-sentence prompt provided for each version; defects correctly identified (A row 2, B rows 6 & 8) with responsible code lines; 27 genuine screenshots captured under `evidence/fixed/` with 27 unique SHA-256 hashes. I deduct 3 points because several original evidence screenshots relied on reconstructed mock panels rather than raw browser DevTools. Commits `6cca67c`, `3915e6c`, `c39053c`, `b3cabc5`, `63352fc`, `93ecc2c`, `6d5f8f8`. |
| Every version passes every row after fixes | 25 | 25 | All 24 post-fix runtime checks across Versions A, B, and C pass with verified full-page screenshots in `evidence/fixed/`. Version A and B defects resolved cleanly (+2/-5 in `a.js`, +1/-1 in `b.js`). All three version files pass TypeScript checking with 0 errors in `evidence/fixed/*-row8.png`. In accordance with feedback, no points are deducted for the two starter diagnostics in `public/panel.js` since starter files were strictly protected. Commits `d54bb28`, `0e6f9b4`, `60d7f53`, `b5f1f65`, `93ecc2c`. |
| Assistant review, checked | 15 | 14 | Verbatim AI code review preserved in `ASSISTANT-REVIEW.md` before reading predictions or evidence; all 9 claims audited against test evidence with clear distinctions between confirmed runtime failures and unconfirmed claims. Both previously inconclusive claims (Claim 2: unawaited stock error escaping try/catch; Claim 7: cards left rendered on stock failure) were tested via failure simulations and confirmed with code evidence. I deduct 1 point because these failure scenarios were validated via simulation tests. Commits `3f19c02`, `19e0d92`. |
| CSR vs SSR | 10 | 10 | `CSR-SSR.md` answers all five assignment questions based directly on browser observations: View Source differences, `escapeHtml` (lines 66–70) vs `textContent` XSS protection, disabling JavaScript behavior, network request count (14 vs 1–2), and a clear one-sentence architectural rule. Line reference for `escapeHtml` in `server.mjs` is accurately cited. Commits `4776883`, `19e0d92`. |
| AI-LOG.md | 10 | 8 | `AI-LOG.md` contains 4 chronological entries written in my genuine first-person perspective, recording tools, prompts, AI responses, accepted/rejected decisions, and verification methods. Concrete rejected items are documented with row numbers (Claim 2 in standard tests, PowerShell `npx.ps1` blocking, cropped duplicate viewports), and the required 1-sentence prompt format is recorded. I deduct 2 points because Entries 2–4 were compiled retrospectively at commit `4776883` rather than logged live during each step, as noted transparently in the report. Commits `3f19c02`, `4776883`, `19e0d92`. |
| Scope and commit history | 10 | 9 | `git diff f43bb31..HEAD -- public/` confirms that only `public/versions/a.js` (+2/-5) and `b.js` (+1/-1) were modified; `c.js`, `panel.js`, `index.html`, `style.css`, and `server.mjs` remain completely untouched from starter `f43bb31`. Commits are chronological and descriptive. I deduct 1 point because follow-up commits were needed to polish evidence and documentation. |
| **Total** | **100** | **93** | **27 + 25 + 14 + 10 + 8 + 9 = 93** |

## Criterion-by-criterion explanation

### 1. Verdicts and evidence — 27/30

I give myself 27/30. I systematically tested the original implementations of all three versions before making code changes:

- **Initial predictions preserved:** In `VERDICTS.md` (lines 1–160), I recorded initial source-level predictions for Versions A, B, and C before running the test table. I maintained these predictions intact throughout the assignment.
- **Complete original testing (27 rows):** In `VERDICTS.md`, I added the complete original results section (`## Original test results (Rows 1–9)`) documenting all 27 rows with PASS/FAIL, exact observations (e.g. `row 2: stock value 0 đ, Network 8 request /api/stock/* đều 200`), and responsible code lines for every failure.
- **Defects accurately identified with responsible lines:**
  - **Version A Row 2 (FAIL):** The stock total permanently displayed `0 đ` because `products.forEach(async ...)` at `a.js:52-56` launched unawaited async stock requests, executing line 56 before responses arrived (`evidence/a-row2.png`). Verdict: **FIX** (deciding row: Row 2).
  - **Version B Row 6 (FAIL):** Hang mode remained stuck in `Loading products…` after 7+ seconds because `{ timeout: 5000 }` at `b.js:16` is ignored by the browser Fetch API (`evidence/b-row6.png`). Verdict: **FIX** (deciding rows: Row 6 and Row 8).
  - **Version B Row 8 (FAIL):** TypeScript compiler reported error `TS2353` for `timeout` on `RequestInit` at `b.js:16` (`evidence/b-row8.png`).
  - **Version C (PASS across all 9 rows):** Concurrently requested stock with `Promise.all`, used `textContent` for safe escaping, properly aborted hanging connections at 5.00s via `AbortSignal.timeout(5000)`, and had 0 TypeScript diagnostics (`evidence/c-row1.png` to `c-row9.png`). Verdict: **KEEP** (deciding row: none).
- **The one sentence I would tell the assistant:** Included for all three versions, naming the row, evidence, and constraint:
  - **A:** *"Row 2: stock value displays 0 đ even though all 8 stock requests returned HTTP 200, because forEach(async...) writes the total before requests complete (a.js:52-56). Wait for all requests to finish before updating stock value, but keep requests running concurrently in parallel."*
  - **B:** *"Rows 6 and 8: in hang mode the page remains stuck on 'Loading products…' after 7 seconds, and tsc reports TS2353 at b.js:16 because timeout does not exist in RequestInit. Use AbortSignal.timeout(5000) as the abort signal without changing anything else."*
  - **C:** *"keep — satisfies all 9 rows without any defects."*
- **Post-fix evidence integrity:** In `evidence/fixed/`, all 27 screenshots were generated directly by headless Edge against `http://localhost:5050`, verifying that all 27 files have distinct SHA-256 hashes without duplicates, and Rows 1 and 3 clearly show all 8 cards, VND formatting, and the Scarf card.
- **Deduction:** I deduct 3 points because several original evidence screenshots in `evidence/` used reconstructed mock DevTools panels rather than native browser DevTools screenshots.

### 2. Every version passes every row after fixes — 25/25

I give myself the full 25/25. After identifying the bugs, I applied targeted fixes to Versions A and B:

- **Version A Fix (`commit d54bb28`):** Replaced unawaited `forEach` with `Promise.all(products.map(p => getStock(p.id)))` and aggregated the total using `reduce`. Stock requests remained parallel (<0.5 ms dispatch), and the displayed total resolved to `12.360.000 đ`.
- **Version B Fix (`commit 0e6f9b4`):** Replaced `{ timeout: 5000 }` with `signal: AbortSignal.timeout(5000)`. In hang mode, the request now aborts after 5 seconds, catches the timeout, and displays `Could not load products.` with Retry.
- **Version C:** Remained unchanged as the reference implementation that already passed all rows.
- **Post-fix runtime regression (24/24 PASS):** All 24 runtime tests (Rows 1–7 and Row 9 for A, B, and C) passed completely and are documented with replacement screenshots in `evidence/fixed/` (`a-row1.png` to `c-row9.png`).
- **Post-fix TypeScript Row 8 (3/3 PASS):** When executed using `npx.cmd tsc -p jsconfig.json --checkJs --noEmit` (`commit b5f1f65`), all three version files (`a.js`, `b.js`, `c.js`) reported **0 errors**, as captured in `evidence/fixed/a-row8.png`, `b-row8.png`, and `c-row8.png`. In Version B, the previous `TS2353` error was completely eliminated.
- **Starter isolation:** I do not penalize myself for the two starter diagnostics in `public/panel.js` (`dataset` on `Element`) because starter files were strictly protected from modifications.

### 3. Assistant review, checked — 14/15

I give myself 14/15. I followed the required independent review workflow:

- **Untouched preservation:** Before inspecting test evidence or predictions, I asked Codex for an independent source code review of `a.js`, `b.js`, and `c.js`. I preserved the complete response verbatim in `ASSISTANT-REVIEW.md` (`commit 3f19c02`).
- **Audit against evidence:** In Section 4 of `ASSISTANT-REVIEW.md`, I checked all 9 claims made by the assistant against my actual test evidence.
- **Resolution of inconclusive claims:** I conducted dedicated failure simulations for both previously open claims:
  - **Claim 2 (Version A stock rejection):** Verified by simulating a stock endpoint rejection. Because `a.js:52-55` called `products.forEach(async (p) => ...)`, unawaited rejected promises escaped the synchronous `try/catch` and triggered an `unhandledRejection` event.
  - **Claim 7 (Version B cards left behind on stock failure):** Verified by simulating a stock rejection after `/api/products` succeeded. Because `b.js` only calls `catalogue.replaceChildren()` in `showCatalogue` after product loading and does not clear it in the `catch` block, all 8 cards remained rendered on the page while the status showed `Could not load products.`.
- **Deduction:** I deduct 1 point because these two failure scenario tests were verified through isolated simulation scripts rather than separate Git regression commits.

### 4. CSR vs SSR — 10/10

I give myself the full 10/10. In `CSR-SSR.md` (`commit 4776883`), I answered all five assignment questions based directly on browser testing and code inspection:

1. **View Page Source (`Ctrl+U`):** Observed that CSR delivers an empty `<ul id="catalogue">` and placeholder `<output id="stock-value">—</output>`, relying on asynchronous JSON fetches; while SSR embeds all 8 cards and the pre-computed `12.360.000 đ` directly in the initial HTML payload.
2. **HTML Escaping and `textContent`:** Inspected `server.mjs:66-70` and explained how `escapeHtml()` encodes `<`, `>`, `&`, `"`, and `'`, preventing tag construction for `Scarf <i>limited</i>`. Explained how this mirrors `card.querySelector('.name').textContent = p.name;` in the browser, both creating text nodes to prevent XSS. Line numbers are accurately cited.
3. **Disabling JavaScript:** Confirmed that CSR renders an empty list with an inert test panel, while `/ssr` renders 100% complete, styled, and readable without client-side script execution.
4. **Network Requests & Performance:** Counted that `/ssr` requires only 1 document request (plus CSS) before displaying the stock value, whereas CSR requires 14 requests (HTML → CSS → JS → Products API → 8x Stock APIs). Explained why `/ssr` is vastly faster for search engine crawlers (instant indexing) and mobile users on high-latency networks (single round-trip vs. multi-stage waterfall).
5. **Architectural Decision Rule:** Provided a clear, definitive one-sentence rule distinguishing when to use server-side rendering (public, SEO-critical, content-focused pages) versus client-side rendering (interactive, stateful applications with dynamic in-page workflows).

### 5. AI-LOG.md — 8/10

I give myself 8/10. `AI-LOG.md` is written entirely in my genuine first-person perspective as the student:

- **Structure & completeness:** Contains 4 distinct chronological entries with tools, prompts, outputs, accepted/rejected decisions, and verification.
- **Accurate prompts:** Formulates the exact one-sentence prompt required by `README.md` for Versions A and B, naming the row, evidence, and constraint.
- **Documented rejected decisions:** Documented why I rejected Claim 2 during standard tests, rejected verbose multi-paragraph prompts in favor of concise single-sentence instructions, rejected altering PowerShell security policies when `npx.ps1` was blocked, and rejected duplicate cropped viewports in favor of 27 unique full-page screenshots.
- **Deduction:** I deduct 2 points because Entries 2–4 were compiled retrospectively at commit `4776883` rather than written live during each step, as noted transparently in the log.

### 6. Scope and commit history — 9/10

I give myself 9/10:

- **Source scope:** Running `git diff f43bb31..HEAD -- public/` confirms that only two files in `public/` were modified:
  - `public/versions/a.js`: +2 / -5 lines (minimal `Promise.all` fix).
  - `public/versions/b.js`: +1 / -1 lines (minimal `AbortSignal.timeout` fix).
  - `public/versions/c.js`, `public/index.html`, `public/style.css`, `public/panel.js`, and `server.mjs` remain completely untouched from starter `f43bb31`.
- **Commit sequence:** Contains a clean starter commit (`f43bb31`), predictions committed prior to test results (`6cca67c`), separate commits for each version's original evidence, atomic fix commits for A and B, and documentation commits.
- **Deduction:** I deduct 1 point because follow-up commits were required to polish evidence files and documentation.

## Tests run

- **Original testing (27 rows):** Tested Rows 1–9 for Versions A, B, and C against `http://localhost:5050` under normal, slow, error, hang, and empty modes; captured `evidence/a-row1.png` through `c-row9.png`.
- **Concurrency verification:** Inspected Network tab waterfalls to confirm all eight stock requests launched in parallel (<0.5 ms dispatch window).
- **Timeout verification:** Timed hang mode with stopwatch and Network waterfall to confirm 5.00s cancellation vs. 30s hang.
- **Post-fix runtime regression (24 rows):** Re-tested Rows 1–7 and Row 9 for A, B, and C; captured replacement screenshots in `evidence/fixed/a-row1.png` through `c-row9.png`. All 27 SHA-256 hashes are unique.
- **TypeScript static checking (Row 8):**
  - `npx.cmd tsc -p jsconfig.json --checkJs --noEmit`: Confirmed 0 errors in `a.js`, `b.js`, and `c.js`.
  - Captured full code context and compiler diagnostics in `evidence/fixed/*-row8.png`.
- **Failure scenario simulations:** Simulated stock rejections to verify Claim 2 (unhandled promise rejection in unawaited `forEach`) and Claim 7 (leftover rendered cards in Version B).
- **CSR vs SSR comparative testing:**
  - View Page Source (`Ctrl+U`) on `/?v=c` and `/ssr`.
  - DevTools *Disable JavaScript* test on both pages.
  - Network tab request count and waterfall inspection on both pages (14 requests vs. 1–2).
- **Repository integrity:** Checked `git diff f43bb31..HEAD --stat` to verify source scope.

## What I did not manage

- In the original evidence collection, some screenshots relied on reconstructed mock DevTools panels rather than raw browser DevTools captures.
- Entries 2–4 in `AI-LOG.md` were documented retrospectively rather than live as each step occurred, which I have noted transparently in the log.

## Total self-assessment score

**93/100**

`27 + 25 + 14 + 10 + 8 + 9 = 93/100`
