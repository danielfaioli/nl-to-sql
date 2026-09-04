# Wanderbricks NL-to-SQL Agent — MVP Tickets

Nine tickets, meant to be worked **in order**, one at a time. Each is scoped
so a coding agent can complete it, you can review/run the output, and only
then move to the next ticket. Don't hand all nine over at once — later
tickets depend on artifacts (files, working scripts) produced by earlier
ones.

Global context to paste into every ticket if your agent doesn't retain
conversation state across tickets:

> Project: a chatbot that answers questions over the Databricks "wanderbricks"
> sample dataset (9 Delta tables in Unity Catalog: users, clickstream,
> bookings, reviews, customer_support_logs, payments, properties,
> booking_updates, destinations, hosts). Stack: Python, FastAPI backend,
> React frontend, ChromaDB for a local metadata vector index, an external
> LLM API (OpenAI or Anthropic) for SQL generation and answer summarization,
> Databricks SQL Warehouse as the query target. Repo root contains `/backend`
> and `/frontend`.

---

## Ticket 1 — Databricks connectivity smoke test

**Goal:** Prove we can authenticate to Databricks and run a query from
Python, before any AI or web code exists.

**Do:**
- Create `/backend/scripts/db_smoke_test.py`.
- Use `databricks-sql-connector` to connect to a Databricks SQL Warehouse
  using credentials from environment variables: `DATABRICKS_SERVER_HOSTNAME`,
  `DATABRICKS_HTTP_PATH`, `DATABRICKS_TOKEN`.
- Load these from a `.env` file via `python-dotenv`; add `.env.example` with
  the three variable names (no real values) and add `.env` to `.gitignore`.
- Run `SELECT * FROM wanderbricks.bookings LIMIT 5` and print the results as
  a formatted table to stdout.
- Wrap the connection/query in a try/except that prints a clear error
  message if auth fails or the table isn't found (don't let it stack-trace
  raw).
- Add `requirements.txt` to `/backend` with `databricks-sql-connector` and
  `python-dotenv` pinned to current stable versions.

**Acceptance criteria:**
- [ ] Running `python backend/scripts/db_smoke_test.py` with valid `.env`
      values prints 5 rows from `bookings`.
- [ ] Missing/invalid credentials produce a readable error, not a raw
      traceback.
- [ ] No credentials are hardcoded anywhere in the script.

**Do not:** build any FastAPI, LLM, or React code in this ticket.

---

## Ticket 2 — Manual schema description authoring

**Goal:** Produce hand-written, accurate descriptions of all 9 tables. This
is a content-authoring ticket, not a coding ticket — but it produces a data
file later tickets depend on.

**Do:**
- Extend `db_smoke_test.py` (or add
  `/backend/scripts/inspect_schema.py`) to, for each of the 9 tables:
  run `DESCRIBE TABLE EXTENDED <table>` and `SELECT * FROM <table> LIMIT 5`,
  and print/save the output to `/backend/data/schema_raw/<table>.txt`.
- **Confirm the actual foreign key columns** for `properties` →
  `destinations` and `properties` → `hosts` (unconfirmed in the source
  diagram) by inspecting the `properties`, `destinations`, and `hosts`
  column lists for matching ID columns. Record what you find.
- Using the raw output, hand-author
  `/backend/data/table_descriptions.json`: an array of 9 objects, one per
  table, each with:
  ```json
  {
    "table_name": "bookings",
    "description": "One row per reservation made by a user for a property.",
    "columns": [
      {"name": "booking_id", "type": "string", "description": "Primary key for a booking."},
      {"name": "user_id", "type": "string", "description": "FK to users.user_id."},
      {"name": "status", "type": "string", "description": "One of: confirmed, cancelled, pending."}
    ],
    "joins": [
      {"to_table": "users", "on": "bookings.user_id = users.user_id"},
      {"to_table": "properties", "on": "bookings.property_id = properties.property_id"}
    ],
    "sample_values": {
      "status": ["confirmed", "cancelled", "pending"]
    }
  }
  ```
- Every categorical/enum-like column (status flags, types, tiers) must have
  a `sample_values` entry listing the distinct values actually observed in
  the `LIMIT 5`/`DESCRIBE` output — flag in a comment if 5 rows weren't
  enough to be confident and a larger sample was pulled.

