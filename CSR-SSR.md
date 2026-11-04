# CSR vs SSR — The Same Catalogue, Built in Two Places

**Course:** CSC13008 – Web Programming  
**Activity:** IA#3 – Three Answers to One Brief  
**Student:** Lê Công Minh Nhựt  
**Date:** 2026-11-04  

---

## Background

During this activity, I worked with two rendering implementations of the same shop catalogue provided by `server.mjs`:
- **Client-Side Rendering (CSR)** accessed via `http://localhost:5050/?v=a` (as well as `?v=b` and `?v=c`).
- **Server-Side Rendering (SSR)** accessed via `http://localhost:5050/ssr`.

Below are my findings and observations answering the five questions from the assignment.

---

### 1. View Page Source: Product Names and Stock Value

> *On `/?v=` any version and on `/ssr`, right-click → **View page source** (`Ctrl+U`). Where are the product names in each? Where is the stock value?*

When I opened `http://localhost:5050/?v=a` (and the same applies to `v=b` and `v=c`) and selected **View page source** (`Ctrl+U`), I observed that:
- **Product names:** They do not appear anywhere in the initial HTML source. The catalogue list delivered from the server is completely empty:
  ```html
  <ul class="catalogue" id="catalogue"></ul>
  ```
  The `<template id="card-template">` only contains empty placeholder elements (`<h2 class="name"></h2>`, `<p class="price"></p>`). In CSR, the product names exist inside JSON data returned later by `GET /api/products`, and the browser's JavaScript dynamically constructs the cards and inserts the names into the DOM.
- **Stock value:** The page source contains only an initial placeholder dash:
  ```html
  <p class="stock-total">Stock value: <output id="stock-value">—</output></p>
  ```
  The actual total is calculated later by JavaScript in the browser after dispatching eight asynchronous requests to `/api/stock/:id`.

When I opened `http://localhost:5050/ssr` and viewed its page source, I observed the opposite:
- **Product names:** Every single product name is already embedded directly in the static HTML markup inside `<li class="card">` elements. For example:
  ```html
  <li class="card" data-id="tshirt">
    <h2 class="name">T-shirt</h2>
    <p class="price">180.000 đ</p>
  </li>
  ...
  <li class="card" data-id="scarf">
    <h2 class="name">Scarf &lt;i&gt;limited&lt;/i&gt;</h2>
    <p class="price">160.000 đ</p>
  </li>
  ```
- **Stock value:** The final stock total is already calculated and rendered directly in the HTML document:
  ```html
  <p class="stock-total">Stock value: <output id="stock-value">12.360.000 đ</output></p>
  ```
  The server ran `PRODUCTS.reduce(...)` during request handling, so the client received the complete text without needing any calculation.

---

### 2. The Scarf Name, HTML Escaping, and `textContent`

> *In `/ssr`'s source, find the scarf. Why does it read `&lt;i&gt;` there, and what does that have in common with `textContent` in the browser?*

In the source of `/ssr`, the scarf card's heading reads:
```html
<h2 class="name">Scarf &lt;i&gt;limited&lt;/i&gt;</h2>
```

**Why it reads `&lt;i&gt;`:**  
Looking at `server.mjs` (lines 29–37), the server passes product names through an `escapeHtml()` function:
```javascript
function escapeHtml(s) {
  return String(s)
    .replaceAll('&', '&amp;')
    .replaceAll('<', '&lt;')
    .replaceAll('>', '&gt;')
    .replaceAll('"', '&quot;')
    .replaceAll("'", '&#39;');
}
```
This replaces the literal `<` and `>` characters with character entities `&lt;` and `&gt;`. When the browser's HTML parser receives this document, it interprets `&lt;` and `&gt;` as literal angle brackets that belong to text content, rather than HTML tag delimiters. Consequently, no `<i>` element is created in the DOM, and "limited" is displayed with visible angle brackets instead of in italics. This prevents Cross-Site Scripting (XSS) and layout corruption from untrusted data.

