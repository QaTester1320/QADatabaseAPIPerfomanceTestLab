================================================================================
 QA TESTING GUIDE — DATABASE, API & PERFORMANCE TESTING (BASIC TO ADVANCED)
 Companion guide for the "QA Store" sandbox web app
================================================================================

HOW TO USE THIS GUIDE
--------------------------------------------------------------------------------
This guide is written to be used side-by-side with the QA Store sandbox app
(the single HTML file you were given). The sandbox gives you three practice
tools, reachable from the "QA Tools" menu in the site header:

  1. MySQL Database console   -> #/database   (phpMyAdmin / Workbench style)
  2. Postman-style API Client  -> #/postman
  3. Performance Lab           -> #/performance

Everything runs inside your browser. Nothing is installed, nothing is sent
to a real server, and nothing you do can break anything permanently — the
Database page has a "Reset demo data" button that restores everything.

Demo logins:
  Admin account    : admin@qa.com    / Admin@123
  Customer account : john@example.com / Test@123
  Blocked account  : ravi@example.com / Test@123   (use this to test blocked-user errors)


================================================================================
 PART 1 — DATABASE TESTING
================================================================================

1.1 WHAT DATABASE TESTING IS
--------------------------------------------------------------------------------
Database testing checks that the data behind an application is correct,
consistent, and behaves the way the business rules say it should. As a
manual/QA tester you are usually checking three things:

  a) Data validity   — does the data stored match what should have been stored
                        after a UI action (e.g. after "Add to cart" and
                        "Place order", is there a row in orders/order_items)?
  b) Data integrity   — do relationships between tables stay consistent
                        (e.g. can you delete a category that still has
                        products pointing at it? Should you be able to?)
  c) Data quality     — are constraints enforced (required fields, allowed
                        values, ranges, uniqueness)?

1.2 THE SANDBOX SCHEMA
--------------------------------------------------------------------------------
Open #/database and look at the left sidebar — it lists every table and its
columns, exactly like phpMyAdmin/Workbench. The schema is:

  categories     (id, name, slug, description, image, status)
  products       (id, name, category_id -> categories.id, price, old_price,
                  stock, sku, description, image, tags, status, created_at)
  customers      (id, name, email, password, phone, role, status, created_at)
  reviews        (id, product_id -> products.id, customer_name, rating 1-5,
                  comment, status, created_at)
  cms_pages      (id, title, slug, content, status, created_at)
  orders         (id, customer_id -> customers.id, total, payment_method,
                  status, shipping_address, created_at)
  order_items    (id, order_id -> orders.id, product_id -> products.id,
                  quantity, price)
  wishlist       (id, customer_id -> customers.id, product_id -> products.id)
  compare_list   (id, customer_id -> customers.id, product_id -> products.id)

Foreign keys and their delete behaviour (this matters for integrity testing):
  products.category_id   -> categories.id   ON DELETE RESTRICT
  reviews.product_id     -> products.id     ON DELETE CASCADE
  orders.customer_id     -> customers.id    ON DELETE SET NULL
  order_items.order_id   -> orders.id       ON DELETE CASCADE
  order_items.product_id -> products.id     ON DELETE RESTRICT
  wishlist / compare_list -> CASCADE on both customer and product

Check constraints enforced by the sandbox:
  reviews.rating   must be between 1 and 5
  products.price   must be >= 0
  products.stock   must be >= 0
  order_items.quantity must be > 0

1.3 BASIC SQL — START HERE
--------------------------------------------------------------------------------
Type these into the SQL editor and click Execute (or press Ctrl+Enter).

  -- View everything in a table
  SELECT * FROM products;

  -- Pick specific columns
  SELECT id, name, price FROM products;

  -- Filter rows
  SELECT * FROM products WHERE price > 100;
  SELECT * FROM products WHERE status = 'active' AND stock < 10;
  SELECT * FROM customers WHERE role = 'admin';

  -- Sort and limit
  SELECT name, price FROM products ORDER BY price DESC LIMIT 5;

  -- Pattern match
  SELECT name FROM products WHERE name LIKE '%watch%';

  -- Ranges and lists
  SELECT * FROM products WHERE price BETWEEN 50 AND 150;
  SELECT * FROM orders WHERE status IN ('Pending','Processing');

  -- Uniqueness
  SELECT DISTINCT status FROM orders;

