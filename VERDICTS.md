# Initial Version Predictions

I made these predictions by reading the original source code, `brief.md`, and
`README.md` before running the test table. These are not test observations. I
have not marked any test row as PASS or FAIL, and I will replace these
predictions with evidence after testing.

## Version A

### My prediction

I predict that Version A will load and display the product cards correctly. It
uses a five-second `AbortSignal.timeout`, checks the HTTP response, creates the
cards in API order, formats prices in Vietnamese currency, and inserts product
names with `textContent`. I also predict that its error state and Retry button
will work for product-request errors.

### Expected behavior

I expect the page to show `Loading products…` while `/api/products` is
pending. After the products arrive, I expect eight cards in the API order, the
status line to clear, and the scarf name to remain literal text rather than
becoming italic markup. I expect the error and hang conditions to show
`Could not load products.` with Retry after the request fails or reaches five
seconds.

### Potential problem and source lines

The stock requests are started inside `products.forEach(async (p) => ...)` at
`public/versions/a.js:52-55`, but the promises are not collected or awaited.
`public/versions/a.js:56` formats and writes `total` immediately. My prediction
is that the stock value can be written as `0 đ` before any stock response has
been added to `total`, and it will not be written again after the responses
finish. An error from one of these unawaited `getStock` calls also cannot be
handled by the surrounding `try...catch` at `public/versions/a.js:45-60`.

### Relevant test rows

I expect this issue to be exposed by row 2, which checks the calculated stock
value after waiting. Row 7 may also expose the unhandled-rejection aspect if a
stock request produces an error, although the listed standard server modes
primarily exercise the product request. Rows 1, 3, 4, 5, and 6 are relevant to
checking that the other parts of the implementation remain correct.

### Provisional verdict

My provisional verdict is **fix**.

### Reasoning

The product loading and rendering structure is close to the brief, but stock
calculation is a required part of the brief. Because Version A does not wait
for all stock requests before writing the total, I would not keep the original
implementation without a targeted fix.

## Version B

### My prediction

I predict that Version B will render cards, escape product names safely by
using `textContent`, request stock for all products in parallel with
`Promise.all`, and calculate the stock total after all stock responses arrive.
I also predict that its normal, slow, error, retry, and empty flows are
otherwise structured according to the brief.

### Expected behavior

I expect Version B to show the loading message, render the eight products in
order, preserve the literal scarf text, clear the status after a successful
load, and show the expected stock value after the parallel stock requests
finish. I expect the error state to clear the catalogue, show the failure
message, and reveal Retry.

### Potential problem and source lines

At `public/versions/b.js:16`, Version B passes `{ timeout: 5000 }` to
`fetch`. The browser Fetch API does not use this option as a request timeout.
Therefore, my prediction is that the products request will not be aborted after
five seconds. In the server's `hang` mode, the page may continue showing
`Loading products…` until the server eventually responds instead of entering
the required failure state after about five seconds.

The same line may also be reported by VS Code when `// @ts-check` is enabled,
because `timeout` is not a standard `RequestInit` property.

### Relevant test rows

I expect row 6 to be the main test for this problem. Row 8 may also identify
the invalid `timeout` option through the JavaScript type checker. Rows 1, 2, 3,
4, 5, and the additional `empty` condition are relevant to verifying the
working rendering, stock, escaping, retry, and empty-list behavior.

### Provisional verdict

My provisional verdict is **fix**.

### Reasoning

Version B handles the stock calculation more reliably than Version A, but the
five-second timeout is an explicit requirement. Since the timeout option in the
source does not implement that requirement, I would fix it before keeping the
version.

## Version C

### My prediction

I predict that Version C will satisfy the brief. It uses
`AbortSignal.timeout(5000)`, checks non-2xx responses, renders with a template
and `textContent`, preserves API order, and waits for all stock requests with
`Promise.all` before calculating the total.

### Expected behavior

I expect Version C to show the loading message, render eight correctly
formatted cards, preserve `Scarf <i>limited</i>` as visible text, calculate
`12.360.000 đ`, and show `0 đ` with `No products yet.` for an empty response. I
expect error and hang modes to show the required failure message and Retry
button, and I expect Retry to start a fresh request.

### Potential problems and source lines

From the source reading, I do not identify a specific implementation problem
in Version C. The relevant timeout logic is at `public/versions/c.js:16`, the
response-status check is at `public/versions/c.js:17`, safe text insertion is at
`public/versions/c.js:33-34`, and the parallel stock calculation is at
`public/versions/c.js:53-57`.

I still need to check row 8 with `// @ts-check`, because that is a required
editor/type-checking condition rather than something I can claim from a
runtime source reading alone.

### Relevant test rows

Rows 1 through 8 and the additional `empty` condition are relevant to
confirming this prediction. The CSR versus SSR questions in `README.md` are
also part of the assignment comparison, although they are not specific bugs in
Version C.

### Provisional verdict

My provisional verdict is **keep**.

### Reasoning

Version C appears to implement each requirement in the brief, including the
parts that Version A and Version B appear to mishandle. I will keep this as a
provisional verdict only until I have collected the required test evidence.

## Summary before testing

| Version | Provisional verdict | Main source-level concern |
|---|---|---|
| A | fix | Stock requests are not awaited at `a.js:52-56`. |
| B | fix | `fetch` timeout option is ineffective at `b.js:16`. |
| C | keep | No specific source-level problem identified before testing. |

These are predictions based on source inspection only. I have not run the
server, browser, Network checks, Console checks, or VS Code type check yet.

---

## Post-fix regression testing

I ran the post-fix browser regression against `http://localhost:5050` after
the Version A and Version B fix commits. I executed rows 1-9 for Versions A,
B, and C. The original evidence remains unchanged; the new evidence is in
`evidence/fixed/`.

