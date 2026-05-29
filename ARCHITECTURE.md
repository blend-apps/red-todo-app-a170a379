# Architecture

Red Todo App is intentionally simple. There are exactly three moving parts.

## Request flow

```
Browser
  │  GET /          → Express serves public/index.html
  │  GET /style.css → Express serves public/style.css
  │  GET /app.js    → Express serves public/app.js
  │
  │  GET  /api/todos          ┐
  │  POST /api/todos          │
  │  PATCH /api/todos/:id/... ├─→ server.js handler → db.js → SQLite file
  │  DELETE /api/todos/:id    ┘
  │
  └  GET /health   → { ok: true }
```

There is no build step, no bundler, no compilation. Static files are served
directly from `public/` by Express's built-in static middleware.

## Files

### `server.js`
Express application. Registers five routes, serves the `public/` directory as
static files, and listens on `PORT` (default 3000).

### `db.js`
Thin wrapper around `better-sqlite3`. Opens (or creates) the SQLite database at
`DB_PATH`, runs the schema migration on startup (idempotent `CREATE TABLE IF NOT
EXISTS`), and exports four functions:

- `getAllTodos()` — `SELECT` all rows ordered newest first
- `createTodo(text)` — `INSERT` a row, return the new record
- `toggleTodo(id)` — flip the `done` bit, return the updated record
- `deleteTodo(id)` — `DELETE` the row

### `public/index.html`
Single-page shell. Contains the "CHANGE ME" banner, the `<header>`, an `<input>`
form, a `<ul>` for the list, and a footer. All dynamic content is injected by
`app.js`.

### `public/style.css`
All visual styling. A `:root` block at the top defines CSS custom properties
(`--color-bg`, `--font-family`, etc.) that control the entire look of the app.
Changing these six lines is all it takes to retheme the app completely.

### `public/app.js`
Vanilla JavaScript. Fetches the initial todo list from `/api/todos`, renders it,
and wires up `submit` / `change` / `click` handlers to the REST API. No
framework, no transpilation — runs directly in any modern browser.

## Database

SQLite with WAL mode enabled. Schema:

```sql
CREATE TABLE IF NOT EXISTS todos (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  text       TEXT    NOT NULL,
  done       INTEGER NOT NULL DEFAULT 0,
  created_at TEXT    NOT NULL DEFAULT (datetime('now'))
);
```

The database file location is controlled by `DB_PATH` (default `./data/todos.db`).
In production Docker deployments, mount a volume at `/app/data` to persist data
across container restarts. In the Blend dev environment, `/workspace/data` is
ephemeral (reset when the dev env is torn down).

## Dev server (no file watcher)

`npm run dev` runs `node server.js` without `--watch`. Static files in `public/` are
served from disk on every request, so a browser reload picks up HTML/CSS/JS changes.
Changes to `server.js` or `db.js` require the Blend agent to call `restart_dev_server`.

## Blend dev environment

When Blend provisions a dev env for this app, it:

1. Pulls `ghcr.io/blend-apps/blend-dev-red-todo:latest` (built from `Dockerfile.dev`)
2. Runs `blend-dev-runtime`'s `entrypoint.sh`, which:
   - Syncs `/workspace` to the user's fork at the requested branch
   - Starts `blend-file-daemon` on port 9999 (live file read/write)
   - Runs `npm install` (fast — `node_modules` are pre-baked, nothing to do if `package-lock.json` unchanged)
   - Starts `node server.js` (restarts only on explicit `restart_dev_server` or crash)
3. Exposes the app on port 3000 and `blend-file-daemon` on port 9999

The Blend AI agent edits files in `/workspace` via `blend-file-daemon`. For CSS/JS/HTML,
a browser reload is enough. For backend changes, the agent calls `restart_dev_server`.

## Customization surface area

Everything a Blend customization session might reasonably change:

| File | What to change |
|---|---|
| `public/style.css` | CSS variables (`:root` block): colors, font, spacing |
| `public/index.html` | Page structure, copy, the "CHANGE ME" banner |
| `public/app.js` | Frontend behavior (sorting, filtering, animations) |
| `server.js` | API behavior, add new routes |
| `db.js` | Data model (add columns, new queries) |

Adding a new column requires updating:
1. The `CREATE TABLE` statement in `db.js` (won't apply to existing DB — add an `ALTER TABLE` migration)
2. The relevant query functions in `db.js`
3. The HTML/JS in `public/`