TEST IDEAS (basic):
  - Run SELECT * FROM customers; and confirm the row count matches what the
    Admin panel's Customers tab shows.
  - Filter products where stock = 0 and confirm they show "Out of stock" on
    the storefront.
  - Check that every product's category_id actually exists in categories
    (SELECT * FROM products WHERE category_id NOT IN (SELECT id FROM categories);
    should return 0 rows).

1.4 INTERMEDIATE SQL — JOINS & AGGREGATES
--------------------------------------------------------------------------------
  -- Inner join: only rows that match on both sides
  SELECT p.name, c.name AS category
  FROM products p
  JOIN categories c ON c.id = p.category_id;

  -- Left join: keep all rows from the left table even with no match
  SELECT cu.name, o.id AS order_id, o.total
  FROM customers cu
  LEFT JOIN orders o ON o.customer_id = cu.id;
  -- (customers with no orders show NULL order_id/total — great for testing
  --  "customers who never ordered")

  -- Aggregates
  SELECT COUNT(*) FROM products;
  SELECT category_id, COUNT(*) AS total, ROUND(AVG(price),2) AS avg_price
  FROM products
  GROUP BY category_id;

  -- Aggregates with a filter on the group (HAVING, not WHERE)
  SELECT category_id, COUNT(*) AS total
  FROM products
  GROUP BY category_id
  HAVING COUNT(*) > 2
  ORDER BY total DESC;

  -- Order total recalculated from line items (useful to catch total bugs)
  SELECT o.id, SUM(oi.quantity * oi.price) AS calculated_total, o.total AS stored_total
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.id
  GROUP BY o.id;

TEST IDEAS (intermediate):
  - Compare calculated_total vs stored_total from the query above for every
    order — flag any mismatch as a bug.
  - Find products that have never been ordered:
    SELECT p.* FROM products p
    LEFT JOIN order_items oi ON oi.product_id = p.id
    WHERE oi.id IS NULL;
  - Find the top 3 best-selling products by quantity.

1.5 DATA MODIFICATION — INSERT / UPDATE / DELETE
--------------------------------------------------------------------------------
  INSERT INTO categories (name, slug, status) VALUES ('Toys','toys','active');
  UPDATE products SET price = price * 1.10 WHERE category_id = 1;
  DELETE FROM reviews WHERE status = 'rejected';

After running one of these, immediately re-run a SELECT to confirm the
change actually happened — never assume "Query OK" means the data is
correct; verify it.

1.6 ADVANCED — CONSTRAINT & NEGATIVE TESTING
--------------------------------------------------------------------------------
This is where database testing gets interesting: proving the database
REJECTS bad data.

  -- Should fail: rating out of range (CHECK constraint)
  INSERT INTO reviews (product_id, customer_name, rating) VALUES (1,'Tester',9);

  -- Should fail: category_id that doesn't exist (foreign key)
  INSERT INTO products (name, category_id, price, stock) VALUES ('X', 999, 10, 5);

  -- Should fail: deleting a category that still has products (RESTRICT)
  DELETE FROM categories WHERE id = 1;

  -- Should succeed and cascade: deleting a product removes its reviews too
  -- (verify with a SELECT on reviews before and after)
  DELETE FROM products WHERE id = 12;

  -- Should fail: duplicate unique value
  INSERT INTO categories (name, slug) VALUES ('Electronics','electronics-2');
  -- (fails because "Electronics" name already exists — try a duplicate slug too)

  -- Should fail: required field missing / wrong type
  INSERT INTO products (name, price) VALUES ('No category', 10);
  UPDATE products SET status = 'discontinued' WHERE id = 1; -- invalid ENUM value

For every negative test, read the exact error the engine returns (it mimics
real MySQL error codes like 1451, 1452, 1062, 1048, 3819) and confirm the
error is meaningful — that is exactly what you would check against a real
MySQL server.

