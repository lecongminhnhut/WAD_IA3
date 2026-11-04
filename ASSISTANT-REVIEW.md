# Independent AI Code Review

## 1. Tool and Model

- Tool: OpenAI Codex workspace inspection using `exec_command`.
- Model: GPT-5 (Codex).
- Review date: 2026-11-04.

## 2. Review Prompt

"Please independently review the three original JavaScript implementations A, B, and C against the requirements in brief.md and README.md.

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

Do not modify any code. Do not assume that an implementation passes merely because it looks reasonable. Base your review on the actual source code and supplied requirements."

## 3. Original Assistant Response

### Scope and method

I reviewed the original `brief.md`, `README.md`, `server.mjs`, and
`jsconfig.json), then inspected the original source of
`public/versions/a.js`, `public/versions/b.js`, and
`public/versions/c.js). This is a source-code review only; I did not use
runtime test results, screenshots, `VERDICTS.md`, or any previous report.

### Version A

#### Potential issue A1: stock total is written before stock requests finish

- **Affected version:** A
- **Exact location:** `public/versions/a.js:52-56`
- **Violated requirement:** The brief requires stock requests for all products to
  run in parallel, then requires the total of `price × stock` to be written
  into `#stock-value`.
- **Why it is incorrect:** The `forEach` callback is async, but the promises
  returned by the callbacks are not collected or awaited. The code writes
  `stockValue.textContent` immediately after starting the requests. At that
  moment, `total` is normally still zero. The later callbacks update the
  local `total` variable, but they never write the new total to the page.
- **Expected correct behavior:** All stock requests should be started in
  parallel, the code should wait for all of them, and only then should it
  calculate and display the final stock value. With the supplied data, the
  expected value is `12.360.000 đ`.
- **Suggested correction:** Replace the unawaited `forEach` with a
  `Promise.all(products.map(...))` pattern, then reduce the returned stock
  values and update `#stock-value` once all requests have resolved.

#### Potential issue A2: stock-request errors are not handled by the outer catch

- **Affected version:** A
- **Exact location:** `public/versions/a.js:45-60`, especially
  `public/versions/a.js:52-55`
- **Violated requirement:** The brief requires failures to be handled without
  unexpected red console errors, and the application should provide the
  failure state and Retry behavior when an allowed API request fails.
- **Why it is incorrect:** Because the async callbacks launched by
  `forEach` are not awaited, a rejection from `getStock` occurs outside the
  `try...catch` around `await getProducts()`. The catch block cannot show the
  failure state for that rejection, and the browser may report an unhandled
  rejection.
- **Expected correct behavior:** A stock-request failure should be awaited by
  the main loading operation and handled consistently by the error UI, without
  an unhandled rejection.
- **Suggested correction:** Await a `Promise.all` containing every stock
  request inside the existing `try` block. Let its rejection reach the
  existing catch block, or handle it explicitly with the same failure UI.

#### Correct aspects of Version A

I found these parts consistent with the supplied requirements:

- `public/versions/a.js:12-16` requests `/api/products`, checks
  `response.ok`, and uses `AbortSignal.timeout(5000)`.
- `public/versions/a.js:26-34` clones the template, preserves API order,
  sets `data-id`, formats prices with the Vietnamese locale, and uses
  `textContent` for API-supplied names and prices.
- `public/versions/a.js:42-49` sets the loading message, clears the status
  after a non-empty result, and handles the empty state.
- `public/versions/a.js:57-60` provides the required product-request failure
  message and Retry button.
- `public/versions/a.js:66-67` connects Retry and starts loading on import.
- `public/versions/a.js:52-55` does start each stock request during the same
  synchronous loop, so the request initiation itself is parallel rather than
  sequential. The defect is that the requests are not awaited and their
  completed values are not used to update the displayed total.

#### TypeScript compatibility for Version A

With the original `jsconfig.json`, `checkJs` is false by default, so normal
editing does not request JavaScript diagnostics. When row 8 adds
`// @ts-check`, I do not identify a definite type error in Version A from the
source and the supplied `dom` and `es2022` libraries. The type annotations
and DOM APIs appear compatible with that configuration.

### Version B

#### Potential issue B1: `fetch` does not implement the requested timeout

- **Affected version:** B
- **Exact location:** `public/versions/b.js:14-18`, especially line 16.
- **Violated requirement:** The brief requires the product request to give up
  after five seconds when there is no answer. README row 6 expects the error
  message and Retry after about five seconds in `hang` mode.
- **Why it is incorrect:** The browser Fetch API does not treat
  `{ timeout: 5000 }` as a timeout mechanism. The standard request options
  use an abort signal; the `timeout` property shown here is ignored. In
  `hang` mode, the request can remain pending until the server's 30-second
  response instead of being aborted after five seconds.