**What it has in common with `textContent`:**  
Both approaches solve the same problem by ensuring untrusted strings are treated strictly as **text character data**, never as executable HTML markup:
- In CSR versions, assigning `card.querySelector('.name').textContent = p.name;` creates a plain Text node in the DOM tree, bypassing the browser's HTML parser entirely (unlike `innerHTML`).
- In SSR, calling `escapeHtml(p.name)` encodes the control characters before the HTML parser sees them, ensuring the parser creates a Text node instead of element nodes.

Both mechanisms guarantee that HTML tags in data strings remain visible text and cannot execute code.

---

### 3. Disabling JavaScript in DevTools

> *Disable JavaScript (DevTools settings, above) and reload both pages. What is left of each?*

When I disabled JavaScript in Chrome DevTools (*Settings → Debugger → Disable JavaScript*) and reloaded both pages, I observed:

- **On the CSR page (`/?v=a`, `b`, or `c`):**  
  Almost nothing of the application remained functional:
  - The catalogue container `<ul id="catalogue"></ul>` was completely empty. Zero product cards were displayed.
  - The stock value remained stuck at the initial placeholder: `Stock value: —`.
  - The status message remained blank; no loading indicator or error message appeared.
  - The buttons in the test conditions panel did nothing because their event listeners were not attached.
  - Only the static HTML layout was visible: the top toolbar ("Shop", version links), the heading, the panel box, and the footer.

- **On the SSR page (`/ssr`):**  
  The page remained **100% complete, fully legible, and identical in appearance**:
  - All eight product cards rendered with their names and prices.
  - The stock total `12.360.000 đ` was completely visible and accurate.
  - The stylesheet (`style.css`) loaded normally, keeping the styling intact.
  - Because all HTML was generated by Node.js on the server before transmission, disabling JavaScript in the client browser had zero impact on viewing the catalogue.

---

### 4. Network Request Count and Performance for Crawlers / Slow Devices

> *In the Network tab, count the requests each page makes before the stock value appears. Which page would a search engine, or a user on a slow phone, see first?*

When I monitored the Network tab with cache disabled, I counted the requests:

- **On `/ssr` (Server-Side Rendering):**  
  It makes **1 document request** (`GET /ssr`), followed by `GET /style.css` for styling. The entire catalogue content and the stock total (`12.360.000 đ`) arrive in the very first HTTP response payload.
- **On `/?v=<v>` (Client-Side Rendering):**  
  It makes **14 separate network requests** before the stock value can appear:
  1. `GET /?v=<v>` (initial HTML shell)
  2. `GET /style.css` (stylesheet)
  3. `GET /panel.js` (harness script)
  4. `GET /__mode` (current server test mode)
  5. `GET /versions/<v>.js` (the catalogue loader script)
  6. `GET /api/products` (catalogue JSON)
  7–14. Eight distinct requests to `GET /api/stock/:id` (`tshirt`, `hoodie`, `somi`, `cap`, `tote`, `socks`, `jacket`, `scarf`)

**Which page is seen first:**  
`/ssr` would be seen much faster by both a search engine and a user on a slow phone:
- **Search engines (crawlers):** Bots such as Googlebot, Bingbot, or social media scrapers index the initial HTML document. On `/ssr`, they immediately see all product titles, prices, and stock values in the initial response. On CSR, crawlers that do not execute JavaScript or have strict crawl/render timeout budgets only see an empty list and no stock total.
- **Users on slow phones (high-latency networks):** On high-latency mobile connections (e.g. 200–400 ms round trip time), CSR forces the browser through a 4-tier network waterfall (HTML → JS scripts → products API → stock APIs). Each sequential round trip adds noticeable delay, leaving the user waiting on a blank or loading screen. On `/ssr`, the server delivers the complete catalogue in a single round trip, resulting in a much faster First Contentful Paint (FCP) and saving phone battery and memory.

---

### 5. When to Choose Server-Side vs Client-Side Rendering

> *One sentence: when would you want the server to render, and when the browser?*

I would choose server-side rendering for public, content-heavy, or SEO-critical pages where fast initial page load, search engine indexing, and reliability on low-end devices or slow networks are essential (such as product catalogues, blogs, and marketing pages); and I would choose client-side rendering for interactive, stateful web applications where users perform frequent in-page interactions, live filtering, or complex workflows that benefit from smooth client updates without full-page reloads (such as administrative dashboards, settings panels, and SaaS tools).
