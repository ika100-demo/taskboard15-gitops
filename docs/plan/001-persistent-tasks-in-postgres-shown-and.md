---
plan_id: 001-persistent-tasks-in-postgres-shown-and
spec: 001-persistent-tasks-in-postgres-shown-and
feature: Persistent tasks in Postgres, shown and created in the web UI
gitops_app: ika100-demo/taskboard15-gitops
status: draft
repos:
  - id: taskboard15-gitops
    shape: gitops-app
    summary: Postgres for the task API, two API replicas in dev, the API address for the web UI
    acs: [AC-001.1, AC-001.2, AC-001.4]
    order: before-merge
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: taskboard15-api}
      - {replicas: 2, component: taskboard15-api, instance: dev}
      - {env: {API_URL: 'http://taskboard15-api'}, component: taskboard15-web}
    depends_on: []
    done: false
  - id: taskboard15-api
    shape: service-python
    summary: Store tasks in Postgres instead of memory
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.6]
    criteria:
      - "Given tasks A, B and C were created through `POST /tasks`, when the service restarts against the same database, then `GET /tasks` returns `200` with exactly A, B and C in creation order, each with the same `id`, `title` and `due_date` as returned on creation (system AC-001.1) (replaces AC-001.8)"
      - "Given an empty database, when a client sends `GET /tasks`, then the response is `200` with an empty JSON array (system AC-001.1) (replaces AC-001.6)"
      - "Given two service instances use the same database, when a task is created through `POST /tasks` on one, then `GET /tasks` on the other contains it with the same `id`, `title` and `due_date` (system AC-001.2)"
      - "Given two service instances start at the same time against an empty database, when both have reported ready, then `POST /tasks` and `GET /tasks` succeed on each of them (system AC-001.2)"
      - "Given the database cannot be reached, when a client sends `GET /tasks` or `POST /tasks`, then the response is `503` with the JSON body `{\"detail\": \"database unavailable\"}` within 5 seconds, and once the database is back `GET /tasks` does not contain the task that was posted (system AC-001.3)"
      - "Given the database cannot be reached, when `GET /ready` is called, then it answers `503` with `{\"status\": \"not ready\"}` while `GET /health` still answers `200`; when the database is reachable again, `GET /ready` answers `200` with `{\"status\": \"ready\"}` without a restart (system AC-001.3)"
      - "Given the database is reachable, when a client sends `POST /tasks` with `{\"title\": \"Write report\", \"due_date\": \"2026-12-01\"}`, then the response is `201` with the task and the following `GET /tasks` lists it last (system AC-001.6)"
    arguments: |
      Connection from DATABASE_URL (PG* are set as well), injected by the gitops entry; no other config.
      Schema creation must be idempotent and safe when two replicas start at once (dev runs 2).
      Creation order is kept by an internal column (sequence or created_at); it is not exposed (system A1).
      The 422 validation and the existing criteria AC-001.1 to AC-001.5 and AC-001.7 stay unchanged.
    depends_on: []
    done: false
  - id: taskboard15-web
    shape: web-nextjs
    summary: Task list and create form on the start page
    acs: [AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8]
    criteria:
      - "Given the task API returns tasks for `GET /tasks`, when a person opens `/`, then the page lists every task in the order the API returns them (oldest first), each showing its `title` and its `due_date` as `YYYY-MM-DD`, or \"No due date\" when `due_date` is `null` (system AC-001.4)"
      - "Given the task API returns an empty array for `GET /tasks`, when a person opens `/`, then the page shows \"No tasks yet\" and the create form with a title field and a date input for the due date (system AC-001.5)"
      - "Given `/` is open, when a person enters the title \"Write report\" and the due date 2026-12-01 and submits, then the web UI sends `POST /tasks` with `{\"title\": \"Write report\", \"due_date\": \"2026-12-01\"}`, and on `201` the returned task appears at the end of the list without a page reload and the form is cleared; a reload lists it from `GET /tasks` (system AC-001.6)"
      - "Given `/` is open, when a person submits with an empty or whitespace-only title, then no `POST /tasks` is sent, the form keeps the entered values, and next to the title field the page shows \"Title is required\" (system AC-001.7)"
      - "Given `/` is open, when a person submits a title longer than 200 characters, then no `POST /tasks` is sent, the form keeps the entered values, and next to the title field the page shows \"Title must be at most 200 characters\" (system AC-001.7)"
      - "Given the task API answers `POST /tasks` with `422` naming the `title` field, when the person submitted the form, then no task is added to the list, the form keeps the entered values, and next to the title field the page shows \"Title is required\" or, for a title over 200 characters, \"Title must be at most 200 characters\" (system AC-001.7)"
      - "Given the task API is unreachable, does not answer within 5 seconds, or answers `GET /tasks` with a status other than `200`, when a person opens `/`, then the page shows \"Tasks could not be loaded. Try again later.\" instead of an error page (system AC-001.8)"
      - "Given the task API is unreachable, does not answer within 5 seconds, or answers `POST /tasks` with a status other than `201` or `422`, when a person submits the form, then the page shows \"Task could not be saved. Try again.\", the list is unchanged, and the form keeps the entered values (system AC-001.8)"
    arguments: |
      The task API is called only server-side (Server Component, Server Action or route handler) at API_URL; the browser
      never calls the API, and API_URL is never a NEXT_PUBLIC_ variable. Document API_URL in docs/env-vars.md.
      /api/ready must not depend on the task API, so that an API outage shows the message instead of taking the UI down.
      Count title length in Unicode code points, as the API does (not UTF-16 units).
    depends_on: []
    done: false