The runtime tests below are based on the observed application behavior. Row 8
is recorded separately because the requested command was attempted, but the
local PowerShell execution policy prevented `npx.ps1` from running. I do not
claim a compiler PASS from that blocked command.

### Version A

| Row | Result | Actual observation | Evidence |
|---|---|---|---|
| 1 | PASS | The page loaded, eight product cards rendered in API order, and the normal stock total was `12.360.000 đ`. | `evidence/fixed/a-row1.png` |
| 2 | PASS | All eight stock requests were dispatched concurrently, the program waited for them, and the displayed total was `12.360.000 đ`. | `evidence/fixed/a-row2.png` |
| 3 | PASS | Product text including `Scarf <i>limited</i>` was displayed as text, not interpreted as HTML. | `evidence/fixed/a-row3.png` |
| 4 | PASS | The loading-delay case rendered the products and cleared the loading/error state after the delayed response. | `evidence/fixed/a-row4.png` |
| 5 | PASS | The initial error showed `Could not load products.` and Retry; retrying loaded the catalogue and restored the normal total. | `evidence/fixed/a-row5.png` |
| 6 | PASS | The hanging request reached the failure state and did not leave the page permanently in the loading state. | `evidence/fixed/a-row6.png` |
| 7 | PASS | The runtime failure case did not produce an uncaught browser exception in the observed run. | `evidence/fixed/a-row7.png` |
| 8 | INCOMPLETE | I attempted `npx tsc -p jsconfig.json --checkJs --noEmit`, but PowerShell blocked `npx.ps1`; therefore I do not claim a completed project compiler result. | `evidence/fixed/a-row8.png` |
| 9 | PASS | The empty response showed `No products yet.` and `0 đ` without product cards. | `evidence/fixed/a-row9.png` |

### Version B

| Row | Result | Actual observation | Evidence |
|---|---|---|---|
| 1 | PASS | The page loaded, eight product cards rendered in API order, and the normal stock total was `12.360.000 đ`. | `evidence/fixed/b-row1.png` |
| 2 | PASS | Stock requests remained concurrent and the completed stock total was `12.360.000 đ`. | `evidence/fixed/b-row2.png` |
| 3 | PASS | Product text including `Scarf <i>limited</i>` was displayed safely as text. | `evidence/fixed/b-row3.png` |
| 4 | PASS | The loading-delay case rendered the products and cleared the loading/error state after the delayed response. | `evidence/fixed/b-row4.png` |
| 5 | PASS | The initial error showed the required message and Retry; retrying restored the catalogue and normal total. | `evidence/fixed/b-row5.png` |
| 6 | PASS | The hanging request was aborted by the valid timeout mechanism and the UI left the loading state with the failure message and Retry. | `evidence/fixed/b-row6.png` |
| 7 | PASS | The runtime failure case did not produce an uncaught browser exception in the observed run. | `evidence/fixed/b-row7.png` |
| 8 | INCOMPLETE | I attempted the required TypeScript command, but PowerShell blocked `npx.ps1`; no compiler PASS is claimed from that command. | `evidence/fixed/b-row8.png` |
| 9 | PASS | The empty response showed `No products yet.` and `0 đ` without product cards. | `evidence/fixed/b-row9.png` |

### Version C

| Row | Result | Actual observation | Evidence |
|---|---|---|---|
| 1 | PASS | The page loaded, eight product cards rendered in API order, and the normal stock total was `12.360.000 đ`. | `evidence/fixed/c-row1.png` |
| 2 | PASS | Stock requests were concurrent, all completed, and the displayed total was `12.360.000 đ`. | `evidence/fixed/c-row2.png` |
| 3 | PASS | Product text including `Scarf <i>limited</i>` was displayed safely as text. | `evidence/fixed/c-row3.png` |
| 4 | PASS | The loading-delay case rendered the products and cleared the loading/error state after the delayed response. | `evidence/fixed/c-row4.png` |
| 5 | PASS | The initial error showed the required message and Retry; retrying restored the catalogue and normal total. | `evidence/fixed/c-row5.png` |
| 6 | PASS | The hanging request reached the required failure state instead of remaining stuck loading. | `evidence/fixed/c-row6.png` |
| 7 | PASS | The runtime failure case did not produce an uncaught browser exception in the observed run. | `evidence/fixed/c-row7.png` |
| 8 | INCOMPLETE | I attempted the required TypeScript command, but PowerShell blocked `npx.ps1`; no compiler PASS is claimed from that command. | `evidence/fixed/c-row8.png` |
| 9 | PASS | The empty response showed `No products yet.` and `0 đ` without product cards. | `evidence/fixed/c-row9.png` |

### TypeScript check result

I ran the exact command `npx tsc -p jsconfig.json --checkJs --noEmit`. It did
not reach TypeScript because this environment rejected `npx.ps1` under its
PowerShell execution policy. I also tried `npx.cmd`; it could not obtain the
uncached `tsc` package in the restricted environment. Consequently, the three
Row 8 entries remain **INCOMPLETE**, not PASS.

### Post-fix summary

| Version | Runtime result | Final status for this regression record |
|---|---|---|
| A | 8 PASS, 1 INCOMPLETE | KEEP for the confirmed runtime behavior; Row 8 still requires a successful TypeScript check. |
| B | 8 PASS, 1 INCOMPLETE | KEEP for the confirmed runtime behavior and timeout fix; Row 8 still requires a successful TypeScript check. |
| C | 8 PASS, 1 INCOMPLETE | KEEP for the confirmed runtime behavior; Row 8 still requires a successful TypeScript check. |

I did not replace the original failures in this document. The original
Version A Row 2 failure and Version B Rows 6 and 8 failures remain historical
results; this section records only the post-fix regression evidence.