1.6b VERY ADVANCED SQL — SUBQUERIES, UNION, EXPLAIN, TRANSACTIONS
--------------------------------------------------------------------------------
The console also supports the features interviewers most often ask about:

  -- Subquery in WHERE (scalar subquery)
  SELECT name, price FROM products
  WHERE price > (SELECT AVG(price) FROM products)
  ORDER BY price DESC;

  -- Subquery with IN
  SELECT * FROM customers
  WHERE id IN (SELECT customer_id FROM orders);

  -- UNION (combines two result sets, removes duplicate rows)
  SELECT name AS label FROM products WHERE price > 500
  UNION
  SELECT title FROM cms_pages;

  -- UNION ALL (keeps duplicates, faster — use when you know there won't be
  -- overlapping rows, or you don't care)
  SELECT name FROM products
  UNION ALL
  SELECT title FROM cms_pages;

  -- EXPLAIN — shows a (simulated, for-learning) query plan: which table is
  -- scanned, whether an index-like lookup is used, and how many rows are
  -- estimated. Real MySQL EXPLAIN is one of the most useful tools for
  -- diagnosing a slow query — practise reading the "type" and "rows"
  -- columns here so the real thing feels familiar.
  EXPLAIN SELECT * FROM products WHERE id = 1;

  -- Transactions — group several statements so they all succeed or all
  -- get undone together. This is exactly how a real e-commerce "place
  -- order" operation is written (insert the order, insert the order
  -- items, reduce stock — either ALL of that happens, or NONE of it does).
  START TRANSACTION;
  UPDATE products SET stock = stock - 1 WHERE id = 1;
  SELECT id, stock FROM products WHERE id = 1;   -- stock is now reduced
  ROLLBACK;                                       -- undo everything above
  SELECT id, stock FROM products WHERE id = 1;   -- stock is back to normal

  -- Use COMMIT instead of ROLLBACK to keep the changes permanently.

TEST IDEAS (very advanced):
  - Write a subquery to find the single most expensive product per
    category (hint: compare each product's price to the MAX(price) of
    products in the same category_id).
  - Use EXPLAIN on a query with a JOIN and note how the "type" column
    differs between the driving table and the joined table.
  - Wrap a multi-step "checkout" simulation (reduce stock, insert an
    order) in a transaction, verify the change, then ROLLBACK and confirm
    everything reverted — this is the same pattern used to test that a
    real payment failure doesn't leave half-finished data behind.

1.7 A SIMPLE DATABASE TEST CHECKLIST
--------------------------------------------------------------------------------
  [ ] Every foreign key column only contains values that exist in the parent
      table.
  [ ] Required (NOT NULL) fields cannot be left empty.
  [ ] Unique fields (email, slug, SKU) reject duplicates.
  [ ] Enum/status fields reject invalid values.
  [ ] Numeric ranges (rating 1-5, price >= 0, stock >= 0) are enforced.
  [ ] Deleting a "parent" record behaves as documented (blocked, cascaded,
      or nulled — know which one is expected for each relationship).
  [ ] Calculated values (order totals, review averages) match what a manual
      SUM/AVG query produces.
  [ ] Row counts shown in the UI (Admin panel counts, product counts per
      category) match COUNT(*) queries.


================================================================================
 PART 2 — API TESTING
================================================================================

2.1 WHAT API TESTING IS
--------------------------------------------------------------------------------
API testing checks a system at the HTTP layer, below the UI: you send a
request (method + URL + headers + body) and verify the response (status
code, headers, and body) is correct — including for invalid input.

Open #/postman. The Base URL for every endpoint in this sandbox is:
  https://api.qasandbox.com/v1

The left panel has a ready-made collection of requests grouped into folders
(Auth, Products, Categories, Orders, Wishlist & Reviews, Negative & Debug).
Click any request to load it into a tab, then click Send.

2.2 HTTP BASICS — METHODS & STATUS CODES
--------------------------------------------------------------------------------
Methods used in this API:
  GET     — read data, never changes anything, safe to repeat
  POST    — create a new record
  PUT     — replace/update an existing record
  PATCH   — partially update an existing record
  DELETE  — remove a record

Status codes you will see from this sandbox API (the same families exist on
every real REST API):
  200 OK                  — request succeeded
  201 Created             — a new resource was created (check the response
                             body AND the Location header)
  204 No Content          — succeeded, nothing to return
  400 Bad Request         — malformed request (bad JSON, bad query params)
  401 Unauthorized        — missing/invalid/expired auth token
  403 Forbidden           — authenticated, but not allowed to do this
  404 Not Found           — resource or route does not exist
  405 Method Not Allowed  — right URL, wrong HTTP method
  409 Conflict            — request conflicts with current state (duplicate,
                             insufficient stock, dependent records exist)
  415 Unsupported Media Type — Content-Type header missing/wrong for a body
  422 Unprocessable Entity — validation failed (see the "errors" object)
  429 Too Many Requests   — rate limit exceeded
  500 Internal Server Error — unexpected server-side failure

