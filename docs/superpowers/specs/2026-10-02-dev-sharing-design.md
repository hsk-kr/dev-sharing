# Dev Sharing — design

Date: 2026-10-02
Status: draft, awaiting review

## Goal

A small dev team (6 people) reads and discusses the **same** AI and software-engineering news, so everyone grows in the same direction instead of each person following a different stream.

An agent publishes 5 curated posts per weekday. Members comment on each post (or mark "nothing to add"). The UI shows, per member, whether they are caught up on the last 7 days.

## Non-goals (v1)

- Notifications (Slack, email, push). The web UI is the product.
- Passwords or real auth. Members pick their name.
- Cloud hosting. It runs on one laptop.
- Multiple teams, roles, admin screens.

## Product rules

| Rule | Decision |
|---|---|
| Members | Sojeong, Seongkuk, Henry, Pablo, Pardeep, Ben. Seeded from `config/members.json` (name, 2-letter code, colour). |
| Identity | First visit: pick your name. Stored in a cookie. Switch from the header. No password. |
| Schedule | Mon–Fri, slots **10:00, 11:00, 14:00, 15:00, 16:00**, timezone `Europe/London` (configurable). |
| Agent output | Each slot publishes **exactly one** post: the single best item found. 5 agent posts per weekday. |
| Missed slots | If the laptop was asleep, today's due-but-not-run slots run on wake, **sequentially**. Missed slots from previous days are skipped. |
| Caught up (member × post) | Member has ≥1 comment on the post, **or** a "nothing to add" mark, **or** is the member who submitted it. Derived, never stored as a flag. Deleting your only comment reverts you to unread. |
| "Nothing to add" | One click; can be undone. Button hidden once you have a comment. |
| Catch-up window | Rolling 7 days (posts created in the last 7×24h). Older posts move to the Archive: readable and commentable, not counted. |
| Member posts | Unlimited. Not counted in the agent's 5/day. Count toward everyone's catch-up. |
| Comments | Always visible. Flat list. Edit/delete your own only. |

## Post content

Every post, from the agent or a member, has the same shape:

| Field | Notes |
|---|---|
| `title` | Short headline. |
| `url` | Link to the **primary source** (not a roundup or newsletter). |
| `source` | Site or org name, e.g. `anthropic.com`. |
| `summary` | What it is. 2–3 sentences. |
| `why_it_matters` | The angle for software engineers at a startup. 1–3 sentences. |
| `discussion_question` | One question to start the conversation. |
| `tags` | 0–3 short tags. |
| `published_at` | Date the source was published (`YYYY-MM-DD`), if known. |

Server-side fields: `id`, `origin` (`agent` \| `member`), `submitted_by` (member id, null for agent), `run_id` (agent only), `created_at` (UTC).

**One schema, three uses.** A single zod 4 schema `PostInput` is the source of truth for:

1. The agent's `--json-schema` (via `z.toJSONSchema()`).
2. Validation on the Submit page and `POST /api/posts`.
3. The copy-ready prompt shown to teammates on the Submit page (generated from the schema, including an example JSON).

## Agent taste (prompt rules)

- **Focus:** software engineering, startups, and the big AI events everyone is talking about. Dev tooling and workflows first: coding agents (Claude Code, Cursor, Codex…), MCP, new dev practices. A major model release or industry shift beats a minor tool update.
- **Skip:** listicles, hype pieces, crypto-AI, research papers (unless the whole industry is talking about one), roundups and newsletters (link the primary source instead).
- **Recency:** published in the last 48 hours; 72 hours on Mondays, to cover the weekend.
- **No repeats:** the prompt includes title + URL of every post from the last 30 days, and the instruction not to cover the same story again even from another URL.
- **Always one:** return exactly one post, the best available.
- The prompt also states today's date and the slot time.

## Architecture

One Bun process, started from a terminal with `bun start`:

```
                 ┌──────────────────────── bun process ─────────────────────────┐
browser ──ngrok──▶ Bun.serve ─┬─ /api/*  → Hono API ──┐                         │
                 │            └─ /*      → React SPA   │                         │
                 │                                     ▼                         │
                 │  scheduler (60s tick) ──▶ agent runner ──▶ SQLite (bun:sqlite)│
                 │                              │                                │
                 └──────────────────────────────┼────────────────────────────────┘
                                                ▼
                                   spawn `claude -p` (Opus 5.5)
```