**Acceptance criteria:**
- [ ] `table_descriptions.json` contains exactly 9 table objects.
- [ ] Every column in every table has a non-empty `description`.
- [ ] Every join relationship shown in the architecture diagram is present,
      with real, verified column names (not guessed).
- [ ] The `properties`↔`destinations` and `properties`↔`hosts` FK columns
      are explicitly confirmed, not assumed.

**Do not:** auto-generate this file with an LLM and skip human review — the
whole project's accuracy depends on this file being correct.

---

## Ticket 3 — Ingestion script (metadata → vector index)

**Goal:** Turn `table_descriptions.json` into a queryable local vector
index.

**Do:**
- Add `chromadb` and an embeddings client library (e.g. `openai`) to
  `requirements.txt`.
- Create `/backend/ingest.py`:
  - Load `table_descriptions.json`.
  - For each table, build one text document combining table description,
    all column names/types/descriptions, joins, and sample values.
  - Embed each document (one embedding call per table document; batch if
    the provider supports it).
  - Write to a local, persistent Chroma collection at
    `/backend/data/metadata_index/` (collection name: `wanderbricks_tables`),
    storing `table_name` as metadata on each entry.
  - Script should be idempotent — re-running it should not duplicate
    entries (delete-and-recreate the collection each run is fine for this
    scale).
- Create `/backend/scripts/test_retrieval.py`: takes a hardcoded test
  question ("which host had the most bookings?"), embeds it, queries the
  Chroma collection for top-3 matches, and prints the matched table names
  with similarity scores.

**Acceptance criteria:**
- [ ] `python backend/ingest.py` runs without error and creates
      `/backend/data/metadata_index/`.
- [ ] `python backend/scripts/test_retrieval.py` for "which host had the
      most bookings?" returns `bookings`, `hosts`, and `properties` in its
      top 3 — and does **not** return `clickstream`.
- [ ] Re-running `ingest.py` twice does not create duplicate entries (row
      count in the collection stays at 9).

---

## Ticket 4 — NL → SQL script (standalone, no server yet)

**Goal:** Get a working retrieve → generate-SQL loop runnable from the
terminal. This is where prompt iteration happens — keep it a script, not a
service, so it's fast to run repeatedly.