2.3 YOUR FIRST REQUESTS (BASIC)
--------------------------------------------------------------------------------
1. Open the "Products" folder, click "List products", click Send.
   - Check: status is 200, response has "data" (an array), "total", "page".
2. Click "Get product by id" (GET /products/1), click Send.
   - Check: response "id" field equals 1.
3. Click "Get product (not found)" (GET /products/9999), click Send.
   - Check: status is 404 and the message explains what went wrong.
4. Open "Auth" -> "Login (customer)", click Send.
   - Check: status 200, response contains a "token" and a "user" object.
   - Notice: the app automatically stores this token as {{token}} for you —
     look at the "Environment" bar at the top, the token now shows as filled.
5. Open "Auth" -> "Get my profile" (GET /me), click Send.
   - This endpoint requires auth. Because you just logged in, it should
     succeed (200). Look at the Headers tab of the request — Authorization
     is auto-filled with "Bearer {{token}}".

2.4 REQUEST COMPONENTS TO PRACTISE WITH
--------------------------------------------------------------------------------
  Params tab   — query string parameters, e.g. ?page=1&limit=10. Try the
                 "Filter by category & price" request and change min_price
                 / max_price / sort values, then Send again.
  Headers tab  — key/value pairs sent with the request. Try removing the
                 Content-Type header from a POST request and re-sending —
                 you should get 415 Unsupported Media Type.
  Body tab     — raw JSON for POST/PUT/PATCH. Try the "Beautify" button,
                 and try breaking the JSON (remove a closing brace) to see
                 the live "Invalid JSON" warning before you even send it.
  Authorization tab — toggle "Bearer Token" on/off to see 401 vs 200.
  Tests tab    — simple JavaScript assertions that run after the response
                 comes back (see section 2.6).

2.5 VALIDATION / NEGATIVE TESTING (INTERMEDIATE)
--------------------------------------------------------------------------------
Good API testing spends MORE time on invalid input than valid input. Try:

  - POST /auth/register with an already-registered email -> expect 422 with
    an "errors.email" message.
  - POST /auth/register with a weak password (e.g. "123") -> expect 422.
  - POST /auth/login with a wrong password -> expect 401.
  - POST /auth/login as ravi@example.com / Test@123 (the blocked account)
    -> expect 403 "account has been blocked".
  - POST /products (admin-only) while logged in as a normal customer
    -> expect 403.
  - POST /products with a negative price or missing category_id -> expect
    422 and check every field-level error message makes sense.
  - GET /products?page=0 or ?limit=999 -> expect 400 (invalid pagination).
  - PUT /orders/1 with an invalid status value -> expect 422.
  - POST /orders for a product with more quantity than is in stock ->
    expect 409 Conflict.
  - Send a request to a route that doesn't exist (see "Unknown route (404)"
    in the Negative & Debug folder).
  - Send a DELETE to the API root (see "Wrong method (405)") and check the
    response's "Allow" header lists the correct methods.

2.6 WRITING TEST ASSERTIONS (INTERMEDIATE -> ADVANCED)
--------------------------------------------------------------------------------
Open the Tests tab on any request and write assertions like:

  pm.test('Status code is 200', () => pm.response.status === 200);
  pm.test('Response has a data array', () => Array.isArray(pm.response.json.data));
  pm.test('Response time is acceptable', () => pm.response.time < 1000);
  pm.test('Product price is a number', () => typeof pm.response.json.price === 'number');

Click Send, then open the "Test Results" tab in the response panel to see
PASS/FAIL for each assertion — this is exactly the workflow used in real
Postman collections and in CI pipelines.

2.7 ADVANCED — DEBUG ENDPOINTS FOR EDGE CASES
--------------------------------------------------------------------------------
The "Negative & Debug" folder has endpoints built specifically for testing
edge cases that are hard to trigger otherwise:

  GET /debug/status/:code   — forces the API to return any status code you
                              choose (e.g. /debug/status/500). Use this to
                              test how your own front-end handles a 500,
                              502, 503 etc. without needing a broken server.
  GET /debug/delay/:ms      — forces a slow response (0-10000 ms). Use this
                              to test loading spinners, timeouts, and to
                              feed the Performance Lab.
  GET /debug/flaky          — randomly fails ~30% of the time. Good for
                              testing retry logic.
  GET /debug/rate-limit     — returns 429 after 5 requests in 10 seconds.
                              Send it 6 times quickly and inspect the
                              Retry-After header on the 6th response.
  ANY /debug/echo           — echoes back your method, query params,
                              headers and body exactly as sent. Useful for
                              confirming exactly what a client is sending.