| Unit | Responsibility | Depends on |
|---|---|---|
| `config` | Members, slots, timezone, port. | — |
| `db` | Schema, migrations-on-start, typed query functions. | bun:sqlite |
| `domain` | Pure logic: URL normalisation, caught-up derivation, 7-day window, slot-due calculation. No I/O. | — |
| `schema` | `PostInput` zod schema, JSON Schema export, teammate prompt generator. | zod |
| `api` | Hono routes; reads member from cookie; enforces own-comment rules. | db, domain, schema |
| `scheduler` | Every 60s: find today's due slots not yet successful; run them one at a time under an in-process lock. | domain, db, agent |
| `agent` | Build prompt, spawn Claude, parse and validate output, insert post. Behind an interface so tests can swap in a fake. | schema, db |
| `web` | React SPA. | api |

### Stack

- Bun 1.3: runtime, `bun:sqlite`, `bun test`, bundler via HTML imports (no Vite).
- Hono for the API.
- React 19, React Router 7, TanStack Query (refetch on focus and every 30s so the board stays fresh), Tailwind CSS v4.
- zod 4.
- TypeScript throughout.

## Agent runner

Verified on 2026-10-02 with Claude Code 2.1.287 (probe run in an empty dir):

```bash
claude -p --model claude-opus-5-5 \
  --tools "WebSearch,WebFetch" --allowedTools "WebSearch,WebFetch" \
  --output-format json --json-schema '<PostInput JSON Schema>' \
  --no-session-persistence --setting-sources "" --strict-mcp-config
# prompt on stdin
```

Probe findings:

- Authenticates with the logged-in Claude subscription. No `ANTHROPIC_API_KEY` needed.
- Do **not** use `--bare`: it forces API-key auth.
- Result is a single JSON object. Check `subtype == "success"` and `is_error == false`, then read `structured_output`.
- A one-search run took ~19 s. A real curation run will take longer; hard timeout **10 minutes** (process killed).
- There is no `--max-turns` flag in this version. The timeout is the runaway bound.

**Isolation (hard requirement).** The spawned session must not inherit the owner's personal Claude config: hooks/plugins (e.g. memory plugins would record every scheduled run), MCP servers (browser automation, company connectors). Measures:

- `--strict-mcp-config` with no `--mcp-config` → no MCP servers.
- `--setting-sources ""` → no user/project settings, so no plugins or hooks.
- `cwd` = an empty directory `data/agent-cwd/` → no CLAUDE.md picked up.
- `--tools` limits built-ins to WebSearch and WebFetch.
- **To verify in the agent issue:** after one run, confirm no plugin hook fired (e.g. no new memory-plugin session for that cwd). If one did, find the flag that disables it before shipping.

**Process model.** The server runs as a normal user process from a terminal, not a launchd daemon: Claude Code's subscription auth reads the login keychain.

## Data model (SQLite)

```sql
members  (id TEXT PK, name TEXT, code TEXT, color TEXT)          -- seeded from config

posts    (id INTEGER PK, origin TEXT CHECK (origin IN ('agent','member')),
          submitted_by TEXT NULL REFERENCES members,
          run_id INTEGER NULL REFERENCES runs,
          title, url, url_normalized TEXT UNIQUE, source, summary,
          why_it_matters, discussion_question,
          tags TEXT,                -- JSON array
          published_at TEXT NULL,   -- YYYY-MM-DD
          created_at TEXT)          -- UTC ISO

comments (id INTEGER PK, post_id REFERENCES posts, member_id REFERENCES members,
          body TEXT, created_at, updated_at)

marks    (post_id, member_id, created_at, PRIMARY KEY (post_id, member_id))
          -- "nothing to add"

runs     (id INTEGER PK, slot_date TEXT, slot_time TEXT,  -- London date, 'HH:MM'; NULL for manual
          trigger TEXT CHECK (trigger IN ('schedule','manual')),
          status TEXT CHECK (status IN ('running','success','failed')),
          error TEXT NULL, started_at, finished_at, post_id NULL)
-- at most one successful run per slot:
CREATE UNIQUE INDEX one_success_per_slot ON runs(slot_date, slot_time) WHERE status = 'success';
```

All timestamps stored in UTC. Slot times and "today" are computed in the configured timezone.

**URL normalisation** (before the unique check): force `https`, lowercase host, drop leading `www.`, drop fragment, drop `utm_*`, `ref`, `fbclid`, `gclid` params, sort remaining params, drop trailing slash.

## API

All routes read the current member from the cookie (except session routes).