gitops_pin: []
---

## taskboard15-gitops — Postgres for the task API, two API replicas in dev, the API address for the web UI

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.4 · **Depends on:** []

Lands before the component PRs merge, so that the API's new image finds its database on first start. It declares the
system's one Postgres (system A9) and gives it only to `taskboard15-api` (`DATABASE_URL`, `PG*`), which is the storage
half of AC-001.1. AC-001.2 needs more than one replica to be real, so dev runs the API with `replicas: 2`. The web UI
gets the API's in-cluster address `http://taskboard15-api` (Service port 80) as `API_URL`, which AC-001.4 needs to
load the list. Exposure already exists for both components in dev; the readiness probe stays `/ready`, which the API
now ties to the database (AC-001.3).

## taskboard15-api — Store tasks in Postgres instead of memory

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.6 · **Depends on:** []

The HTTP contract stays as it is (system A1); only the store changes from memory to Postgres. The API owns durability
across restarts (AC-001.1), one shared state for every replica (AC-001.2), the `503` and readiness behaviour on a
database outage (AC-001.3), and the "still in `GET /tasks`" part of AC-001.6. Its spec supersedes the in-memory
criteria AC-001.8 (lost on restart) and AC-001.6 (empty since start) of its spec 001.

## taskboard15-web — Task list and create form on the start page

**Shape:** `web-nextjs` · **Criteria:** AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8 · **Depends on:** []

The web UI has no features yet; this adds the list and the form on `/` (system A4), checks titles before sending with
the API's rules (A5), uses the browser's date input (A6), and shows the exact English messages (A7). It reads tasks
only through the task API (A9). It builds in parallel with the API against the contract below; the contract is the
existing API one, so nothing waits.

Decomposition: one repo per concern the spec names (storage in the API, the page in the web UI, wiring in the gitops
repo); no library, since the two components share no code, only the HTTP contract.

## Contract

Caller: `taskboard15-web` (server side), at `API_URL` (`http://taskboard15-api` in the cluster). Callee:
`taskboard15-api`. JSON, UTF-8, no authentication (system A10, dev only).

### `GET /tasks`

- `200`: a bare JSON array of tasks, creation order, oldest first, no upper bound, no pagination.
- `503`: database unreachable, body `{"detail": "database unavailable"}`.

### `POST /tasks`

Request body:

| Field | Type | Rules |
|---|---|---|
| `title` | string, required | 1 to 200 characters (Unicode code points), at least one non-whitespace character; stored as sent, not trimmed |
| `due_date` | string or `null`, optional | calendar date `YYYY-MM-DD`; absent or `null` means no due date; past dates accepted |

Responses:

- `201`: the created task.
- `422`: invalid input, FastAPI's standard body `{"detail": [{"loc": ["body", "<field>"], "msg": "...", "type": "..."}]}`;
  the field is `loc[1]` (`title` or `due_date`). No task is created.
- `503`: database unreachable, body `{"detail": "database unavailable"}`. No task is created.

### Task

| Field | Type | Notes |
|---|---|---|
| `id` | string | server-generated, opaque, unique, never reused; clients cannot set it |
| `title` | string | as sent |
| `due_date` | string `YYYY-MM-DD` or `null` | |

### Probes (cluster only)

- `GET /ready`: `200 {"status": "ready"}` when the database answers; `503 {"status": "not ready"}` otherwise.
- `GET /health`: `200` while the process runs, independent of the database.

The web UI's `/api/ready` and `/api/health` do not call the task API.

No events.

### Errors

- Error bodies are JSON with a `detail` field: a string for `503`, FastAPI's list for `422`.
- The web UI treats `422` naming `title` as a field error at the title field (its messages, see its criteria), and any
  other `422` as a failed save.
- Any response the contract does not list for a call (other `4xx`, `5xx`, a non-JSON body, a body not matching the
  shapes above), a connection error and a timeout count as "the API answers with an error": on `GET /tasks` the page
  shows "Tasks could not be loaded. Try again later.", on `POST /tasks` it shows "Task could not be saved. Try again."
  and keeps the form values. The page itself never answers with an error page for this.
- The web UI logs each such failure with the status or error kind, never the task title.

### Timeouts

- Web to API, `GET /tasks`: 5 s total, no retry; on timeout the load message is shown.
- Web to API, `POST /tasks`: 5 s total, no retry (a retry could create the task twice); on timeout the save message is
  shown, and a later reload shows whether the task was stored.
- API to Postgres: connect timeout 2 s, statement timeout 3 s; on either, the request answers `503` within 5 s.
- `/ready` checks the database with a 1 s timeout, so a down database takes every replica out of rotation; the
  Service then refuses connections, which the web UI handles as unreachable. The API reconnects by itself when the
  database is back, without a restart.

## gitops-app PR

The entry PR (`taskboard15-gitops`, before-merge) adds the `postgres` resource, `uses: postgres` on `taskboard15-api`,
`replicas: 2` for the API in dev and `API_URL` for the web UI; `/system:ship` applies them with `cplat`. No pins: dev
tracks `main`, and the spec leaves staging and prod out (non-goal; promotion follows the normal pin policy, and access
control is required first).

After Argo reconciles dev, verify:

- the Postgres instance is ready and the API pods have `DATABASE_URL`; both API replicas report `/ready` `200`;
- `taskboard15-web` has `API_URL=http://taskboard15-api`;
- a task created through `POST /tasks` is still in `GET /tasks` after `kubectl rollout restart` of the API;
- `http://taskboard15-web.taskboard15-dev.localhost/` lists tasks and creates one.