2.7b I DON'T KNOW HOW TO WRITE A TEST SCRIPT — START HERE
--------------------------------------------------------------------------------
If you've never written an automated test assertion before, open any
request's "Tests" tab and use the "Insert a ready-made test snippet…"
dropdown. Pick one (e.g. "Status code equals…") and it is typed into the
box for you automatically. You can insert several — each one becomes its
own pm.test(...) line. Click Send, then open the response's "Test Results"
tab to see PASS/FAIL. Once you're comfortable, try editing a snippet's
numbers/text yourself (e.g. change 200 to 201, or change the field name)
— that small step is exactly how real testers learn to write their own
assertions: start from a working example and adjust it.

2.7c TESTING YOUR OWN CUSTOM ENDPOINT (ANY PATH, ANY STATUS CODE)
--------------------------------------------------------------------------------
Sometimes you want to test how a front-end handles a very specific
response — a particular status code, a particular JSON shape, or an
endpoint at a path YOU choose. You have two ways to do this:

  a) QUICK — GET /debug/status/:code
     Call https://api.qasandbox.com/v1/debug/status/503 (or any code
     100-599) and you'll get exactly that status back immediately. Good
     for a one-off check with no setup.

  b) FULL CONTROL — build your own mock endpoint
     Log in as admin (admin@qa.com / Admin@123) and open
     Admin Panel -> API Mocks -> "+ Add new". Fill in:
       - HTTP method (GET/POST/PUT/PATCH/DELETE)
       - Path, e.g. /my/orders-service-down (any path you like — it does
         NOT need to start with /v1)
       - Status code — literally anything, e.g. 200, 404, 500, 503, 418
       - Whether it should require a Bearer token
       - A JSON response body
     Save it, then go to #/postman — your new endpoint automatically
     appears in a "My Custom Mocks" folder in the collection sidebar, or
     you can just type its full URL (https://api.qasandbox.com/my/…)
     into any request tab yourself and hit Send. This is the same idea as
     a "mock server" in real test automation (e.g. WireMock, json-server,
     Postman Mock Servers) — you decide exactly what the API returns so
     you can test how your application reacts to conditions that are hard
     to trigger from a real backend, like a 500 error or a 429 rate limit.

2.8 A SIMPLE API TEST CHECKLIST
--------------------------------------------------------------------------------
  [ ] Every endpoint returns the documented status code for valid input.
  [ ] Every required field is actually enforced (send a request without it).
  [ ] Every field's type/format is enforced (string where a number is
      expected, invalid email format, invalid enum value).
  [ ] Auth-protected endpoints reject missing/invalid/expired tokens (401)
      and reject the wrong role (403).
  [ ] Pagination parameters are validated and page 2, 3... return different
      data than page 1.
  [ ] Response bodies match documented shape — no unexpected missing/extra
      fields your consumers rely on.
  [ ] Idempotent methods (GET, PUT, DELETE) can be called repeatedly with
      the same result.
  [ ] Rate limiting, timeouts and 5xx errors are handled gracefully by
      whatever consumes this API.


================================================================================
 PART 3 — PERFORMANCE TESTING
================================================================================

3.1 WHAT PERFORMANCE TESTING IS
--------------------------------------------------------------------------------
Performance testing checks how a system behaves under load: is it fast
enough, does it stay correct under concurrent users, and where does it
break first? Common types:

  Load testing        — expected normal/peak traffic, checking response
                         times stay acceptable.
  Stress testing       — push traffic beyond expected peak to find the
                         breaking point.
  Spike testing        — sudden burst of traffic, then back to normal.
  Soak/endurance test   — moderate load sustained for a long time, looking
                         for memory leaks or slow degradation.

3.2 KEY METRICS
--------------------------------------------------------------------------------
  Response time (avg)  — the average time per request.
  Percentiles (p90/p95/p99) — e.g. p95 = 95% of requests were faster than
                         this value. Percentiles matter more than the
                         average because they reveal how bad the WORST
                         requests are, which is what real users notice.
  Throughput            — requests handled per second.
  Error rate            — percentage of requests that failed (non-2xx or
                         network errors).
  Concurrency / virtual users — how many simulated users are hitting the
                         system at the same time.

