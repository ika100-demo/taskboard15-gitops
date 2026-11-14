---
spec_id: 001-persistent-tasks-in-postgres-shown-and
title: Persistent tasks in Postgres, shown and created in the web UI
status: draft
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres, shown and created in the web UI

## Problem

The task API keeps tasks only in its memory: every restart, redeploy or new replica loses or splits them, so nobody can rely on the board. People also have no way to see or add tasks except by calling the API by hand, because the web UI has no features yet. This spec makes tasks durable in a Postgres database and gives people a page in the web UI where they see all tasks and create new ones.

## Stories

As a team member, I want tasks I created to still be there after the system restarts or is redeployed, so that I can rely on the board as the record of our work.
**Repos:** taskboard15-api, taskboard15-gitops

As a team member, I want to open the web UI and see all tasks with their due dates, so that I know what has been recorded without calling the API.
**Repos:** taskboard15-web, taskboard15-api, taskboard15-gitops

As a team member, I want to create a task with a title and an optional due date from the web UI, so that I can record work without tools.
**Repos:** taskboard15-web, taskboard15-api

## Acceptance criteria

- **AC-001.1** Given tasks A, B and C were created through `POST /tasks`, when the task API restarts or is redeployed, then `GET /tasks` returns `200` with exactly A, B and C in creation order, each with the same `id`, `title` and `due_date` as returned on creation.
- **AC-001.2** Given the task API runs with more than one replica, when a task is created through any of them, then every following `GET /tasks`, whichever replica answers, contains that task.
- **AC-001.3** Given the database cannot be reached, when a client sends `GET /tasks` or `POST /tasks`, then the response is `503` with a JSON error body, no task is created, and the API's readiness check reports not ready until the database is reachable again.
- **AC-001.4** Given tasks exist, when a person opens the web UI's start page, then it lists every task in creation order (oldest first), each showing its title and its due date as `YYYY-MM-DD`, or "No due date" when it has none.
- **AC-001.5** Given no task exists, when a person opens the web UI's start page, then it shows the text "No tasks yet" and the form to create a task.
- **AC-001.6** Given the start page is open, when a person enters the title "Write report" and the due date 2026-12-01 and submits the form, then the new task appears at the end of the list without the person reloading the page, the form is cleared, and the task is still listed after a reload of the page and in `GET /tasks`.
- **AC-001.7** Given the start page is open, when a person submits the form with an empty or whitespace-only title or one longer than 200 characters, then no task is created, the form keeps the entered values, and a message next to the title field says "Title is required" (empty or whitespace) or "Title must be at most 200 characters" (too long).
- **AC-001.8** Given the task API is unreachable or answers with an error, when a person opens the start page or submits the form, then the page shows "Tasks could not be loaded. Try again later." (when loading) or "Task could not be saved. Try again." (when saving) instead of an error page, and on a failed save the form keeps the entered values.

## Non-goals

- Editing, completing or deleting tasks, in the API or the web UI.
- Login, users, task ownership or any access control: in dev anyone who reaches the web UI or the task API can read and create tasks; access control is a separate spec, required before staging or prod.
- Filtering, sorting options, search or pagination of the list.
- Live updates of the list when someone else creates a task (a reload shows them).
- Carrying over tasks that exist only in the API's memory when this change is deployed.
- Database backups and restore (a separate decision about `backup:`).
- Running the database or the web UI in staging or prod as part of this spec; it targets dev, promotion follows the normal pin policy.

## Assumptions

- **A1** The task API contract stays as it is (`/tasks`, fields `id`, `title`, `due_date`, `422` on invalid input, list oldest first); only storage changes (other option: extend the contract now, for example with `created_at`; affects AC-001.1, AC-001.4).
- **A2** Tasks held in the API's memory at the moment of the switch are not carried over; they were documented as lost on restart anyway (other option: export and import them once; affects AC-001.1).
- **A3** A database outage answers `503` with a JSON error body and makes readiness report not ready, so the API is taken out of rotation instead of answering `500` (other option: keep ready and answer `500`; affects AC-001.3).
- **A4** The task list and the create form live on the web UI's start page (`/`), a single page (other option: a separate `/tasks` page; affects AC-001.4 to AC-001.8).
- **A5** The web UI checks the title (required, not whitespace-only, at most 200 characters) before sending, with the same rules as the API; a `422` from the API is shown at the field it names (other option: rely only on the API's answer; affects AC-001.7).
- **A6** The due date is entered with the browser's date input, so an invalid calendar date cannot be submitted; the date is shown as `YYYY-MM-DD` (other option: free text, or a localized date format; affects AC-001.4, AC-001.6).
- **A7** Messages are in English with the exact wording in AC-001.5, AC-001.7 and AC-001.8 (other option: other wording or a German UI; affects AC-001.5, AC-001.7, AC-001.8).
- **A8** Every task is listed on one page with no upper bound (other option: show the newest 100; affects AC-001.4).
- **A9** The system gets one Postgres database, used only by the task API; the web UI reaches tasks only through the task API, never the database (other option: the web UI reads the database too; affects AC-001.4, AC-001.6).
- **A10** The feature ships to dev without login; anyone who can reach the web UI or the task API can read and create all tasks (other option: access control first; affects AC-001.4, AC-001.6) (answered by the user 2026-11-14).

## Notes for the architect

- The database is declared with `/resource:add postgres` and given to the task API with `uses: [postgres]`; the web UI gets the task API's in-cluster address through `env` in `system.yaml` (for example `API_URL`).
- The task API's own spec 001 (`001-create-and-list-in-memory-tasks`, in taskboard15-api) has AC-001.8 "tasks are lost on restart"; the API's component spec derived from this one must supersede it.

## Open questions

- ~~Anyone who can reach the web UI or the task API can read all tasks and create new ones, and these tasks now persist in a database; both components are exposed in dev. Is it acceptable to ship this without any login or access control? (suggested: yes for dev, as stated in the non-goals; access control is a separate spec before exposing staging or prod; affects AC-001.4, AC-001.6, new criterion)~~ Answered 2026-11-14: yes for dev, ship without login; access control becomes a separate spec before staging or prod (non-goals, A10).

## Changelog

- 2026-11-14 created
- 2026-11-14 amended: open question on access control answered (no login in dev; access control is a separate spec before staging or prod), folded into the non-goals and A10