**Do:**
- Create `/backend/nl_to_sql.py` with a function
  `generate_sql(question: str) -> str` that:
  1. Embeds `question` and retrieves top-5 table documents from the Chroma
     collection built in Ticket 3.
  2. Builds an LLM system prompt containing only those retrieved table
     schemas (verbatim from `table_descriptions.json`, not paraphrased),
     an instruction that this is Databricks SQL / Spark SQL dialect, and an
     explicit instruction to use only the provided tables/columns.
  3. Calls the LLM (env var `LLM_API_KEY`, model name from env var
     `LLM_MODEL`) asking for SQL only, inside a ` ```sql ` fenced block.
  4. Parses the SQL out of the fenced block with a regex and returns it as
     a string.
- Add a `if __name__ == "__main__"` block that takes a question as a CLI
  arg, prints the retrieved table names, then the generated SQL.
- Test manually with at least 5 varied questions covering different join
  paths (e.g. "how many bookings does each destination have", "what's the
  average payment amount for cancelled bookings", "list the 5 most recent
  customer support logs"). Save these 5 questions and the SQL they
  produced to `/backend/data/manual_test_log.md` for your own reference —
  this becomes the seed of a future eval set.

**Acceptance criteria:**
- [ ] `python backend/nl_to_sql.py "which host had the most bookings?"`
      prints retrieved tables and a syntactically plausible SQL query.
- [ ] At least 5 manually-tested questions and their generated SQL are
      logged in `manual_test_log.md`.
- [ ] Generated SQL only references tables/columns that exist in
      `table_descriptions.json`.

**Do not:** execute the SQL against Databricks yet — that's Ticket 5.

---

## Ticket 5 — SQL execution + guardrails

**Goal:** Safely run the LLM-generated SQL against Databricks.

**Do:**
- Create `/backend/sql_guard.py` with a function
  `validate_sql(sql: str) -> str` (returns the sanitized/validated SQL or
  raises `SQLGuardError`) that enforces:
  - Statement is a single `SELECT` (reject `INSERT`/`UPDATE`/`DELETE`/
    `DROP`/`ALTER`/`MERGE`/`CREATE`, case-insensitive).
  - No statement chaining — reject if a semicolon appears anywhere except
    optionally as the very last character.
  - Every table name referenced appears in the whitelist loaded from
    `table_descriptions.json`.
  - If no `LIMIT` clause is present, append `LIMIT 200`.
- Create `/backend/db_executor.py` with `run_query(sql: str) -> list[dict]`
  that opens a Databricks connection (reuse Ticket 1's connection logic),
  sets a statement timeout (e.g. 30s), runs the validated SQL, and returns
  rows as a list of dicts (column name → value). Raise a clear
  `QueryExecutionError` on failure, capturing the Databricks error message.
- Wire these into `nl_to_sql.py`: after generating SQL, call
  `validate_sql`, then `run_query`. On a `QueryExecutionError`, feed the
  error message back to the LLM with the original question and schemas,
  ask it to produce a corrected query, retry up to 2 times, then give up
  with a clear failure message.
- Update the CLI entry point to print the final row results (or the
  failure reason) instead of just the SQL.

**Acceptance criteria:**
- [ ] A valid generated query executes and prints real rows from
      Databricks.
- [ ] Manually feeding `validate_sql` a `DROP TABLE` or `DELETE` statement
      raises `SQLGuardError` and never reaches `run_query`.
- [ ] Manually feeding it two chained statements
      (`SELECT 1; DROP TABLE users`) is rejected.
- [ ] A query missing `LIMIT` is auto-limited before execution.
- [ ] Deliberately triggering a bad-SQL error (e.g. by asking an
      unanswerable question) shows at least one retry attempt in the logs
      before failing or succeeding.

---

## Ticket 6 — Summarization (results → natural-language answer)

**Goal:** Complete the terminal pipeline: question → answer.

**Do:**
- Add `summarize_results(question: str, rows: list[dict]) -> str` to
  `nl_to_sql.py` (or a new `summarize.py`): builds a prompt with the
  original question and the result rows (cap at first 50 rows; if more
  were returned, tell the LLM the results were truncated), and asks for a
  concise natural-language answer.
- Wire into the CLI flow: retrieve → generate SQL → validate → execute
  (with repair retries) → summarize → print the final natural-language
  answer as the primary output, with SQL and row count shown as secondary
  debug info.
- Handle the zero-rows case explicitly — the summarizer should say no
  matching data was found, not hallucinate an answer.

**Acceptance criteria:**
- [ ] `python backend/nl_to_sql.py "which host had the most bookings?"`
      prints a plain-English answer as the primary output.
- [ ] A query that legitimately returns 0 rows produces an answer that
      says so, not a fabricated number.
- [ ] The full pipeline (retrieve → SQL → execute → summarize) completes
      for all 5 questions logged in Ticket 4's `manual_test_log.md`;
      update that log with the final answers.

---

## Ticket 7 — Wrap the pipeline in FastAPI

**Goal:** Expose the working terminal pipeline as an HTTP API.

**Do:**
- Add `fastapi` and `uvicorn` to `requirements.txt`.
- Create `/backend/main.py`:
  - `POST /chat` — request body:
    ```json
    { "messages": [{"role": "user", "content": "which host had the most bookings?"}] }
    ```
    (accept a full message history array now, even though v1 logic will
    only use the latest user message, so the frontend doesn't need to
    change later).
    Response body:
    ```json
    {
      "answer": "...",
      "sql": "SELECT ...",
      "retrieved_tables": ["bookings", "hosts", "properties"],
      "row_count": 12
    }
    ```
  - On any failure (guardrail rejection, exhausted retries, execution
    error), return HTTP 200 with an `"error"` field in the JSON body
    containing a user-safe message — do not leak raw SQL error text or
    stack traces to the client. Log full details server-side.
  - `GET /health` — returns `{"status": "ok"}`, used to verify the metadata
    index loaded successfully at startup.
  - Load the Chroma collection once at app startup (not per-request).
  - Add CORS middleware allowing the frontend's local dev origin (e.g.
    `http://localhost:5173` or `:3000` — confirm against the frontend
    tooling chosen in Ticket 8).
- Refactor `nl_to_sql.py`'s pipeline into an importable function so
  `main.py` calls it rather than duplicating logic.