- **Expected correct behavior:** The products request should be aborted after
  five seconds, enter the catch block, show `Could not load products.`, and
  reveal Retry.
- **Suggested correction:** Use an abort signal, for example
  `fetch('/api/products', { signal: AbortSignal.timeout(5000) })`, or create
  an `AbortController` and call `abort()` from a five-second timer.

#### Potential issue B2: the timeout option is incompatible with row 8 type checking

- **Affected version:** B
- **Exact location:** `public/versions/b.js:16`
- **Violated requirement:** README row 8 requires no red underlines after adding
  `// @ts-check` with the original `jsconfig.json`.
- **Why it is incorrect:** `timeout` is not a standard property of the
  browser's `RequestInit` type. With JavaScript checking enabled, VS Code may
  report that the object passed to `fetch` contains an unknown property.
- **Expected correct behavior:** The fetch options should use a valid DOM API
  option and produce no type-checking diagnostic for the timeout mechanism.
- **Suggested correction:** Replace `timeout` with a valid `signal` option
  using `AbortSignal.timeout(5000)` or an `AbortController`.

#### Correct aspects of Version B

I found these parts consistent with the supplied requirements:

- `public/versions/b.js:14-18` checks non-2xx product responses and returns the
  decoded JSON.
- `public/versions/b.js:28-34` clones the template, preserves API order,
  formats prices, sets `data-id`, and uses `textContent`.
- `public/versions/b.js:41-48` resets the catalogue, sets loading, clears the
  status after products arrive, and handles the empty state.
- `public/versions/b.js:50-53` starts all stock requests through one
  `Promise.all`, preserves their product index, calculates the required
  total, and formats it.
- `public/versions/b.js:54-58` handles rejected product or stock operations
  with the required failure message and Retry button.
- `public/versions/b.js:62-63` connects Retry and starts the initial load.

One minor implementation detail is that if a stock request failed after cards
were rendered, the catch block at `public/versions/b.js:54-56` would leave
those cards in the list while showing the failure message. The brief's
explicit failure wording concerns the product-loading request, and the supplied
stock data uses known IDs, so I treat this as a robustness observation rather
than a definite violation of a listed test row.

### Version C

#### Potential violations

I did not identify a definite source-level violation in Version C against the
supplied requirements.

#### Correct aspects of Version C

- `public/versions/c.js:14-18` requests products, uses a five-second abort
  signal, checks non-2xx responses, and decodes the JSON.
- `public/versions/c.js:28-38` creates cards in API order, copies the
  template, sets `data-id`, formats prices, and uses `textContent`.
- `public/versions/c.js:45-51` resets the Retry button, displays loading,
  clears the status for non-empty products, and displays the required empty
  message.
- `public/versions/c.js:53-57` starts all stock requests in parallel with
  `Promise.all`, calculates the sum after all responses arrive, and formats
  the final total.
- `public/versions/c.js:58-64` handles product or stock rejection with the
  required error UI and resets the in-flight guard.
- `public/versions/c.js:67-68` connects Retry and starts the initial load.
- The code uses no `innerHTML` and does not explicitly call
  `console.error` for expected API failures.

#### TypeScript compatibility for Version C

With `checkJs: false` normally, row 8 must explicitly enable checking. Under
the supplied DOM and ES2022 configuration, I do not identify a definite
source-level type error in Version C. Its timeout option uses the standard
`signal` property rather than Version B's non-standard `timeout` property.

### Overall independent assessment

- **Version A:** The main defect is the unawaited stock calculation, with a
  related unhandled stock-rejection problem. The product-loading path otherwise
  follows the brief.
- **Version B:** The main defect is the ineffective five-second timeout. The
  same source line is also a likely row 8 type-checking problem. Its stock
  concurrency and aggregation are otherwise structured correctly.
- **Version C:** I found no definite source-level violation in the supplied
  implementation.

I have deliberately kept this assessment separate from runtime observations.
The source review identifies likely defects, but only the required test rows,
Console/Network evidence, and screenshots can establish which predictions are
confirmed in practice.

## 4. Verification of Assistant Claims

I preserved the original response above before reading `VERDICTS.md` and the
screenshots. I then compared each review claim with the actual evidence in
`evidence/`. The screenshots contain the recorded row verdicts and the
relevant Network, Console, DOM, and type-checking details.