3.3 USING THE PERFORMANCE LAB (BASIC)
--------------------------------------------------------------------------------
Open #/performance.
  1. Under "Test plan", pick an endpoint (e.g. "GET /products (list)") and a
     number of virtual users (e.g. 5).
  2. Click "Run load test".
  3. Watch the KPIs update live: Total requests, Avg response, 95th
     percentile, Error rate.
  4. Watch the chart: the purple line is response time per request, red
     dots mark failed requests.
  5. When it finishes, read the PASS/FAIL verdict at the bottom-left, based
     on: avg < 400ms, p95 < 800ms, error rate < 5%.

3.4 COMPARING ENDPOINTS (INTERMEDIATE)
--------------------------------------------------------------------------------
  - Run 10 users against "GET /categories" (a small, simple response) and
    note the avg/p95.
  - Run 10 users against "GET /products (list)" (a bigger response) and
    compare. Bigger/heavier endpoints should generally be slower.
  - Run 10 users against "GET /debug/delay/300" — this simulates a
    slow downstream dependency (e.g. a slow database query) and shows how
    a single slow endpoint drags down your percentiles.

3.5 TESTING ERROR HANDLING UNDER LOAD (ADVANCED)
--------------------------------------------------------------------------------
  - Run a load test against "GET /debug/flaky" (fails ~30% of the time).
    Confirm the error rate KPI is roughly 30% and the verdict correctly
    fails the < 5% error-rate threshold.
  - Add multiple stages to the test plan (click "+ Add stage") with
    different endpoints and user counts to simulate a mixed, realistic
    traffic pattern (this mirrors how real tools like JMeter or k6 let you
    define multi-stage scenarios).
  - Increase virtual users on a single stage (e.g. from 5 to 50) and watch
    whether p95 grows disproportionately — in a real system this is the
    signal that you are approaching a bottleneck.

3.6 A SIMPLE PERFORMANCE TEST CHECKLIST
--------------------------------------------------------------------------------
  [ ] Baseline recorded for each key endpoint at low concurrency.
  [ ] Response time thresholds agreed and documented (e.g. p95 < 800ms).
  [ ] Load test run at expected normal traffic — thresholds met.
  [ ] Load test run at expected peak traffic — thresholds met or a known,
      accepted degradation.
  [ ] Error rate stays under the agreed limit throughout the test.
  [ ] Slow/flaky dependencies are identified and reported, not just the
      overall pass/fail.
  [ ] Results are compared over time (this run vs the last run) to catch
      performance regressions early.


================================================================================
 PART 4 — PUTTING IT ALL TOGETHER (END-TO-END PRACTICE)
================================================================================
IMPORTANT: the storefront, the Admin panel, the Database console, and the
API client are NOT separate demos with separate fake data — they all read
and write the exact same underlying data (kept in your browser's storage).
That means:
  - Add a product in Admin -> Products, and it instantly appears in the
    Shop page and is queryable in the Database console and via
    GET /v1/products in the API client.
  - Place an order on the storefront, and you can immediately find it with
    SELECT * FROM orders in the Database console, see it via GET /v1/orders
    in the API client, and see/update its status in Admin -> Orders.
  - Run an UPDATE or DELETE in the Database console, and the storefront
    and Admin panel reflect it the moment you reload that page.
  - A customer registered through the storefront's Sign Up page, through
    the Admin -> Customers "+ Add new" form, or through POST /v1/auth/register
    in the API client all create the exact same kind of row in the same
    customers table — try all three and confirm with a SELECT.

The real value of this sandbox is combining all three tools on ONE scenario,
just like you would on a real project:

  1. On the storefront, log in as john@example.com, add a product to the
     cart, and complete checkout.
  2. Open #/database and run:
       SELECT * FROM orders ORDER BY id DESC LIMIT 1;
       SELECT * FROM order_items WHERE order_id = <that id>;
     Confirm the row matches what you just did in the UI (correct product,
     quantity, total).
  3. Open #/postman, log in via "Login (customer)", then send "List my
     orders" (GET /orders) and confirm the same order appears in the API
     response.
  4. Open #/performance and load-test "GET /products (list)" to see how the
     product catalogue endpoint — the same one that powered the page you
     just shopped on — behaves under 20 concurrent users.

This "UI action -> database row -> API response -> performance under load"
loop is exactly the mindset a strong QA/SDET brings to a real application,
and you can now rehearse it as many times as you like, for free, with
nothing to install.

Good luck, and happy testing!
================================================================================