**Acceptance criteria:**
- [ ] `uvicorn backend.main:app --reload` starts without error.
- [ ] `curl localhost:8000/health` returns `{"status": "ok"}`.
- [ ] `curl -X POST localhost:8000/chat -d '{"messages":[{"role":"user","content":"which host had the most bookings?"}]}' -H "Content-Type: application/json"`
      returns a JSON response matching the schema above with a real
      answer.
- [ ] Sending a message that triggers a guardrail rejection (e.g. asking
      it to delete something) returns HTTP 200 with a clear `error` field,
      not a 500.

---

## Ticket 8 — React chat UI

**Goal:** A minimal chat interface against the working API.

**Do:**
- Scaffold `/frontend` with Vite + React (plain JS or TS, your choice —
  keep consistent).
- Single-page app, one component tree:
  - `App` holds `messages: {role, content}[]` state.
  - `MessageList` renders the conversation, distinguishing user vs.
    assistant messages visually.
  - `InputBar` — text input + submit button (and Enter-to-send).
- On submit: append the user message to state, POST the full `messages`
  array to `http://localhost:8000/chat`, append the returned `answer` as
  an assistant message on success, or a visibly-styled error message using
  the response's `error` field on failure.
- Show a loading indicator on the pending request (disable input while
  waiting).
- Render `sql` and `retrieved_tables` from the response inside a collapsed
  `<details>` under each assistant message ("Show query used").
- Add a `.env` / Vite env var for the API base URL rather than hardcoding
  `localhost:8000`.
- No routing, no auth, no message persistence across page reloads.

**Acceptance criteria:**
- [ ] `npm run dev` starts the app and it loads with an empty chat and
      visible input box.
- [ ] Submitting a question shows a loading state, then renders the
      assistant's answer.
- [ ] The "Show query used" detail correctly displays the SQL from that
      response.
- [ ] A backend error response renders as a visibly different (e.g.
      red-toned) message instead of crashing the UI or showing nothing.
- [ ] Refreshing the page starts a new, empty conversation (no persistence
      expected in this ticket).

---

## Ticket 9 — Retry loop, logging, and query-visibility polish

**Goal:** Close the loop between what Ticket 5's retry logic does server-
side and what's visible for debugging/trust, plus basic operational
logging.

**Do:**
- In `main.py`, log (to stdout or a rotating file, `/backend/logs/app.log`)
  for every `/chat` call: timestamp, the user's question, retrieved table
  names, the final SQL used, number of repair retries taken, row count,
  and total latency in ms. One structured (JSON) log line per request.
- If a repair retry occurred, include a `"repaired": true` flag in the
  `/chat` response so the frontend can optionally surface "I had to fix my
  query once" — add a small, non-intrusive note in `MessageList` when this
  flag is present.
- Add a basic request-level try/except in `main.py` so any unhandled
  exception still returns the standard `{"error": "..."}` shape instead of
  an unhandled 500 with a stack trace to the client.
- Write a short `/backend/README.md` and `/frontend/README.md` covering:
  required env vars, how to run `ingest.py`, how to start the backend, how
  to start the frontend, and where the logs live.

**Acceptance criteria:**
- [ ] Every `/chat` call produces one structured log line with all fields
      listed above.
- [ ] Deliberately asking a question likely to produce a first-pass SQL
      error and succeed on retry shows `"repaired": true` in the response
      and a corresponding note in the UI.
- [ ] An unhandled backend exception (simulate by temporarily breaking the
      DB connection) still returns a clean JSON error to the frontend, not
      a raw 500/stack trace.
- [ ] Both README files let a new developer go from clone to a working
      chat in under 10 minutes, assuming they already have Databricks and
      LLM credentials.

---

## Handoff notes for the coding agent

- Work tickets in order; do not start Ticket *n+1* until Ticket *n*'s
  acceptance criteria are checked off and you've had a chance to run it
  yourself.
- Tickets 1–6 are deliberately CLI/script-only — resist the urge to jump to
  FastAPI or React early, even if it seems faster. Prompt and SQL quality
  problems are much cheaper to debug from a terminal loop.
- Where a ticket says "confirm" or "verify" (Tickets 1 and 2 especially),
  that step is not optional scaffolding — it produces facts later tickets
  depend on being correct.
- Do not add authentication, streaming responses, persistence, or a
  managed vector store in this MVP — those are explicitly out of scope
  (see the parent plan's "Path to production" section) and would be
  premature here.