| Method | Route | Purpose |
|---|---|---|
| GET | `/api/members` | Members list. |
| GET/POST | `/api/session` | Current member / pick member (sets cookie). |
| GET | `/api/feed?scope=recent\|archive&unread=1` | Posts grouped by day, with per-member caught-up and comment count. |
| GET | `/api/posts/:id` | Post, comments, marks, per-member caught-up. |
| POST | `/api/posts` | Member submission (validated by `PostInput`). |
| GET | `/api/submit-prompt` | Generated teammate prompt + JSON example. |
| POST | `/api/posts/:id/comments` | Add comment. |
| PATCH/DELETE | `/api/comments/:id` | Edit/delete own comment (403 otherwise). |
| PUT/DELETE | `/api/posts/:id/mark` | Set/undo "nothing to add". |
| GET | `/api/board` | 7-day posts × members matrix + per-member counts. |
| GET/POST | `/api/runs` | Recent runs / "Run now" (manual trigger). |

## Screens

1. **Pick your name**: 6 name buttons.
2. **Home**: progress strip (`Sojeong 25/25 · Ben 18/25 …`), then posts newest first grouped by day. Card: title, origin badge (`agent` / `found by Pablo`), 2-line summary, comment count, **member chips** (2-letter code in the member's colour, filled = caught up, outlined = not). Unread posts highlighted. "My unread" filter.
3. **Post page**: What it is, Why it matters for us, Discussion question, link, comments, comment box, "Read, nothing to add".
4. **Team board**: grid of last-7-day posts × members; ✓ commented, – nothing to add, ★ submitted it, blank not yet.
5. **Submit**: copy-ready prompt, JSON textarea, inline validation errors, preview, publish.
6. **Archive**: posts older than 7 days, same card layout.
7. **Agent runs**: today's slots with status (pending / running / success / failed + error), recent history, "Run now".

Two-letter codes, because initials collide (Sojeong/Seongkuk, Pablo/Pardeep): `SJ SK HE PA PD BE`.

## Error handling

| Failure | Behaviour |
|---|---|
| Agent: non-zero exit, timeout, `is_error`, invalid output, duplicate URL | Run marked `failed` with the error. Retried on a later tick, ≥5 min after the last attempt, max 3 attempts per slot. Visible on Agent runs page. Manual ("Run now") runs are not retried. |
| Server restarts mid-run | On startup, runs left `running` are marked `failed` ("interrupted") and become eligible for retry. |
| Two ticks overlap | In-process lock: only one agent run at a time. The partial unique index blocks a second success for the same slot. |
| Submit: invalid JSON or schema errors | Shown inline with field paths. Nothing saved. |
| Submit: duplicate URL | "Already posted" with a link to the existing post. |
| Edit/delete someone else's comment | 403. |

## Hosting

- `bun start` in a terminal on the laptop. The start script wraps the server in `caffeinate -i` so the Mac does not idle-sleep while it runs (closing the lid still sleeps; missed slots catch up on wake).
- Public URL: `ngrok http <port>`. ngrok is installed and configured on the laptop. **To verify in the hosting issue:** whether the account has a fixed domain (so the link stays the same across restarts) and whether visitors see an ngrok warning page first; document both in the README.
- SQLite file in `data/`, gitignored.

## Testing

- `bun test` on pure logic:
  - slot-due calculation with an injected clock (weekends, before/after slots, catch-up, DST change days);
  - recency window (48h, 72h on Mondays);
  - caught-up derivation (comment, mark, own post, deleted comment);
  - URL normalisation;
  - `PostInput` validation and prompt generation;
  - board/feed aggregation against an in-memory SQLite DB.
- Agent runner tested with a fake spawner and a recorded Claude JSON output fixture (success, `is_error`, invalid output).
- Scheduler tested with the fake agent: sequential catch-up, retry limits, interrupted runs.
- Manual smoke test: "Run now" produces a real post.

## Build order (UI-first)

1. Scaffold, config, DB schema, **seed script with ~20 realistic fake posts, comments and marks**.
2. Read API + screens against seed data: name picker, home, post page, board, archive.
3. Write paths: comments, marks, Submit page (schema + generated prompt).
4. Scheduler + Agent runs page, using the fake agent.
5. Real Claude agent runner, prompt, isolation check.
6. Hosting: start script, `caffeinate`, ngrok, README.

## Repo

- Public GitHub repo `hsk-kr/dev-sharing`.
- No employer or customer names in code or docs. Member first names live in `config/members.json`.
- `data/` gitignored.