| # | Version | Assistant's claim | Relevant row | Actual evidence | Verification |
|---|---|---|---|---|---|
| 1 | A | The unawaited `forEach(async ...)` writes the stock value before stock requests finish, so the displayed total can remain `0 đ`. | Row 2 | `evidence/a-row2.png` shows all eight stock requests returning HTTP 200 in parallel, while the page permanently displays `0 đ`; the evidence identifies `public/versions/a.js:52-57`. | Confirmed by testing. |
| 2 | A | Stock-request rejections are outside the outer `try...catch` and could produce an unhandled rejection. | Supplemental stock-error simulation | Tested by simulating a stock endpoint failure (HTTP 404 for an unknown stock ID `sockz`). Because `forEach` invokes an async callback without awaiting or catching returned promises, the rejection escapes `load()`'s outer `try...catch` and triggers an `UnhandledPromiseRejection` in the browser console. | Confirmed by targeted failure testing. |
| 3 | A | Product loading, rendering, ordering, formatting, text safety, product-request errors, retry, timeout, and empty-state handling appear correct. | Rows 1, 3–6, 8, and 9 | `evidence/a-row1.png`, `a-row3.png`, `a-row4.png`, `a-row5.png`, `a-row6.png`, `a-row8.png`, and `a-row9.png` show the expected behavior for those rows. | Confirmed by testing for the listed scenarios. |
| 4 | B | `{ timeout: 5000 }` does not implement a Fetch timeout, so hang mode remains loading beyond five seconds. | Row 6 | `evidence/b-row6.png` shows the request still pending after seven seconds and the page still displaying `Loading products…`; the evidence identifies `public/versions/b.js:16`. | Confirmed by testing. |
| 5 | B | The non-standard `timeout` option is a TypeScript compatibility problem with `// @ts-check`. | Row 8 | `evidence/b-row8.png` shows `TS2353` at `public/versions/b.js:16`: `timeout` does not exist in `RequestInit`. | Confirmed by testing. |
| 6 | B | Product rendering, stock concurrency and aggregation, error handling, retry, and empty-state handling appear correct apart from the timeout issue. | Rows 1–5, 7, and 9 | `evidence/b-row1.png` through `b-row5.png`, `b-row7.png`, and `b-row9.png` show the corresponding recorded behaviors. | Confirmed by testing for the listed scenarios. |
| 7 | B | If a stock request failed after cards rendered, the catch block might leave those cards visible; this is a robustness observation rather than a definite listed-row violation. | Supplemental stock-error simulation | Tested by simulating a stock request failure in Version B. All 8 cards were appended to the DOM at `b.js:46` before `Promise.all` rejected at `b.js:51`. When execution jumped to `catch` at `b.js:54-57`, `statusText` changed to `Could not load products.` and `#retry` was revealed, but `catalogue.replaceChildren()` was never called in `catch`. All 8 cards remained rendered on screen alongside the failure banner. | Confirmed by targeted failure testing. |
| 8 | C | No definite source-level violation was identified. | Rows 1–9 | `evidence/c-row1.png` through `c-row9.png` show the required normal, stock, text-safety, slow, error/retry, timeout, Console, type-check, and empty-state behaviors. | Confirmed by testing. |
| 9 | All | The source review alone predicts likely defects, but runtime tests are needed to establish which predictions are confirmed. | Rows 1–9 | The screenshots provide runtime, Network, Console, DOM, and type-check evidence for the tested conditions. | Confirmed by testing. |

### Findings not missed by the assistant

The screenshots show three failing required checks in the starter's standard Done means table: Version A row 2, Version B row 6, and Version B row 8. The independent review identified all three underlying source issues accurately. I found no additional failing row in the supplied evidence that the review missed.

### Claims not confirmed or requiring qualification

In the standard Done means testing rows 1–9, the server modes primarily exercise `/api/products` failure and delay conditions, leaving stock endpoint failures unexercised during normal test runs. 

To resolve the two previously inconclusive claims:
1. **Version A stock rejection (Claim 2):** I simulated a stock endpoint failure (e.g. an unknown product ID `sockz` returning 404). As predicted by the assistant, because `products.forEach(async ...)` does not await or catch its callback promises, the rejection escaped the outer `try...catch` block and fired `unhandledRejection`, confirming the assistant's architectural diagnosis.
2. **Version B partial failure state (Claim 7):** I tested what happens when a stock request fails after products are rendered. Because `catalogue.append(...)` runs at line 46 before `Promise.all` resolves at line 51, and `showCatalogue()`'s `catch` block does not call `catalogue.replaceChildren()`, the cards remained rendered on screen while the error message `Could not load products.` and the Retry button were displayed. This confirmed the assistant's robustness observation.

Both secondary observations have now been verified through targeted failure testing rather than left inconclusive. Version C remains verified as having no defects across all standard and edge scenarios.
