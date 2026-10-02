# Dev Sharing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A small team web app where an AI agent posts 5 curated AI/software-engineering items per weekday, members comment or mark "nothing to add", and a board shows who is caught up on the last 7 days.

**Architecture:** One Bun process: `Bun.serve` routes `/api/*` to a Hono app and everything else to a React SPA (Bun HTML imports, Tailwind via `bun-plugin-tailwind`). SQLite via `bun:sqlite`. A 60-second scheduler runs due slots sequentially by spawning headless Claude Code (`claude -p`) and stores the validated result. Pure logic (time, URLs, status) lives in `src/domain/` and is unit-tested; DB access lives in `src/db/`.

**Tech Stack:** Bun 1.3, TypeScript 7 (strict), Hono 4, React 19, React Router 8 (declarative mode), TanStack Query 5, Tailwind CSS 4, zod 4.

**Spec:** `docs/superpowers/specs/2026-10-02-dev-sharing-design.md`

## Global Constraints

- Runtime: Bun ≥ 1.3. No Vite, no Node-only tooling. Tests: `bun test`. Typecheck: `bun run typecheck` (must be clean at the end of every task).
- `tsconfig.json` must keep `"types": ["bun"]` and `"lib": ["ESNext", "DOM", "DOM.Iterable"]` (TypeScript 7 no longer auto-includes `@types/*`).
- `verbatimModuleSyntax` is on: type-only imports must use `import type`.
- Timezone `Europe/London` for slots, "today" and weekdays. Store every timestamp as UTC ISO (`date.toISOString()`).
- Slots exactly `10:00, 11:00, 14:00, 15:00, 16:00`, Monday–Friday. One agent post per slot.
- Catch-up window: rolling 7 days (`config.windowDays = 7`).
- Members (in this order): Sojeong `SJ`, Seongkuk `SK`, Henry `HE`, Pablo `PA`, Pardeep `PD`, Ben `BE`. Defined only in `config/members.json`.
- Agent: `claude -p --model claude-opus-5-5 --tools "WebSearch,WebFetch" --allowedTools "WebSearch,WebFetch" --output-format json --json-schema <schema> --no-session-persistence --setting-sources "" --strict-mcp-config`, prompt on stdin, cwd `data/agent-cwd`. **Never** `--bare` (forces API-key auth). The JSON schema passed to `--json-schema` must not contain a `$schema` key (the CLI rejects the draft 2020-12 meta-schema ref).
- Databases: real `data/dev-sharing.db`, development `data/dev.db`. `data/` is gitignored. The seed script refuses to touch the real DB.
- No employer, customer or company names anywhere in code, docs or commit messages.
- Each task ends with a commit whose message ends with `Closes #<issue>` for that task's GitHub issue.

## File Structure

```
config/members.json          members (name, 2-letter code, colour)
src/types.ts                 shared types (server + web)
src/config.ts                runtime config (env + constants + members)
src/domain/url.ts            normalizeUrl
src/domain/time.ts           localNow, dueSlots, recencyHours, windowStart
src/domain/status.ts         cellStatus
src/schema/post.ts           PostInput zod schema, JSON Schema, teammate prompt, parsePostJson
src/db/db.ts                 openDb: schema + member seeding
src/db/posts.ts              insertPost, getPost, listPosts, findPostIdByUrl, recentPostRefs
src/db/engagement.ts         comments + marks queries, engagementFor
src/db/runs.ts               runs queries, slotHistory
src/feed.ts                  getFeed, getBoard, getPostDetail
src/runs-view.ts             getRunsView
src/scheduler.ts             createScheduler (tick, runNow, start)
src/agent/types.ts           Agent, AgentContext
src/agent/fake.ts            fakeAgent (dev)
src/agent/prompt.ts          buildAgentPrompt
src/agent/claude.ts          createClaudeAgent, bunSpawner, parseClaudeResult
src/api/common.ts            Env, cookie name, memberGuard, readJson, ApiDeps
src/api/session.ts           /members, /session
src/api/posts.ts             /feed, /posts, /board, /submit-prompt, submit
src/api/engagement.ts        comments + marks routes
src/api/runs.ts              /runs
src/api/app.ts               createApi
src/server.ts                Bun.serve entry
scripts/seed.ts              dev seed data
scripts/agent-smoke.ts       one real agent call, printed
web/index.html, web/styles.css, web/main.tsx, web/App.tsx
web/api.ts                   fetch client + query hooks
web/me.ts                    current-member context
web/format.ts                dayLabel, timeLabel, groupByDay
web/status.ts                statusLabel, cellSymbol
web/components/*.tsx         MemberChip, OriginBadge, PostCard, PostList, ProgressStrip, Header, StatusRow, Comments, MarkButton
web/pages/*.tsx              PickName, Home, PostPage, BoardPage, Archive, Submit, Runs
tests/*.test.ts              unit + API tests
```

---

### Task 1: Scaffold, config and database

**Files:**
- Create: `package.json`, `tsconfig.json`, `.gitignore`, `LICENSE`
- Create: `config/members.json`, `src/types.ts`, `src/config.ts`, `src/db/db.ts`
- Test: `tests/helpers.ts`, `tests/config.test.ts`, `tests/db.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `config: Config` from `src/config.ts` with fields `timeZone: string`, `slots: string[]`, `port: number`, `dbPath: string`, `agent: "claude" | "fake"`, `schedulerEnabled: boolean`, `members: Member[]`, `windowDays: number`, `maxAttemptsPerSlot: number`, `retryAfterMs: number`, `agentTimeoutMs: number`.
  - `openDb(path: string, members: Member[]): Database` from `src/db/db.ts`.
  - All types in `src/types.ts` (below).
  - `testDb(): Database` and `members: Member[]` from `tests/helpers.ts`.

- [ ] **Step 1: Create project files**

`package.json`:

```json
{
  "name": "dev-sharing",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "DB_PATH=data/dev.db AGENT=fake bun --watch src/server.ts",
    "start": "NODE_ENV=production caffeinate -i bun src/server.ts",
    "seed": "DB_PATH=data/dev.db bun scripts/seed.ts",
    "test": "bun test",
    "typecheck": "tsc --noEmit -p ."
  }
}
```

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "lib": ["ESNext", "DOM", "DOM.Iterable"],
    "types": ["bun"],
    "target": "ESNext",
    "module": "Preserve",
    "moduleDetection": "force",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "resolveJsonModule": true,
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,
    "strict": true,
    "skipLibCheck": true,
    "noUncheckedIndexedAccess": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitOverride": true
  },
  "include": ["src", "web", "tests", "scripts"]
}
```

`.gitignore`:

```
node_modules/
data/
.DS_Store
```

`LICENSE`: the standard MIT license text, `Copyright (c) 2026 hsk-kr`.

- [ ] **Step 2: Install dependencies**

Run:

```bash
bun add hono react react-dom react-router @tanstack/react-query zod tailwindcss bun-plugin-tailwind
bun add -d @types/bun @types/react @types/react-dom typescript
```

Expected: `bun.lock` created; `package.json` gains `dependencies` and `devDependencies`.

- [ ] **Step 3: Write the failing tests**

`tests/helpers.ts`:

```ts
import { config } from "../src/config";
import { openDb } from "../src/db/db";

export const members = config.members;
export const testDb = () => openDb(":memory:", members);
```

`tests/config.test.ts`:

```ts
import { expect, test } from "bun:test";
import { config } from "../src/config";

test("config lists the six members in order with unique 2-letter codes", () => {
  expect(config.members.map((m) => m.name)).toEqual(["Sojeong", "Seongkuk", "Henry", "Pablo", "Pardeep", "Ben"]);
  expect(config.members.map((m) => m.code)).toEqual(["SJ", "SK", "HE", "PA", "PD", "BE"]);
  for (const m of config.members) expect(m.color).toMatch(/^#[0-9a-f]{6}$/);
});

test("config has the agreed schedule", () => {
  expect(config.timeZone).toBe("Europe/London");
  expect(config.slots).toEqual(["10:00", "11:00", "14:00", "15:00", "16:00"]);
  expect(config.windowDays).toBe(7);
});
```

`tests/db.test.ts`:

```ts
import { expect, test } from "bun:test";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { openDb } from "../src/db/db";
import { members, testDb } from "./helpers";

test("openDb creates all tables and seeds members", () => {
  const db = testDb();
  const tables = db
    .query<{ name: string }, []>("SELECT name FROM sqlite_master WHERE type = 'table' ORDER BY name")
    .all()
    .map((r) => r.name);
  expect(tables).toEqual(["comments", "marks", "members", "posts", "runs"]);
  const rows = db.query<{ id: string }, []>("SELECT id FROM members ORDER BY rowid").all();
  expect(rows.map((r) => r.id)).toEqual(members.map((m) => m.id));
});

test("openDb is idempotent and updates changed member details", () => {
  const path = join(tmpdir(), `dev-sharing-${crypto.randomUUID()}.db`);
  openDb(path, members).close();
  const renamed = members.map((m) => (m.id === "ben" ? { ...m, name: "Benjamin" } : m));
  const db = openDb(path, renamed);
  const ben = db.query<{ name: string }, { id: string }>("SELECT name FROM members WHERE id = $id").get({ id: "ben" });
  expect(ben?.name).toBe("Benjamin");
  expect(db.query<{ n: number }, []>("SELECT COUNT(*) AS n FROM members").get()?.n).toBe(6);
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `bun test`
Expected: FAIL, cannot resolve `../src/config` / `../src/db/db`.

- [ ] **Step 5: Implement**

`config/members.json`:

```json
[
  { "id": "sojeong", "name": "Sojeong", "code": "SJ", "color": "#e11d48" },
  { "id": "seongkuk", "name": "Seongkuk", "code": "SK", "color": "#2563eb" },
  { "id": "henry", "name": "Henry", "code": "HE", "color": "#16a34a" },
  { "id": "pablo", "name": "Pablo", "code": "PA", "color": "#d97706" },
  { "id": "pardeep", "name": "Pardeep", "code": "PD", "color": "#7c3aed" },
  { "id": "ben", "name": "Ben", "code": "BE", "color": "#0891b2" }
]
```

`src/types.ts`:

```ts
export type Member = { id: string; name: string; code: string; color: string };

export type Origin = "agent" | "member";

export type Post = {
  id: number;
  origin: Origin;
  submitted_by: string | null;
  run_id: number | null;
  title: string;
  url: string;
  source: string;
  summary: string;
  why_it_matters: string;
  discussion_question: string;
  tags: string[];
  published_at: string | null;
  created_at: string;
};

/** One member's state on one post. null = not caught up yet. */
export type CellStatus = "author" | "commented" | "nothing" | null;

export type FeedPost = Post & { comment_count: number; status: Record<string, CellStatus> };

export type Comment = {
  id: number;
  post_id: number;
  member_id: string;
  body: string;
  created_at: string;
  updated_at: string;
};

export type PostDetail = FeedPost & { comments: Comment[] };

export type Board = {
  members: Member[];
  posts: FeedPost[];
  counts: Record<string, { done: number; total: number }>;
};

export type RunStatus = "running" | "success" | "failed";

export type Run = {
  id: number;
  slot_date: string | null;
  slot_time: string | null;
  trigger: "schedule" | "manual";
  status: RunStatus;
  error: string | null;
  started_at: string;
  finished_at: string | null;
  post_id: number | null;
};

export type SlotView = {
  time: string;
  state: "pending" | "running" | "success" | "failed" | "skipped";
  attempts: number;
  post_id: number | null;
  error: string | null;
};

export type RunsView = { today: string; isWorkday: boolean; slots: SlotView[]; recent: Run[] };
```

`src/config.ts`:

```ts
import members from "../config/members.json";
import type { Member } from "./types";

export const config = {
  timeZone: process.env.APP_TIME_ZONE ?? "Europe/London",
  slots: ["10:00", "11:00", "14:00", "15:00", "16:00"],
  port: Number(process.env.PORT ?? 3000),
  dbPath: process.env.DB_PATH ?? "data/dev-sharing.db",
  agent: (process.env.AGENT === "fake" ? "fake" : "claude") as "claude" | "fake",
  schedulerEnabled: process.env.SCHEDULER !== "off",
  members: members as Member[],
  windowDays: 7,
  maxAttemptsPerSlot: 3,
  retryAfterMs: 5 * 60_000,
  agentTimeoutMs: 10 * 60_000,
};

export type Config = typeof config;
```

`src/db/db.ts`:

```ts
import { Database } from "bun:sqlite";
import { mkdirSync } from "node:fs";
import { dirname } from "node:path";
import type { Member } from "../types";

const SCHEMA = `
PRAGMA foreign_keys = ON;

CREATE TABLE IF NOT EXISTS members (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  code TEXT NOT NULL,
  color TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS runs (
  id INTEGER PRIMARY KEY,
  slot_date TEXT,
  slot_time TEXT,
  trigger TEXT NOT NULL CHECK (trigger IN ('schedule', 'manual')),
  status TEXT NOT NULL CHECK (status IN ('running', 'success', 'failed')),
  error TEXT,
  started_at TEXT NOT NULL,
  finished_at TEXT,
  post_id INTEGER REFERENCES posts(id) ON DELETE SET NULL
);
CREATE UNIQUE INDEX IF NOT EXISTS one_success_per_slot
  ON runs(slot_date, slot_time) WHERE status = 'success';

CREATE TABLE IF NOT EXISTS posts (
  id INTEGER PRIMARY KEY,
  origin TEXT NOT NULL CHECK (origin IN ('agent', 'member')),
  submitted_by TEXT REFERENCES members(id),
  run_id INTEGER REFERENCES runs(id),
  title TEXT NOT NULL,
  url TEXT NOT NULL,
  url_normalized TEXT NOT NULL UNIQUE,
  source TEXT NOT NULL,
  summary TEXT NOT NULL,
  why_it_matters TEXT NOT NULL,
  discussion_question TEXT NOT NULL,
  tags TEXT NOT NULL DEFAULT '[]',
  published_at TEXT,
  created_at TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS posts_created_at ON posts(created_at);

CREATE TABLE IF NOT EXISTS comments (
  id INTEGER PRIMARY KEY,
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  member_id TEXT NOT NULL REFERENCES members(id),
  body TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS comments_post ON comments(post_id);

CREATE TABLE IF NOT EXISTS marks (
  post_id INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  member_id TEXT NOT NULL REFERENCES members(id),
  created_at TEXT NOT NULL,
  PRIMARY KEY (post_id, member_id)
);
`;

/** Open (or create) the database, apply the schema and upsert members. */
export function openDb(path: string, members: Member[]): Database {
  if (path !== ":memory:") mkdirSync(dirname(path), { recursive: true });
  const db = new Database(path, { create: true, strict: true });
  db.run("PRAGMA journal_mode = WAL;");
  db.run(SCHEMA);
  const upsert = db.query(
    `INSERT INTO members (id, name, code, color) VALUES ($id, $name, $code, $color)
     ON CONFLICT(id) DO UPDATE SET name = excluded.name, code = excluded.code, color = excluded.color`,
  );
  db.transaction(() => {
    for (const m of members) upsert.run(m);
  })();
  return db;
}
```

- [ ] **Step 6: Run tests and typecheck**

Run: `bun test && bun run typecheck`
Expected: 4 tests pass; typecheck prints nothing and exits 0.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: scaffold project, config and SQLite schema

Closes #<issue>"
```

---

### Task 2: Domain logic (time, URLs, status)

**Files:**
- Create: `src/domain/time.ts`, `src/domain/url.ts`, `src/domain/status.ts`
- Test: `tests/time.test.ts`, `tests/url.test.ts`, `tests/status.test.ts`

**Interfaces:**
- Consumes: `CellStatus` from `src/types.ts`.
- Produces:
  - `localNow(now: Date, timeZone: string): { date: string; time: string; weekday: number }` (weekday 1 = Monday … 7 = Sunday)
  - `isWorkday(l: LocalNow): boolean`
  - `dueSlots(now: Date, timeZone: string, slots: string[]): { date: string; time: string }[]`
  - `recencyHours(now: Date, timeZone: string): 48 | 72`
  - `windowStart(now: Date, days: number): string` (UTC ISO)
  - `normalizeUrl(raw: string): string` (throws on invalid URL)
  - `cellStatus(f: { isAuthor: boolean; hasComment: boolean; hasMark: boolean }): CellStatus`

- [ ] **Step 1: Write the failing tests**

`tests/time.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import { dueSlots, localNow, recencyHours, windowStart } from "../src/domain/time";

const TZ = "Europe/London";
const SLOTS = ["10:00", "11:00", "14:00", "15:00", "16:00"];
const at = (iso: string) => new Date(iso);

describe("localNow", () => {
  test("converts UTC to London summer time", () => {
    expect(localNow(at("2026-09-30T09:05:00Z"), TZ)).toEqual({ date: "2026-09-30", time: "10:05", weekday: 3 });
  });
  test("rolls the date over at local midnight", () => {
    expect(localNow(at("2026-09-30T23:30:00Z"), TZ)).toEqual({ date: "2026-10-01", time: "00:30", weekday: 4 });
  });
  test("uses GMT after the clocks go back", () => {
    expect(localNow(at("2026-10-26T10:00:00Z"), TZ)).toEqual({ date: "2026-10-26", time: "10:00", weekday: 1 });
  });
});

describe("dueSlots", () => {
  test("nothing is due before 10:00", () => {
    expect(dueSlots(at("2026-09-30T08:59:00Z"), TZ, SLOTS)).toEqual([]);
  });
  test("10:00 is due at 10:00 local time", () => {
    expect(dueSlots(at("2026-09-30T09:00:00Z"), TZ, SLOTS)).toEqual([{ date: "2026-09-30", time: "10:00" }]);
  });
  test("every past slot is due, for catch-up after sleep", () => {
    expect(dueSlots(at("2026-09-30T13:30:00Z"), TZ, SLOTS).map((s) => s.time)).toEqual(["10:00", "11:00", "14:00"]);
  });
  test("nothing is due on weekends", () => {
    expect(dueSlots(at("2026-10-03T15:00:00Z"), TZ, SLOTS)).toEqual([]);
  });
  test("slot times follow local time across the DST change", () => {
    expect(dueSlots(at("2026-10-26T09:30:00Z"), TZ, SLOTS)).toEqual([]);
    expect(dueSlots(at("2026-10-26T10:00:00Z"), TZ, SLOTS).map((s) => s.time)).toEqual(["10:00"]);
  });
});

describe("recencyHours", () => {
  test("72 hours on Monday to cover the weekend", () => {
    expect(recencyHours(at("2026-09-28T09:00:00Z"), TZ)).toBe(72);
  });
  test("48 hours on other weekdays", () => {
    expect(recencyHours(at("2026-09-29T09:00:00Z"), TZ)).toBe(48);
  });
});

test("windowStart goes back N days in UTC", () => {
  expect(windowStart(at("2026-10-08T12:00:00Z"), 7)).toBe("2026-10-01T12:00:00.000Z");
});
```

`tests/url.test.ts`:

```ts
import { expect, test } from "bun:test";
import { normalizeUrl } from "../src/domain/url";

test("forces https, lowercases host, drops www, fragment, tracking params and trailing slash", () => {
  expect(normalizeUrl("http://www.Example.com/a/B/?utm_source=x&b=2&a=1#frag")).toBe("https://example.com/a/B?a=1&b=2");
});

test("root URL has no trailing slash", () => {
  expect(normalizeUrl("https://example.com/")).toBe("https://example.com");
});

test("keeps meaningful query params", () => {
  expect(normalizeUrl("https://youtube.com/watch?v=abc")).toBe("https://youtube.com/watch?v=abc");
});

test("drops fbclid, gclid and ref", () => {
  expect(normalizeUrl("https://example.com/x?fbclid=1&gclid=2&ref=hn&id=7")).toBe("https://example.com/x?id=7");
});

test("keeps non-default ports", () => {
  expect(normalizeUrl("https://example.com:8443/x")).toBe("https://example.com:8443/x");
});

test("throws on an invalid URL", () => {
  expect(() => normalizeUrl("not a url")).toThrow();
});
```

`tests/status.test.ts`:

```ts
import { expect, test } from "bun:test";
import { cellStatus } from "../src/domain/status";

test("author wins over everything", () => {
  expect(cellStatus({ isAuthor: true, hasComment: true, hasMark: true })).toBe("author");
});
test("a comment wins over a mark", () => {
  expect(cellStatus({ isAuthor: false, hasComment: true, hasMark: true })).toBe("commented");
});
test("a mark alone means nothing to add", () => {
  expect(cellStatus({ isAuthor: false, hasComment: false, hasMark: true })).toBe("nothing");
});
test("nothing means not caught up", () => {
  expect(cellStatus({ isAuthor: false, hasComment: false, hasMark: false })).toBeNull();
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `bun test tests/time.test.ts tests/url.test.ts tests/status.test.ts`
Expected: FAIL, modules not found.

- [ ] **Step 3: Implement**

`src/domain/time.ts`:

```ts
export type LocalNow = { date: string; time: string; weekday: number };

const WEEKDAYS = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"];

/** Wall-clock date, time and ISO weekday (1 = Monday) of `now` in `timeZone`. */
export function localNow(now: Date, timeZone: string): LocalNow {
  const parts = new Intl.DateTimeFormat("en-GB", {
    timeZone,
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    hourCycle: "h23",
    weekday: "short",
  }).formatToParts(now);
  const get = (type: Intl.DateTimeFormatPartTypes) => parts.find((p) => p.type === type)?.value ?? "";
  return {
    date: `${get("year")}-${get("month")}-${get("day")}`,
    time: `${get("hour")}:${get("minute")}`,
    weekday: WEEKDAYS.indexOf(get("weekday")) + 1,
  };
}

export const isWorkday = (l: LocalNow) => l.weekday >= 1 && l.weekday <= 5;

/** Today's slots whose time has come, in order. Empty on weekends. */
export function dueSlots(now: Date, timeZone: string, slots: string[]): { date: string; time: string }[] {
  const l = localNow(now, timeZone);
  if (!isWorkday(l)) return [];
  return slots.filter((s) => s <= l.time).map((time) => ({ date: l.date, time }));
}

/** How far back the agent may look: 72h on Mondays (covers the weekend), otherwise 48h. */
export const recencyHours = (now: Date, timeZone: string): 48 | 72 =>
  localNow(now, timeZone).weekday === 1 ? 72 : 48;

export const windowStart = (now: Date, days: number) => new Date(now.getTime() - days * 86_400_000).toISOString();
```

`src/domain/url.ts`:

```ts
const TRACKING_PARAM = /^(utm_.+|ref|fbclid|gclid)$/i;

/** Canonical form of a URL, used to detect the same story posted twice. */
export function normalizeUrl(raw: string): string {
  const u = new URL(raw.trim());
  const host = u.hostname.toLowerCase().replace(/^www\./, "");
  const params = [...u.searchParams]
    .filter(([key]) => !TRACKING_PARAM.test(key))
    .sort(([a, av], [b, bv]) => (a === b ? av.localeCompare(bv) : a.localeCompare(b)));
  const query = new URLSearchParams(params).toString();
  const path = u.pathname.replace(/\/+$/, "");
  return `https://${host}${u.port ? `:${u.port}` : ""}${path}${query ? `?${query}` : ""}`;
}
```

`src/domain/status.ts`:

```ts
import type { CellStatus } from "../types";

export function cellStatus(f: { isAuthor: boolean; hasComment: boolean; hasMark: boolean }): CellStatus {
  if (f.isAuthor) return "author";
  if (f.hasComment) return "commented";
  if (f.hasMark) return "nothing";
  return null;
}
```

- [ ] **Step 4: Run tests and typecheck**

Run: `bun test && bun run typecheck`
Expected: all pass, typecheck clean.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add time, URL and status domain logic

Closes #<issue>"
```

---

### Task 3: Post schema, JSON Schema and teammate prompt

**Files:**
- Create: `src/schema/post.ts`
- Test: `tests/schema.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `PostInput` (zod schema) and `type PostInput`
  - `postJsonSchema(): Record<string, unknown>` (no `$schema` key)
  - `EXAMPLE_POST: PostInput`
  - `teammatePrompt(): string`
  - `parsePostJson(text: string): { ok: true; value: PostInput } | { ok: false; error: string }`

- [ ] **Step 1: Write the failing tests**

`tests/schema.test.ts`:

```ts
import { expect, test } from "bun:test";
import { EXAMPLE_POST, PostInput, parsePostJson, postJsonSchema, teammatePrompt } from "../src/schema/post";

const FIELDS = ["title", "url", "source", "summary", "why_it_matters", "discussion_question", "tags", "published_at"];

test("EXAMPLE_POST is valid", () => {
  expect(PostInput.parse(EXAMPLE_POST)).toEqual(EXAMPLE_POST);
});

test("rejects non-http URLs, short titles, more than 3 tags and bad dates", () => {
  const bad = PostInput.safeParse({ ...EXAMPLE_POST, url: "ftp://x.y/z", title: "Hi", tags: ["a", "b", "c", "d"], published_at: "yesterday" });
  expect(bad.success).toBe(false);
  const paths = bad.error!.issues.map((i) => i.path.join("."));
  expect(paths).toEqual(expect.arrayContaining(["url", "title", "tags", "published_at"]));
});

test("postJsonSchema has every field required and no $schema key", () => {
  const schema = postJsonSchema();
  expect(schema.$schema).toBeUndefined();
  expect(schema.required).toEqual(FIELDS);
  expect(Object.keys(schema.properties as object)).toEqual(FIELDS);
});

test("teammatePrompt describes every field and embeds a valid example", () => {
  const prompt = teammatePrompt();
  for (const f of FIELDS) expect(prompt).toContain(`- ${f}:`);
  const json = prompt.slice(prompt.indexOf("{"), prompt.lastIndexOf("}") + 1);
  expect(PostInput.parse(JSON.parse(json))).toEqual(EXAMPLE_POST);
});

test("parsePostJson accepts JSON wrapped in a markdown code fence", () => {
  const r = parsePostJson("```json\n" + JSON.stringify(EXAMPLE_POST) + "\n```");
  expect(r).toEqual({ ok: true, value: EXAMPLE_POST });
});

test("parsePostJson explains invalid JSON and schema errors", () => {
  expect(parsePostJson("{nope")).toEqual({ ok: false, error: "Not valid JSON. Paste only the JSON object." });
  const r = parsePostJson(JSON.stringify({ ...EXAMPLE_POST, url: "nope" }));
  expect(r.ok).toBe(false);
  if (!r.ok) expect(r.error).toContain("url");
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `bun test tests/schema.test.ts`
Expected: FAIL, module not found.

- [ ] **Step 3: Implement**

`src/schema/post.ts`:

```ts
import { z } from "zod";

export const PostInput = z.object({
  title: z.string().trim().min(5).max(160).describe("Short, specific headline."),
  url: z.url({ protocol: /^https?$/ }).describe("Link to the primary source, not a roundup or newsletter."),
  source: z.string().trim().min(2).max(80).describe("Site or organisation name, e.g. anthropic.com."),
  summary: z.string().trim().min(20).max(600).describe("What it is, in 2-3 plain sentences."),
  why_it_matters: z
    .string()
    .trim()
    .min(20)
    .max(500)
    .describe("Why software engineers at a startup should care, in 1-3 sentences."),
  discussion_question: z
    .string()
    .trim()
    .min(10)
    .max(300)
    .describe("One open question that invites each teammate to share their own take."),
  tags: z.array(z.string().trim().min(1).max(24)).max(3).describe("Up to 3 short lowercase tags."),
  published_at: z.iso.date().nullable().describe("Publish date of the source as YYYY-MM-DD, or null if unknown."),
});

export type PostInput = z.infer<typeof PostInput>;

/** JSON Schema for `claude --json-schema`. The CLI rejects the 2020-12 `$schema` ref, so it is removed. */
export function postJsonSchema(): Record<string, unknown> {
  const { $schema: _ignored, ...schema } = z.toJSONSchema(PostInput) as Record<string, unknown>;
  return schema;
}

export const EXAMPLE_POST: PostInput = {
  title: "Example: open-source coding agent adds parallel sub-agents",
  url: "https://example.com/blog/parallel-sub-agents",
  source: "example.com",
  summary:
    "The agent can now split a task into independent pieces and run them in parallel sub-agents, then merge the results. It works from the terminal and in CI.",
  why_it_matters:
    "Large refactors and test fixes could finish in a fraction of the time, which changes how we plan bigger tickets.",
  discussion_question: "Which of our current tickets would you hand to parallel agents first, and why?",
  tags: ["agents", "tooling"],
  published_at: "2026-10-01",
};

/** Copy-ready prompt teammates paste into their own AI, generated from the schema. */
export function teammatePrompt(): string {
  const properties = postJsonSchema().properties as Record<string, { description?: string }>;
  const fields = Object.entries(properties)
    .map(([key, prop]) => `- ${key}: ${prop.description ?? ""}`)
    .join("\n");
  return [
    "I want to share an article with my team on our Dev Sharing board.",
    "Read the link at the bottom and return ONLY a JSON object (no markdown, no commentary) with exactly these fields:",
    "",
    fields,
    "",
    "Example of the format:",
    JSON.stringify(EXAMPLE_POST, null, 2),
    "",
    "Link: <paste the URL here>",
  ].join("\n");
}

const stripCodeFence = (text: string) =>
  text
    .trim()
    .replace(/^```(?:json)?\s*/i, "")
    .replace(/\s*```$/, "");

export function parsePostJson(text: string): { ok: true; value: PostInput } | { ok: false; error: string } {
  let raw: unknown;
  try {
    raw = JSON.parse(stripCodeFence(text));
  } catch {
    return { ok: false, error: "Not valid JSON. Paste only the JSON object." };
  }
  const result = PostInput.safeParse(raw);
  return result.success ? { ok: true, value: result.data } : { ok: false, error: z.prettifyError(result.error) };
}
```

- [ ] **Step 4: Run tests and typecheck**

Run: `bun test && bun run typecheck`
Expected: all pass, typecheck clean.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add post schema, JSON Schema export and teammate prompt

Closes #<issue>"
```

---

### Task 4: Posts, comments and marks storage + feed/board queries

**Files:**
- Create: `src/db/posts.ts`, `src/db/engagement.ts`, `src/feed.ts`
- Modify: `tests/helpers.ts` (add `samplePost`)
- Test: `tests/feed.test.ts`

**Interfaces:**
- Consumes: `openDb`, `normalizeUrl`, `cellStatus`, `windowStart`, `PostInput`, types.
- Produces:
  - `src/db/posts.ts`: `class DuplicateUrlError extends Error { existingId: number }`; `insertPost(db, input: PostInput, meta: { origin: Origin; submittedBy: string | null; runId: number | null; now: Date }): Post`; `getPost(db, id: number): Post | null`; `listPosts(db, range: { since?: string; before?: string }): Post[]` (newest first); `findPostIdByUrl(db, url: string): number | null`; `recentPostRefs(db, since: string): { title: string; url: string }[]`
  - `src/db/engagement.ts`: `addComment(db, postId, memberId, body, now): Comment`; `getComment(db, id): Comment | null`; `updateComment(db, id, body, now): Comment`; `deleteComment(db, id): void`; `listComments(db, postId): Comment[]` (oldest first); `setMark(db, postId, memberId, now): void`; `clearMark(db, postId, memberId): void`; `engagementFor(db, postIds: number[]): Map<number, Engagement>`
  - `src/feed.ts`: `getFeed(db, members, scope: "recent" | "archive", now: Date, windowDays: number): FeedPost[]`; `getBoard(db, members, now, windowDays): Board`; `getPostDetail(db, members, id): PostDetail | null`
  - `tests/helpers.ts`: `samplePost(overrides?: Partial<PostInput>): PostInput` (unique URL per call)

- [ ] **Step 1: Add `samplePost` to the test helpers**

Replace `tests/helpers.ts` with:

```ts
import { config } from "../src/config";
import { openDb } from "../src/db/db";
import type { PostInput } from "../src/schema/post";

export const members = config.members;
export const testDb = () => openDb(":memory:", members);

let counter = 0;
export function samplePost(overrides: Partial<PostInput> = {}): PostInput {
  counter += 1;
  return {
    title: `Sample post number ${counter}`,
    url: `https://example.com/post-${counter}`,
    source: "example.com",
    summary: "A sample summary that is long enough to pass validation.",
    why_it_matters: "It matters because the tests need realistic data.",
    discussion_question: "Would you use this at work?",
    tags: ["test"],
    published_at: "2026-10-01",
    ...overrides,
  };
}
```

- [ ] **Step 2: Write the failing tests**

`tests/feed.test.ts`:

```ts
import { describe, expect, test } from "bun:test";
import {
  addComment,
  clearMark,
  deleteComment,
  engagementFor,
  getComment,
  listComments,
  setMark,
  updateComment,
} from "../src/db/engagement";
import { DuplicateUrlError, findPostIdByUrl, getPost, insertPost, recentPostRefs } from "../src/db/posts";
import { getBoard, getFeed, getPostDetail } from "../src/feed";
import { members, samplePost, testDb } from "./helpers";

const NOW = new Date("2026-10-02T12:00:00Z");
const hoursAgo = (h: number) => new Date(NOW.getTime() - h * 3_600_000);
const agent = (at: Date) => ({ origin: "agent" as const, submittedBy: null, runId: null, now: at });
const byMember = (id: string, at: Date) => ({ origin: "member" as const, submittedBy: id, runId: null, now: at });

describe("posts", () => {
  test("insertPost stores the post and returns tags as an array", () => {
    const db = testDb();
    const p = insertPost(db, samplePost({ tags: ["a", "b"] }), agent(NOW));
    expect(p.id).toBeGreaterThan(0);
    expect(p.tags).toEqual(["a", "b"]);
    expect(p.created_at).toBe(NOW.toISOString());
    expect(getPost(db, p.id)).toEqual(p);
    expect(getPost(db, 999)).toBeNull();
  });

  test("insertPost rejects a URL that normalises to an existing post", () => {
    const db = testDb();
    const first = insertPost(db, samplePost({ url: "https://example.com/a" }), agent(NOW));
    let error: unknown;
    try {
      insertPost(db, samplePost({ url: "http://www.example.com/a/?utm_source=x" }), agent(NOW));
    } catch (e) {
      error = e;
    }
    expect(error).toBeInstanceOf(DuplicateUrlError);
    expect((error as DuplicateUrlError).existingId).toBe(first.id);
    expect(findPostIdByUrl(db, "https://example.com/a#top")).toBe(first.id);
    expect(findPostIdByUrl(db, "https://example.com/b")).toBeNull();
  });

  test("recentPostRefs lists title and url since a date, newest first", () => {
    const db = testDb();
    insertPost(db, samplePost({ title: "Old one here" }), agent(hoursAgo(24 * 40)));
    insertPost(db, samplePost({ title: "Older here" }), agent(hoursAgo(5)));
    insertPost(db, samplePost({ title: "Newest here" }), agent(hoursAgo(1)));
    const refs = recentPostRefs(db, hoursAgo(24 * 30).toISOString());
    expect(refs.map((r) => r.title)).toEqual(["Newest here", "Older here"]);
    expect(refs[0]!.url).toStartWith("https://example.com/");
  });
});

describe("caught-up status", () => {
  test("status per member is author / commented / nothing / null", () => {
    const db = testDb();
    const p = insertPost(db, samplePost(), byMember("pablo", hoursAgo(1)));
    addComment(db, p.id, "sojeong", "Nice", NOW);
    setMark(db, p.id, "ben", NOW);
    const [fp] = getFeed(db, members, "recent", NOW, 7);
    expect(fp!.status).toEqual({
      sojeong: "commented",
      seongkuk: null,
      henry: null,
      pablo: "author",
      pardeep: null,
      ben: "nothing",
    });
    expect(fp!.comment_count).toBe(1);
  });

  test("deleting your only comment makes the post unread again", () => {
    const db = testDb();
    const p = insertPost(db, samplePost(), agent(hoursAgo(1)));
    const c = addComment(db, p.id, "henry", "Hmm", NOW);
    deleteComment(db, c.id);
    expect(getFeed(db, members, "recent", NOW, 7)[0]!.status.henry).toBeNull();
  });

  test("a mark keeps you caught up after deleting your comment; clearMark undoes it", () => {
    const db = testDb();
    const p = insertPost(db, samplePost(), agent(hoursAgo(1)));
    const c = addComment(db, p.id, "henry", "Hmm", NOW);
    setMark(db, p.id, "henry", NOW);
    setMark(db, p.id, "henry", NOW); // idempotent
    deleteComment(db, c.id);
    expect(getFeed(db, members, "recent", NOW, 7)[0]!.status.henry).toBe("nothing");
    clearMark(db, p.id, "henry");
    expect(getFeed(db, members, "recent", NOW, 7)[0]!.status.henry).toBeNull();
  });

  test("engagementFor with no ids returns an empty map", () => {
    expect(engagementFor(testDb(), []).size).toBe(0);
  });
});

describe("window", () => {
  test("recent has the last 7 days, archive the rest, both newest first", () => {
    const db = testDb();
    insertPost(db, samplePost({ title: "Eight days ago" }), agent(hoursAgo(24 * 8)));
    insertPost(db, samplePost({ title: "Six days ago" }), agent(hoursAgo(24 * 6)));
    insertPost(db, samplePost({ title: "One hour ago" }), agent(hoursAgo(1)));
    expect(getFeed(db, members, "recent", NOW, 7).map((p) => p.title)).toEqual(["One hour ago", "Six days ago"]);
    expect(getFeed(db, members, "archive", NOW, 7).map((p) => p.title)).toEqual(["Eight days ago"]);
  });
});

describe("board", () => {
  test("counts caught-up posts per member over the last 7 days", () => {
    const db = testDb();
    const a = insertPost(db, samplePost(), agent(hoursAgo(1)));
    const b = insertPost(db, samplePost(), byMember("ben", hoursAgo(2)));
    insertPost(db, samplePost(), agent(hoursAgo(24 * 9))); // archived, not counted
    addComment(db, a.id, "sojeong", "Yes", NOW);
    setMark(db, b.id, "sojeong", NOW);
    const board = getBoard(db, members, NOW, 7);
    expect(board.posts).toHaveLength(2);
    expect(board.counts.sojeong).toEqual({ done: 2, total: 2 });
    expect(board.counts.ben).toEqual({ done: 1, total: 2 });
    expect(board.counts.henry).toEqual({ done: 0, total: 2 });
  });
});

describe("comments", () => {
  test("update changes body and updated_at; list is oldest first; detail includes comments", () => {
    const db = testDb();
    const p = insertPost(db, samplePost(), agent(hoursAgo(3)));
    const first = addComment(db, p.id, "sojeong", "First", hoursAgo(2));
    addComment(db, p.id, "ben", "Second", hoursAgo(1));
    const edited = updateComment(db, first.id, "First, edited", NOW);
    expect(edited.body).toBe("First, edited");
    expect(edited.updated_at).toBe(NOW.toISOString());
    expect(edited.created_at).toBe(first.created_at);
    expect(getComment(db, first.id)).toEqual(edited);
    expect(listComments(db, p.id).map((c) => c.body)).toEqual(["First, edited", "Second"]);
    const detail = getPostDetail(db, members, p.id)!;
    expect(detail.comments).toHaveLength(2);
    expect(detail.status.ben).toBe("commented");
    expect(getPostDetail(db, members, 999)).toBeNull();
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `bun test tests/feed.test.ts`
Expected: FAIL, modules not found.

- [ ] **Step 4: Implement**

`src/db/posts.ts`:

```ts
import type { Database } from "bun:sqlite";
import { normalizeUrl } from "../domain/url";
import type { PostInput } from "../schema/post";
import type { Origin, Post } from "../types";

export class DuplicateUrlError extends Error {
  constructor(public existingId: number) {
    super(`Already posted as post #${existingId}`);
  }
}

const POST_COLUMNS =
  "id, origin, submitted_by, run_id, title, url, source, summary, why_it_matters, discussion_question, tags, published_at, created_at";

type PostRow = Omit<Post, "tags"> & { tags: string };
const toPost = (row: PostRow): Post => ({ ...row, tags: JSON.parse(row.tags) as string[] });

export function findPostIdByUrl(db: Database, url: string): number | null {
  const row = db
    .query<{ id: number }, { u: string }>("SELECT id FROM posts WHERE url_normalized = $u")
    .get({ u: normalizeUrl(url) });
  return row?.id ?? null;
}

export function insertPost(
  db: Database,
  input: PostInput,
  meta: { origin: Origin; submittedBy: string | null; runId: number | null; now: Date },
): Post {
  const existing = findPostIdByUrl(db, input.url);
  if (existing !== null) throw new DuplicateUrlError(existing);
  const row = db
    .query<PostRow, Record<string, string | number | null>>(
      `INSERT INTO posts (origin, submitted_by, run_id, title, url, url_normalized, source, summary,
         why_it_matters, discussion_question, tags, published_at, created_at)
       VALUES ($origin, $submittedBy, $runId, $title, $url, $urlNormalized, $source, $summary,
         $whyItMatters, $discussionQuestion, $tags, $publishedAt, $createdAt)
       RETURNING ${POST_COLUMNS}`,
    )
    .get({
      origin: meta.origin,
      submittedBy: meta.submittedBy,
      runId: meta.runId,
      title: input.title,
      url: input.url,
      urlNormalized: normalizeUrl(input.url),
      source: input.source,
      summary: input.summary,
      whyItMatters: input.why_it_matters,
      discussionQuestion: input.discussion_question,
      tags: JSON.stringify(input.tags),
      publishedAt: input.published_at,
      createdAt: meta.now.toISOString(),
    });
  return toPost(row!);
}

export function getPost(db: Database, id: number): Post | null {
  const row = db.query<PostRow, { id: number }>(`SELECT ${POST_COLUMNS} FROM posts WHERE id = $id`).get({ id });
  return row ? toPost(row) : null;
}

/** Posts in [since, before), newest first. Both bounds are optional UTC ISO strings. */
export function listPosts(db: Database, range: { since?: string; before?: string }): Post[] {
  return db
    .query<PostRow, { since: string; before: string }>(
      `SELECT ${POST_COLUMNS} FROM posts
       WHERE created_at >= $since AND created_at < $before
       ORDER BY created_at DESC, id DESC`,
    )
    .all({ since: range.since ?? "", before: range.before ?? "9999" })
    .map(toPost);
}

export function recentPostRefs(db: Database, since: string): { title: string; url: string }[] {
  return listPosts(db, { since }).map(({ title, url }) => ({ title, url }));
}
```

`src/db/engagement.ts`:

```ts
import type { Database } from "bun:sqlite";
import type { Comment } from "../types";

const COMMENT_COLUMNS = "id, post_id, member_id, body, created_at, updated_at";

export function addComment(db: Database, postId: number, memberId: string, body: string, now: Date): Comment {
  return db
    .query<Comment, { postId: number; memberId: string; body: string; at: string }>(
      `INSERT INTO comments (post_id, member_id, body, created_at, updated_at)
       VALUES ($postId, $memberId, $body, $at, $at)
       RETURNING ${COMMENT_COLUMNS}`,
    )
    .get({ postId, memberId, body, at: now.toISOString() })!;
}

export function getComment(db: Database, id: number): Comment | null {
  return db.query<Comment, { id: number }>(`SELECT ${COMMENT_COLUMNS} FROM comments WHERE id = $id`).get({ id });
}

export function updateComment(db: Database, id: number, body: string, now: Date): Comment {
  return db
    .query<Comment, { id: number; body: string; at: string }>(
      `UPDATE comments SET body = $body, updated_at = $at WHERE id = $id RETURNING ${COMMENT_COLUMNS}`,
    )
    .get({ id, body, at: now.toISOString() })!;
}

export function deleteComment(db: Database, id: number): void {
  db.query("DELETE FROM comments WHERE id = $id").run({ id });
}

export function listComments(db: Database, postId: number): Comment[] {
  return db
    .query<Comment, { postId: number }>(
      `SELECT ${COMMENT_COLUMNS} FROM comments WHERE post_id = $postId ORDER BY created_at, id`,
    )
    .all({ postId });
}

export function setMark(db: Database, postId: number, memberId: string, now: Date): void {
  db.query("INSERT OR IGNORE INTO marks (post_id, member_id, created_at) VALUES ($postId, $memberId, $at)").run({
    postId,
    memberId,
    at: now.toISOString(),
  });
}

export function clearMark(db: Database, postId: number, memberId: string): void {
  db.query("DELETE FROM marks WHERE post_id = $postId AND member_id = $memberId").run({ postId, memberId });
}

export type Engagement = { commenters: Set<string>; markers: Set<string>; commentCount: number };

/** Who commented / marked on each of the given posts. */
export function engagementFor(db: Database, postIds: number[]): Map<number, Engagement> {
  const result = new Map<number, Engagement>();
  if (postIds.length === 0) return result;
  const ids = JSON.stringify(postIds);
  const entry = (postId: number) => {
    let e = result.get(postId);
    if (!e) {
      e = { commenters: new Set(), markers: new Set(), commentCount: 0 };
      result.set(postId, e);
    }
    return e;
  };
  const comments = db
    .query<{ post_id: number; member_id: string; n: number }, { ids: string }>(
      `SELECT post_id, member_id, COUNT(*) AS n FROM comments
       WHERE post_id IN (SELECT value FROM json_each($ids)) GROUP BY post_id, member_id`,
    )
    .all({ ids });
  for (const row of comments) {
    const e = entry(row.post_id);
    e.commenters.add(row.member_id);
    e.commentCount += row.n;
  }
  const marks = db
    .query<{ post_id: number; member_id: string }, { ids: string }>(
      "SELECT post_id, member_id FROM marks WHERE post_id IN (SELECT value FROM json_each($ids))",
    )
    .all({ ids });
  for (const row of marks) entry(row.post_id).markers.add(row.member_id);
  return result;
}
```

`src/feed.ts`:

```ts
import type { Database } from "bun:sqlite";
import { engagementFor, listComments } from "./db/engagement";
import { getPost, listPosts } from "./db/posts";
import { cellStatus } from "./domain/status";
import { windowStart } from "./domain/time";
import type { Board, FeedPost, Member, Post, PostDetail } from "./types";

function toFeedPosts(db: Database, posts: Post[], members: Member[]): FeedPost[] {
  const engagement = engagementFor(
    db,
    posts.map((p) => p.id),
  );
  return posts.map((post) => {
    const e = engagement.get(post.id);
    const status = Object.fromEntries(
      members.map((m) => [
        m.id,
        cellStatus({
          isAuthor: post.submitted_by === m.id,
          hasComment: e?.commenters.has(m.id) ?? false,
          hasMark: e?.markers.has(m.id) ?? false,
        }),
      ]),
    );
    return { ...post, comment_count: e?.commentCount ?? 0, status };
  });
}

export function getFeed(
  db: Database,
  members: Member[],
  scope: "recent" | "archive",
  now: Date,
  windowDays: number,
): FeedPost[] {
  const start = windowStart(now, windowDays);
  const posts = scope === "recent" ? listPosts(db, { since: start }) : listPosts(db, { before: start });
  return toFeedPosts(db, posts, members);
}

export function getBoard(db: Database, members: Member[], now: Date, windowDays: number): Board {
  const posts = getFeed(db, members, "recent", now, windowDays);
  const counts = Object.fromEntries(
    members.map((m) => [m.id, { done: posts.filter((p) => p.status[m.id] !== null).length, total: posts.length }]),
  );
  return { members, posts, counts };
}

export function getPostDetail(db: Database, members: Member[], id: number): PostDetail | null {
  const post = getPost(db, id);
  if (!post) return null;
  const [feedPost] = toFeedPosts(db, [post], members);
  return { ...feedPost!, comments: listComments(db, id) };
}
```

- [ ] **Step 5: Run tests and typecheck**

Run: `bun test && bun run typecheck`
Expected: all pass, typecheck clean.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: add post, comment and mark storage with feed and board queries

Closes #<issue>"
```

---

### Task 5: Seed script with realistic fake data

**Files:**
- Create: `scripts/seed.ts`

**Interfaces:**
- Consumes: `config`, `openDb`, `insertPost`, `addComment`, `setMark`, `PostInput`.
- Produces: `bun run seed` → fresh `data/dev.db` with 18 posts (14 in the last 7 days, 4 archived), member posts, comments and marks.

- [ ] **Step 1: Write the script**

`scripts/seed.ts`:

```ts
import { rmSync } from "node:fs";
import { config } from "../src/config";
import { openDb } from "../src/db/db";
import { addComment, setMark } from "../src/db/engagement";
import { insertPost } from "../src/db/posts";
import type { PostInput } from "../src/schema/post";

const REAL_DB = "data/dev-sharing.db";
if (config.dbPath === REAL_DB) {
  console.error(`Refusing to seed the real database (${REAL_DB}). Use: bun run seed`);
  process.exit(1);
}

for (const suffix of ["", "-wal", "-shm"]) rmSync(config.dbPath + suffix, { force: true });
const db = openDb(config.dbPath, config.members);

type Seed = Omit<PostInput, "url" | "published_at">;
const s = (title: string, source: string, tags: string[], summary: string, why: string, question: string): Seed => ({
  title,
  source,
  tags,
  summary,
  why_it_matters: why,
  discussion_question: question,
});

// Fictional products and sources: this is demo data only.
const SEEDS: Seed[] = [
  s("Tern 2.0 coding agent runs parallel sub-agents", "tern.example", ["agents", "tooling"],
    "Tern can now split a task into independent pieces, run them in parallel and merge the results. It works in the terminal and in CI.",
    "Big refactors and flaky-test sweeps could take a fraction of the time.",
    "Which ticket on our board would you hand to parallel agents first?"),
  s("Atlas 5 tops SWE benchmarks at half the price", "atlas-labs.example", ["models"],
    "A new frontier model claims the best score on real-repo coding benchmarks while halving token prices.",
    "Cheaper strong models change which tasks are worth automating.",
    "Would a 50% price cut change how much you use agents day to day?"),
  s("Startup cuts CI time 60% with an agent that triages flaky tests", "eng-blog.example", ["ci", "agents"],
    "An engineering team let an agent quarantine, reproduce and fix flaky tests. CI time dropped 60% in a month.",
    "Our CI is slow too; this is a concrete playbook we could copy.",
    "Should we trial this on our slowest pipeline?"),
  s("Survey: most developers now review more AI code than they write", "devsurvey.example", ["survey", "practices"],
    "A survey of 5,000 developers finds reviewing AI-generated code is now the main coding activity for most respondents.",
    "If review is the bottleneck, our review habits matter more than ever.",
    "How has your own review load changed this year?"),
  s("Quill editor ships inline agent reviews on every save", "quill.example", ["editors"],
    "The Quill editor now runs a lightweight review agent on save and annotates risky changes inline.",
    "Catching problems before a PR exists could shorten our review cycles.",
    "Would always-on review help you, or just be noise?"),
  s("Tool-calling protocol adds streaming results", "protocol.example", ["mcp", "protocols"],
    "The tool-calling protocol spec now lets servers stream partial results back to the model.",
    "Long-running tools (builds, test runs) become much more usable from agents.",
    "Which of our internal tools would benefit most from streaming?"),
  s("Prompt files in the repo are becoming a standard pattern", "patterns.example", ["practices"],
    "More teams keep agent instructions as versioned files next to the code, reviewed like any other change.",
    "We could make our agents behave consistently across the whole team.",
    "What would go in our first shared instructions file?"),
  s("Cloud provider cuts inference prices by 40%", "cloud.example", ["pricing"],
    "A major cloud provider cut prices for hosted model inference by 40% across all regions.",
    "Features we shelved as too expensive may now be viable.",
    "Which shelved AI feature should we revisit first?"),
  s("Half of the latest accelerator batch is agent-first", "vc-news.example", ["startups"],
    "Over half the companies in the latest accelerator batch build products where an agent does the core work.",
    "Our competitors may soon ship agent-first alternatives.",
    "Where could an agent-first competitor beat us?"),
  s("Postmortem: an agent deleted a staging database", "incidents.example", ["safety", "incidents"],
    "A team shares how an agent with broad credentials dropped a staging database, and which guardrails failed.",
    "We give agents credentials too; this is a checklist of what can go wrong.",
    "What permissions do our agents have that they do not need?"),
  s("A 30B coding model runs on a laptop at usable speed", "local-ai.example", ["local", "models"],
    "A new open-weight coding model runs locally on a recent laptop with acceptable latency.",
    "Local models could cover sensitive code we cannot send to an API.",
    "Would you switch to a local model for some tasks?"),
  s("Code review bots: measured impact at three startups", "metrics.example", ["review", "metrics"],
    "Three startups share before/after numbers on cycle time and defect rates after adding review bots.",
    "Real numbers help us decide whether a review bot is worth it.",
    "Which metric would convince you a review bot works?"),
  s("Browser agents get a standard permission model", "web-standards.example", ["browser", "security"],
    "Browser vendors agreed a permission model for agents acting on web pages on a user's behalf.",
    "Anything we build for browsers may need to respect these prompts.",
    "Does this affect any feature on our roadmap?"),
  s("New eval suite tests agents on real repos, not puzzles", "evals.example", ["evals"],
    "An open benchmark measures coding agents on multi-day tasks taken from real open-source repositories.",
    "Better evals make it easier to choose tools based on evidence.",
    "How do we currently decide which AI tool is better?"),
  s("Dev-tools startup raises $40M for agent observability", "vc-news.example", ["funding", "observability"],
    "A startup building tracing and replay for agent runs raised a $40M Series B.",
    "Debugging agent runs is a real pain; tooling is arriving.",
    "How do you debug an agent run that went wrong today?"),
  s("Voice pair-programming: a team reports after three months", "eng-blog.example", ["workflow"],
    "A team that switched to talking to their coding agent shares what worked and what did not.",
    "It is a cheap experiment that might suit some of us.",
    "Would you try voice for a week?"),
  s("Spec-driven development: write the spec for the agent first", "patterns.example", ["practices", "agents"],
    "Teams report better agent output when they write a short spec and plan before any code.",
    "It matches how we already like to plan features.",
    "Should specs become part of our ticket template?"),
  s("2M-token context windows: does retrieval still matter?", "atlas-labs.example", ["models", "rag"],
    "With 2M-token context windows, some teams are dropping retrieval pipelines entirely.",
    "Could simplify parts of our stack, or could be a cost trap.",
    "Where would you still keep retrieval?"),
];

const COMMENTS = [
  "We should try this on the billing service, its tests are the flakiest.",
  "Feels like hype until I see it on a real codebase. Anyone tried it?",
  "Tried it yesterday. Impressive on small tasks, struggled with our monorepo.",
  "This is exactly the kind of thing we argued about in the last retro.",
  "Bookmarked. Let's discuss in Friday's demo slot.",
  "The cost angle matters more to me than the benchmark.",
  "I'd want to see how it handles our auth flows before trusting it.",
  "Good read. The second half is the interesting part.",
];

const now = Date.now();
const HOUR = 3_600_000;
const memberIds = config.members.map((m) => m.id);

SEEDS.forEach((seed, i) => {
  const createdAt = new Date(now - (i * 12 + 1) * HOUR); // i = 14..17 fall outside the 7-day window
  const submittedBy = i % 5 === 2 ? memberIds[(i * 2) % memberIds.length]! : null;
  const post = insertPost(
    db,
    { ...seed, url: `https://${seed.source}/seed/${i + 1}`, published_at: createdAt.toISOString().slice(0, 10) },
    { origin: submittedBy ? "member" : "agent", submittedBy, runId: null, now: createdAt },
  );
  memberIds.forEach((memberId, j) => {
    if (memberId === submittedBy) return;
    if (i < 2 && j > 1) return; // the newest posts are still mostly unread
    const roll = (i * 7 + j * 3) % 6;
    const at = new Date(createdAt.getTime() + (j + 1) * 20 * 60_000);
    if (roll < 2) addComment(db, post.id, memberId, COMMENTS[(i + j) % COMMENTS.length]!, at);
    else if (roll === 2) setMark(db, post.id, memberId, at);
  });
});

const count = (sql: string) => db.query<{ n: number }, []>(sql).get()!.n;
console.log(
  `Seeded ${config.dbPath}: ${count("SELECT COUNT(*) AS n FROM posts")} posts, ` +
    `${count("SELECT COUNT(*) AS n FROM comments")} comments, ${count("SELECT COUNT(*) AS n FROM marks")} marks`,
);
```

- [ ] **Step 2: Run it**

Run: `bun run seed`
Expected: `Seeded data/dev.db: 18 posts, <n> comments, <m> marks` with n > 0 and m > 0.

Run: `DB_PATH=data/dev-sharing.db bun scripts/seed.ts; echo "exit=$?"`
Expected: `Refusing to seed the real database …` and `exit=1`.

- [ ] **Step 3: Check the window split**

Run:

```bash
DB_PATH=data/dev.db bun -e 'import {config} from "./src/config"; import {openDb} from "./src/db/db"; import {getFeed} from "./src/feed"; const db = openDb(config.dbPath, config.members); console.log(getFeed(db, config.members, "recent", new Date(), 7).length, getFeed(db, config.members, "archive", new Date(), 7).length)'
```

Expected: `14 4`

- [ ] **Step 4: Typecheck and commit**

Run: `bun run typecheck` (expected clean), then:

```bash
git add -A
git commit -m "feat: add dev seed script with realistic fake posts

Closes #<issue>"
```

---

### Task 6: Read API and member session

**Files:**
- Create: `src/api/common.ts`, `src/api/session.ts`, `src/api/posts.ts`, `src/api/app.ts`
- Test: `tests/api-read.test.ts`

**Interfaces:**
- Consumes: `getFeed`, `getBoard`, `getPostDetail`, `teammatePrompt`, `Config`.
- Produces:
  - `src/api/common.ts`: `type Env = { Variables: { memberId: string } }`; `MEMBER_COOKIE = "member"`; `type ApiDeps = { db: Database; config: Config; now?: () => Date; scheduler?: Scheduler }` (the `Scheduler` type is imported from `src/scheduler.ts` in Task 10; until then declare `scheduler?: unknown`); `memberGuard(config): MiddlewareHandler<Env>`; `readJson(c: Context): Promise<Record<string, unknown>>`
  - `createApi(deps: ApiDeps): Hono` from `src/api/app.ts` (routes under `/api`)
  - Routes: `GET /api/members`, `GET|POST|DELETE /api/session`, `GET /api/feed?scope=recent|archive`, `GET /api/posts/:id`, `GET /api/board`, `GET /api/submit-prompt`. All but members/session return 401 `{ error }` without a valid `member` cookie. Unknown `/api/*` → 404 `{ error: "Not found" }`.

- [ ] **Step 1: Write the failing tests**

`tests/api-read.test.ts`:

```ts
import { expect, test } from "bun:test";
import { createApi } from "../src/api/app";
import { config } from "../src/config";
import { addComment } from "../src/db/engagement";
import { insertPost } from "../src/db/posts";
import { samplePost, testDb } from "./helpers";

const NOW = new Date("2026-10-02T12:00:00Z");
const hoursAgo = (h: number) => new Date(NOW.getTime() - h * 3_600_000);
const as = (member: string): RequestInit => ({ headers: { Cookie: `member=${member}` } });

function setup() {
  const db = testDb();
  const app = createApi({ db, config, now: () => NOW });
  const recent = insertPost(db, samplePost({ title: "Recent post" }), { origin: "agent", submittedBy: null, runId: null, now: hoursAgo(1) });
  insertPost(db, samplePost({ title: "Archived post" }), { origin: "agent", submittedBy: null, runId: null, now: hoursAgo(24 * 9) });
  addComment(db, recent.id, "henry", "Interesting", NOW);
  return { db, app, recent };
}

test("GET /api/members lists the six members", async () => {
  const { app } = setup();
  const res = await app.request("/api/members");
  expect(res.status).toBe(200);
  expect(((await res.json()) as { code: string }[]).map((m) => m.code)).toEqual(["SJ", "SK", "HE", "PA", "PD", "BE"]);
});

test("session: none without cookie, set by POST, cleared by DELETE, unknown member rejected", async () => {
  const { app } = setup();
  expect(await (await app.request("/api/session")).json()).toEqual({ memberId: null });
  const set = await app.request("/api/session", { method: "POST", body: JSON.stringify({ memberId: "sojeong" }) });
  expect(set.status).toBe(200);
  expect(set.headers.get("Set-Cookie")).toContain("member=sojeong");
  expect(await (await app.request("/api/session", as("sojeong"))).json()).toEqual({ memberId: "sojeong" });
  const bad = await app.request("/api/session", { method: "POST", body: JSON.stringify({ memberId: "mallory" }) });
  expect(bad.status).toBe(400);
  const cleared = await app.request("/api/session", { method: "DELETE", ...as("sojeong") });
  expect(cleared.status).toBe(204);
  expect(cleared.headers.get("Set-Cookie")).toContain("member=;");
});

test("member-only routes return 401 without a valid cookie", async () => {
  const { app } = setup();
  for (const path of ["/api/feed", "/api/board", "/api/posts/1", "/api/submit-prompt"]) {
    expect((await app.request(path)).status).toBe(401);
  }
  expect((await app.request("/api/feed", as("mallory"))).status).toBe(401);
});

test("GET /api/feed returns recent posts by default and older ones with scope=archive", async () => {
  const { app } = setup();
  const recent = (await (await app.request("/api/feed", as("ben"))).json()) as { title: string; status: Record<string, unknown>; comment_count: number }[];
  expect(recent.map((p) => p.title)).toEqual(["Recent post"]);
  expect(recent[0]!.status.henry).toBe("commented");
  expect(recent[0]!.comment_count).toBe(1);
  const archive = (await (await app.request("/api/feed?scope=archive", as("ben"))).json()) as { title: string }[];
  expect(archive.map((p) => p.title)).toEqual(["Archived post"]);
});

test("GET /api/posts/:id returns post with comments; 404 when missing", async () => {
  const { app, recent } = setup();
  const res = await app.request(`/api/posts/${recent.id}`, as("ben"));
  expect(res.status).toBe(200);
  const body = (await res.json()) as { title: string; comments: { body: string }[] };
  expect(body.title).toBe("Recent post");
  expect(body.comments.map((c) => c.body)).toEqual(["Interesting"]);
  expect((await app.request("/api/posts/999", as("ben"))).status).toBe(404);
});

test("GET /api/board returns members, recent posts and counts", async () => {
  const { app } = setup();
  const board = (await (await app.request("/api/board", as("ben"))).json()) as {
    members: unknown[];
    posts: unknown[];
    counts: Record<string, { done: number; total: number }>;
  };
  expect(board.members).toHaveLength(6);
  expect(board.posts).toHaveLength(1);
  expect(board.counts.henry).toEqual({ done: 1, total: 1 });
});

test("GET /api/submit-prompt returns the teammate prompt", async () => {
  const { app } = setup();
  const body = (await (await app.request("/api/submit-prompt", as("ben"))).json()) as { prompt: string };
  expect(body.prompt).toContain("- discussion_question:");
});

test("unknown API routes return a JSON 404", async () => {
  const { app } = setup();
  const res = await app.request("/api/nope", as("ben"));
  expect(res.status).toBe(404);
  expect(await res.json()).toEqual({ error: "Not found" });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `bun test tests/api-read.test.ts`
Expected: FAIL, `../src/api/app` not found.

- [ ] **Step 3: Implement**

`src/api/common.ts`:

```ts
import type { Database } from "bun:sqlite";
import type { Context } from "hono";
import { getCookie } from "hono/cookie";
import { createMiddleware } from "hono/factory";
import type { Config } from "../config";

export type Env = { Variables: { memberId: string } };

export const MEMBER_COOKIE = "member";

export type ApiDeps = {
  db: Database;
  config: Config;
  now?: () => Date;
  scheduler?: unknown; // replaced by the Scheduler type in Task 10
};

/** Rejects requests without a known member cookie; exposes the member id as c.get("memberId"). */
export function memberGuard(config: Config) {
  const ids = new Set(config.members.map((m) => m.id));
  return createMiddleware<Env>(async (c, next) => {
    const id = getCookie(c, MEMBER_COOKIE);
    if (!id || !ids.has(id)) return c.json({ error: "Pick your name first" }, 401);
    c.set("memberId", id);
    await next();
  });
}

export async function readJson(c: Context): Promise<Record<string, unknown>> {
  try {
    const body: unknown = await c.req.json();
    return body && typeof body === "object" ? (body as Record<string, unknown>) : {};
  } catch {
    return {};
  }
}
```

`src/api/session.ts`:

```ts
import { Hono } from "hono";
import { deleteCookie, getCookie, setCookie } from "hono/cookie";
import { type ApiDeps, type Env, MEMBER_COOKIE, readJson } from "./common";

export function sessionRoutes({ config }: ApiDeps) {
  const r = new Hono<Env>();
  const ids = new Set(config.members.map((m) => m.id));

  r.get("/members", (c) => c.json(config.members));

  r.get("/session", (c) => {
    const id = getCookie(c, MEMBER_COOKIE);
    return c.json({ memberId: id && ids.has(id) ? id : null });
  });

  r.post("/session", async (c) => {
    const { memberId } = await readJson(c);
    if (typeof memberId !== "string" || !ids.has(memberId)) return c.json({ error: "Unknown member" }, 400);
    setCookie(c, MEMBER_COOKIE, memberId, { path: "/", httpOnly: true, sameSite: "Lax", maxAge: 60 * 60 * 24 * 365 });
    return c.json({ memberId });
  });

  r.delete("/session", (c) => {
    deleteCookie(c, MEMBER_COOKIE, { path: "/" });
    return c.body(null, 204);
  });

  return r;
}
```

`src/api/posts.ts`:

```ts
import { Hono } from "hono";
import { getBoard, getFeed, getPostDetail } from "../feed";
import { teammatePrompt } from "../schema/post";
import { type ApiDeps, type Env, memberGuard } from "./common";

export function postRoutes({ db, config, now = () => new Date() }: ApiDeps) {
  const r = new Hono<Env>();
  const member = memberGuard(config);

  r.get("/feed", member, (c) => {
    const scope = c.req.query("scope") === "archive" ? "archive" : "recent";
    return c.json(getFeed(db, config.members, scope, now(), config.windowDays));
  });

  r.get("/posts/:id{[0-9]+}", member, (c) => {
    const post = getPostDetail(db, config.members, Number(c.req.param("id")));
    return post ? c.json(post) : c.json({ error: "Post not found" }, 404);
  });

  r.get("/board", member, (c) => c.json(getBoard(db, config.members, now(), config.windowDays)));

  r.get("/submit-prompt", member, (c) => c.json({ prompt: teammatePrompt() }));

  return r;
}
```

`src/api/app.ts`:

```ts
import { Hono } from "hono";
import type { ApiDeps, Env } from "./common";
import { postRoutes } from "./posts";
import { sessionRoutes } from "./session";

export function createApi(deps: ApiDeps) {
  const app = new Hono<Env>().basePath("/api");
  app.route("/", sessionRoutes(deps));
  app.route("/", postRoutes(deps));
  app.notFound((c) => c.json({ error: "Not found" }, 404));
  return app;
}
```

Note: apply `memberGuard` per route as shown. Do **not** use `r.use("*", …)` inside a sub-app mounted at `/`: it would also guard `/api/session` and `/api/members`.

- [ ] **Step 4: Run tests and typecheck**

Run: `bun test && bun run typecheck`
Expected: all pass, typecheck clean.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add read API and member session

Closes #<issue>"
```

---

### Task 7: Web shell, name picker and home feed

**Files:**
- Create: `bunfig.toml`, `src/server.ts`
- Create: `web/index.html`, `web/styles.css`, `web/main.tsx`, `web/App.tsx`, `web/api.ts`, `web/me.ts`, `web/format.ts`, `web/status.ts`
- Create: `web/components/MemberChip.tsx`, `web/components/OriginBadge.tsx`, `web/components/PostCard.tsx`, `web/components/PostList.tsx`, `web/components/ProgressStrip.tsx`, `web/components/Header.tsx`
- Create: `web/pages/PickName.tsx`, `web/pages/Home.tsx`
- Test: `tests/format.test.ts`

**Interfaces:**
- Consumes: `createApi`, `openDb`, `config`, types from `src/types.ts`, `PostInput` type.
- Produces:
  - `api` client object and hooks `useMembers`, `useSession`, `useFeed(scope)`, `usePost(id)`, `useBoard()` from `web/api.ts` (the client already includes write and runs calls used by Tasks 9–10); `ApiError` with `status` and `body`.
  - `MeContext`, `useMe(): string` from `web/me.ts`.
  - `dayLabel(iso, now?)`, `timeLabel(iso)`, `groupByDay(items)` from `web/format.ts`.
  - `statusLabel(s)`, `cellSymbol(s)` from `web/status.ts`.
  - Components `MemberChip({ member, filled, title? })`, `OriginBadge({ post, members })`, `PostCard({ post, members })`, `PostList({ posts, members, empty })`, `ProgressStrip()`, `Header()`.
  - `App` routes: `/` (Home). Later tasks add `/posts/:id`, `/board`, `/archive`, `/submit`, `/runs`.

- [ ] **Step 1: Write the failing test for the pure helpers**

`tests/format.test.ts`:

```ts
import { expect, test } from "bun:test";
import { dayLabel, groupByDay } from "../web/format";
import { cellSymbol, statusLabel } from "../web/status";

const NOW = new Date(2026, 9, 2, 15, 0); // local time, 2 Oct 2026 15:00

test("dayLabel says Today and Yesterday, otherwise a date", () => {
  expect(dayLabel(new Date(2026, 9, 2, 9, 0).toISOString(), NOW)).toBe("Today");
  expect(dayLabel(new Date(2026, 9, 1, 23, 0).toISOString(), NOW)).toBe("Yesterday");
  expect(dayLabel(new Date(2026, 8, 28, 10, 0).toISOString(), NOW)).not.toMatch(/Today|Yesterday/);
});

test("groupByDay keeps order and groups consecutive items of the same day", () => {
  const items = [
    { id: 1, created_at: new Date(2026, 9, 2, 14, 0).toISOString() },
    { id: 2, created_at: new Date(2026, 9, 2, 10, 0).toISOString() },
    { id: 3, created_at: new Date(2026, 9, 1, 16, 0).toISOString() },
  ];
  const groups = groupByDay(items, NOW);
  expect(groups.map((g) => [g.label, g.items.map((i) => i.id)])).toEqual([
    ["Today", [1, 2]],
    ["Yesterday", [3]],
  ]);
});

test("status labels and board symbols", () => {
  expect(statusLabel("commented")).toBe("commented");
  expect(statusLabel(null)).toBe("not yet");
  expect([cellSymbol("commented"), cellSymbol("nothing"), cellSymbol("author"), cellSymbol(null)]).toEqual(["✓", "–", "★", ""]);
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `bun test tests/format.test.ts`
Expected: FAIL, modules not found.

- [ ] **Step 3: Implement the helpers**

`web/format.ts`:

```ts
const startOfDay = (d: Date) => new Date(d.getFullYear(), d.getMonth(), d.getDate()).getTime();

export function dayLabel(iso: string, now = new Date()): string {
  const d = new Date(iso);
  const diff = Math.round((startOfDay(now) - startOfDay(d)) / 86_400_000);
  if (diff === 0) return "Today";
  if (diff === 1) return "Yesterday";
  return d.toLocaleDateString(undefined, { weekday: "long", day: "numeric", month: "short" });
}

export const timeLabel = (iso: string) =>
  new Date(iso).toLocaleTimeString(undefined, { hour: "2-digit", minute: "2-digit" });

export function groupByDay<T extends { created_at: string }>(items: T[], now = new Date()) {
  const groups: { label: string; items: T[] }[] = [];
  for (const item of items) {
    const label = dayLabel(item.created_at, now);
    const last = groups.at(-1);
    if (last?.label === label) last.items.push(item);
    else groups.push({ label, items: [item] });
  }
  return groups;
}
```

`web/status.ts`:

```ts
import type { CellStatus } from "../src/types";

export function statusLabel(s: CellStatus | undefined): string {
  if (s === "author") return "shared it";
  if (s === "commented") return "commented";
  if (s === "nothing") return "read, nothing to add";
  return "not yet";
}

export function cellSymbol(s: CellStatus | undefined): string {
  if (s === "commented") return "✓";
  if (s === "nothing") return "–";
  if (s === "author") return "★";
  return "";
}
```

Run: `bun test tests/format.test.ts` → PASS.

- [ ] **Step 4: Create the server entry and bundler config**

`bunfig.toml`:

```toml
[serve.static]
plugins = ["bun-plugin-tailwind"]
```

`src/server.ts`:

```ts
import index from "../web/index.html";
import { createApi } from "./api/app";
import { config } from "./config";
import { openDb } from "./db/db";

const db = openDb(config.dbPath, config.members);
const api = createApi({ db, config });

const server = Bun.serve({
  port: config.port,
  development: process.env.NODE_ENV !== "production",
  routes: {
    "/api/*": api.fetch,
    "/*": index,
  },
});

console.log(`Dev Sharing running at ${server.url} (db: ${config.dbPath}, agent: ${config.agent})`);
```

- [ ] **Step 5: Create the web shell**

`web/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Dev Sharing</title>
    <link rel="stylesheet" href="./styles.css" />
  </head>
  <body class="bg-zinc-50 text-zinc-900 antialiased dark:bg-zinc-950 dark:text-zinc-100">
    <div id="root"></div>
    <script type="module" src="./main.tsx"></script>
  </body>
</html>
```

`web/styles.css`:

```css
@import "tailwindcss";
```

`web/main.tsx`:

```tsx
import { QueryClientProvider } from "@tanstack/react-query";
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router";
import { queryClient } from "./api";
import { App } from "./App";

createRoot(document.getElementById("root")!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </QueryClientProvider>
  </StrictMode>,
);
```

`web/me.ts`:

```ts
import { createContext, useContext } from "react";

export const MeContext = createContext<string | null>(null);

export function useMe(): string {
  const me = useContext(MeContext);
  if (!me) throw new Error("useMe() used outside MeContext");
  return me;
}
```

`web/api.ts`:

```ts
import { QueryClient, useQuery } from "@tanstack/react-query";
import type { PostInput } from "../src/schema/post";
import type { Board, Comment, FeedPost, Member, Post, PostDetail, RunsView } from "../src/types";

export class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
    public body: Record<string, unknown> = {},
  ) {
    super(message);
  }
}

async function request<T>(path: string, init: RequestInit = {}): Promise<T> {
  const res = await fetch(`/api${path}`, {
    ...init,
    headers: { "Content-Type": "application/json", ...init.headers },
  });
  const body = res.status === 204 ? null : await res.json().catch(() => null);
  if (!res.ok) throw new ApiError(res.status, body?.error ?? res.statusText, body ?? {});
  return body as T;
}

const send = (method: string, body?: unknown): RequestInit => ({
  method,
  body: body === undefined ? undefined : JSON.stringify(body),
});

export const api = {
  members: () => request<Member[]>("/members"),
  session: () => request<{ memberId: string | null }>("/session"),
  pickMember: (memberId: string) => request<{ memberId: string }>("/session", send("POST", { memberId })),
  clearSession: () => request<null>("/session", send("DELETE")),
  feed: (scope: "recent" | "archive") => request<FeedPost[]>(`/feed?scope=${scope}`),
  post: (id: number) => request<PostDetail>(`/posts/${id}`),
  board: () => request<Board>("/board"),
  submitPrompt: () => request<{ prompt: string }>("/submit-prompt"),
  previewPost: (text: string) => request<{ post: PostInput }>("/posts/preview", send("POST", { text })),
  publishPost: (text: string) => request<Post>("/posts", send("POST", { text })),
  addComment: (postId: number, body: string) => request<Comment>(`/posts/${postId}/comments`, send("POST", { body })),
  editComment: (id: number, body: string) => request<Comment>(`/comments/${id}`, send("PATCH", { body })),
  deleteComment: (id: number) => request<null>(`/comments/${id}`, send("DELETE")),
  setMark: (postId: number) => request<null>(`/posts/${postId}/mark`, send("PUT")),
  clearMark: (postId: number) => request<null>(`/posts/${postId}/mark`, send("DELETE")),
  runs: () => request<RunsView>("/runs"),
  runNow: () => request<{ started: true }>("/runs", send("POST")),
};

export const queryClient = new QueryClient({
  defaultOptions: { queries: { refetchInterval: 30_000, refetchOnWindowFocus: true, retry: 1 } },
});

export const useMembers = () =>
  useQuery({ queryKey: ["members"], queryFn: api.members, staleTime: Infinity, refetchInterval: false });
export const useSession = () => useQuery({ queryKey: ["session"], queryFn: api.session, refetchInterval: false });
export const useFeed = (scope: "recent" | "archive") =>
  useQuery({ queryKey: ["feed", scope], queryFn: () => api.feed(scope) });
export const usePost = (id: number) => useQuery({ queryKey: ["post", id], queryFn: () => api.post(id) });
export const useBoard = () => useQuery({ queryKey: ["board"], queryFn: api.board });
```

`web/components/MemberChip.tsx`:

```tsx
import type { Member } from "../../src/types";

export function MemberChip({ member, filled, title }: { member: Member; filled: boolean; title?: string }) {
  return (
    <span
      title={title ?? member.name}
      className="inline-flex h-6 min-w-7 items-center justify-center rounded-full border-2 px-1 text-[10px] font-bold tracking-wide"
      style={{
        borderColor: member.color,
        backgroundColor: filled ? member.color : "transparent",
        color: filled ? "#fff" : member.color,
      }}
    >
      {member.code}
    </span>
  );
}
```

`web/components/OriginBadge.tsx`:

```tsx
import type { Member, Post } from "../../src/types";

export function OriginBadge({ post, members }: { post: Post; members: Member[] }) {
  if (post.origin === "agent") {
    return (
      <span className="shrink-0 rounded-full bg-zinc-100 px-2 py-0.5 text-[11px] text-zinc-600 dark:bg-zinc-800 dark:text-zinc-300">
        agent
      </span>
    );
  }
  const m = members.find((x) => x.id === post.submitted_by);
  return (
    <span
      className="shrink-0 rounded-full px-2 py-0.5 text-[11px] font-medium text-white"
      style={{ backgroundColor: m?.color ?? "#71717a" }}
    >
      found by {m?.name ?? "someone"}
    </span>
  );
}
```

`web/components/PostCard.tsx`:

```tsx
import { Link } from "react-router";
import type { FeedPost, Member } from "../../src/types";
import { timeLabel } from "../format";
import { useMe } from "../me";
import { statusLabel } from "../status";
import { MemberChip } from "./MemberChip";
import { OriginBadge } from "./OriginBadge";

export function PostCard({ post, members }: { post: FeedPost; members: Member[] }) {
  const me = useMe();
  const unread = post.status[me] === null;
  return (
    <Link
      to={`/posts/${post.id}`}
      className={`block rounded-xl border bg-white p-4 shadow-sm transition hover:shadow-md dark:bg-zinc-900 ${
        unread ? "border-amber-400 dark:border-amber-500" : "border-zinc-200 dark:border-zinc-800"
      }`}
    >
      <div className="flex items-start gap-2">
        {unread && <span className="mt-0.5 rounded bg-amber-400 px-1.5 text-[10px] font-bold text-zinc-900">NEW</span>}
        <h3 className="flex-1 font-semibold leading-snug">{post.title}</h3>
        <OriginBadge post={post} members={members} />
      </div>
      <p className="mt-1 line-clamp-2 text-sm text-zinc-600 dark:text-zinc-400">{post.summary}</p>
      <div className="mt-3 flex flex-wrap items-center gap-3 text-xs text-zinc-500">
        <span>💬 {post.comment_count}</span>
        <span className="flex gap-1">
          {members.map((m) => (
            <MemberChip
              key={m.id}
              member={m}
              filled={post.status[m.id] != null}
              title={`${m.name}: ${statusLabel(post.status[m.id])}`}
            />
          ))}
        </span>
        <span className="ml-auto">
          {timeLabel(post.created_at)} · {post.source}
        </span>
      </div>
    </Link>
  );
}
```

`web/components/PostList.tsx`:

```tsx
import type { FeedPost, Member } from "../../src/types";
import { groupByDay } from "../format";
import { PostCard } from "./PostCard";

export function PostList({ posts, members, empty }: { posts: FeedPost[]; members: Member[]; empty: string }) {
  if (posts.length === 0) {
    return (
      <p className="rounded-xl border border-dashed border-zinc-300 p-8 text-center text-zinc-500 dark:border-zinc-700">
        {empty}
      </p>
    );
  }
  return (
    <div className="space-y-6">
      {groupByDay(posts).map((group) => (
        <section key={group.label} className="space-y-2">
          <h2 className="text-xs font-semibold uppercase tracking-wide text-zinc-500">{group.label}</h2>
          {group.items.map((post) => (
            <PostCard key={post.id} post={post} members={members} />
          ))}
        </section>
      ))}
    </div>
  );
}
```

`web/components/ProgressStrip.tsx`:

```tsx
import { useBoard } from "../api";

export function ProgressStrip() {
  const board = useBoard().data;
  if (!board) return null;
  return (
    <div className="grid grid-cols-2 gap-2 sm:grid-cols-3">
      {board.members.map((m) => {
        const c = board.counts[m.id] ?? { done: 0, total: 0 };
        const pct = c.total ? Math.round((c.done / c.total) * 100) : 100;
        return (
          <div key={m.id} className="rounded-lg border border-zinc-200 bg-white p-2 dark:border-zinc-800 dark:bg-zinc-900">
            <div className="flex items-center justify-between text-sm">
              <span className="font-medium">{m.name}</span>
              <span className="tabular-nums text-zinc-500">
                {c.done}/{c.total}
              </span>
            </div>
            <div className="mt-1 h-1.5 rounded-full bg-zinc-100 dark:bg-zinc-800">
              <div className="h-1.5 rounded-full" style={{ width: `${pct}%`, backgroundColor: m.color }} />
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

`web/components/Header.tsx`:

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { NavLink } from "react-router";
import { api, useMembers } from "../api";
import { useMe } from "../me";
import { MemberChip } from "./MemberChip";

const LINKS = [
  { to: "/", label: "Home", end: true },
  { to: "/board", label: "Board" },
  { to: "/archive", label: "Archive" },
  { to: "/submit", label: "Submit" },
  { to: "/runs", label: "Agent" },
];

export function Header() {
  const me = useMe();
  const member = (useMembers().data ?? []).find((m) => m.id === me);
  const qc = useQueryClient();
  const switchUser = useMutation({
    mutationFn: api.clearSession,
    onSuccess: () => qc.invalidateQueries({ queryKey: ["session"] }),
  });
  return (
    <header className="sticky top-0 z-10 border-b border-zinc-200 bg-white/80 backdrop-blur dark:border-zinc-800 dark:bg-zinc-900/80">
      <div className="mx-auto flex max-w-3xl items-center gap-4 px-4 py-3">
        <span className="font-semibold">Dev Sharing</span>
        <nav className="flex gap-1 overflow-x-auto text-sm">
          {LINKS.map((l) => (
            <NavLink
              key={l.to}
              to={l.to}
              end={l.end}
              className={({ isActive }) =>
                `rounded-md px-2 py-1 whitespace-nowrap ${
                  isActive
                    ? "bg-zinc-900 text-white dark:bg-zinc-100 dark:text-zinc-900"
                    : "text-zinc-600 hover:bg-zinc-100 dark:text-zinc-400 dark:hover:bg-zinc-800"
                }`
              }
            >
              {l.label}
            </NavLink>
          ))}
        </nav>
        <div className="ml-auto flex items-center gap-2 text-sm">
          {member && <MemberChip member={member} filled />}
          <span className="hidden sm:inline">{member?.name}</span>
          <button
            type="button"
            onClick={() => switchUser.mutate()}
            className="text-zinc-500 underline-offset-2 hover:underline"
          >
            switch
          </button>
        </div>
      </div>
    </header>
  );
}
```

`web/pages/PickName.tsx`:

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { api, useMembers } from "../api";
import { MemberChip } from "../components/MemberChip";

export function PickName() {
  const members = useMembers().data ?? [];
  const qc = useQueryClient();
  const pick = useMutation({
    mutationFn: api.pickMember,
    onSuccess: () => qc.invalidateQueries({ queryKey: ["session"] }),
  });
  return (
    <main className="mx-auto flex min-h-screen max-w-md flex-col justify-center px-4">
      <h1 className="text-2xl font-bold">Dev Sharing</h1>
      <p className="mt-1 text-zinc-500">Who are you?</p>
      <div className="mt-6 grid grid-cols-2 gap-3">
        {members.map((m) => (
          <button
            key={m.id}
            type="button"
            onClick={() => pick.mutate(m.id)}
            className="flex items-center gap-3 rounded-xl border border-zinc-200 bg-white p-3 text-left hover:border-zinc-400 dark:border-zinc-800 dark:bg-zinc-900 dark:hover:border-zinc-600"
          >
            <MemberChip member={m} filled />
            <span className="font-medium">{m.name}</span>
          </button>
        ))}
      </div>
    </main>
  );
}
```

`web/pages/Home.tsx`:

```tsx
import { useState } from "react";
import { useFeed, useMembers } from "../api";
import { PostList } from "../components/PostList";
import { ProgressStrip } from "../components/ProgressStrip";
import { useMe } from "../me";

export function Home() {
  const me = useMe();
  const [onlyUnread, setOnlyUnread] = useState(false);
  const feed = useFeed("recent");
  const members = useMembers().data ?? [];
  const posts = (feed.data ?? []).filter((p) => !onlyUnread || p.status[me] === null);
  return (
    <div className="space-y-6">
      <ProgressStrip />
      <div className="flex items-center justify-between">
        <h1 className="text-lg font-semibold">Last 7 days</h1>
        <div className="flex rounded-lg border border-zinc-200 p-0.5 text-sm dark:border-zinc-800">
          {[
            { label: "All", value: false },
            { label: "My unread", value: true },
          ].map((o) => (
            <button
              key={o.label}
              type="button"
              onClick={() => setOnlyUnread(o.value)}
              className={`rounded-md px-3 py-1 ${
                onlyUnread === o.value ? "bg-zinc-900 text-white dark:bg-zinc-100 dark:text-zinc-900" : "text-zinc-600 dark:text-zinc-400"
              }`}
            >
              {o.label}
            </button>
          ))}
        </div>
      </div>
      {feed.isPending ? (
        <p className="text-zinc-500">Loading…</p>
      ) : (
        <PostList posts={posts} members={members} empty={onlyUnread ? "You're all caught up 🎉" : "No posts yet."} />
      )}
    </div>
  );
}
```

`web/App.tsx`:

```tsx
import { Navigate, Route, Routes } from "react-router";
import { useSession } from "./api";
import { Header } from "./components/Header";
import { MeContext } from "./me";
import { Home } from "./pages/Home";
import { PickName } from "./pages/PickName";

export function App() {
  const session = useSession();
  if (session.isPending) return null;
  const me = session.data?.memberId;
  if (!me) return <PickName />;
  return (
    <MeContext value={me}>
      <Header />
      <main className="mx-auto max-w-3xl px-4 py-6">
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="*" element={<Navigate to="/" replace />} />
        </Routes>
      </main>
    </MeContext>
  );
}
```

- [ ] **Step 6: Typecheck, run, and look at it**

Run: `bun test && bun run typecheck` → all pass, clean.

Run: `bun run seed && bun run dev` (leave running), then in another terminal:

```bash
curl -s localhost:3000/api/members | head -c 120; echo
curl -s localhost:3000/ | grep -c 'id="root"'
curl -s localhost:3000/board | grep -c 'id="root"'
```

Expected: members JSON; `1`; `1` (SPA served for client routes).

Open `http://localhost:3000` in a browser: pick a name → home shows the 6-member progress strip, "Last 7 days" with 14 cards grouped by day, NEW badges on posts you have not engaged with, colored 2-letter chips, "My unread" filter works, "switch" returns to the name picker. Check both light and dark system themes.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: add web shell, name picker and home feed

Closes #<issue>"
```

---

### Task 8: Post page, team board and archive (read-only)

**Files:**
- Create: `web/components/StatusRow.tsx`, `web/components/Comments.tsx`
- Create: `web/pages/PostPage.tsx`, `web/pages/BoardPage.tsx`, `web/pages/Archive.tsx`
- Modify: `web/App.tsx` (add routes)

**Interfaces:**
- Consumes: `usePost`, `useBoard`, `useFeed`, `useMembers`, `MemberChip`, `OriginBadge`, `PostList`, `dayLabel`, `timeLabel`, `statusLabel`, `cellSymbol`.
- Produces: `StatusRow({ post, members })`; `Comments({ post, members })` (read-only here, replaced with the interactive version in Task 9); routes `/posts/:id`, `/board`, `/archive`.

- [ ] **Step 1: Create the components**

`web/components/StatusRow.tsx`:

```tsx
import type { FeedPost, Member } from "../../src/types";
import { statusLabel } from "../status";
import { MemberChip } from "./MemberChip";

export function StatusRow({ post, members }: { post: FeedPost; members: Member[] }) {
  return (
    <div className="flex flex-wrap gap-x-4 gap-y-2">
      {members.map((m) => (
        <span key={m.id} className="flex items-center gap-1.5 text-xs text-zinc-500">
          <MemberChip member={m} filled={post.status[m.id] != null} />
          {statusLabel(post.status[m.id])}
        </span>
      ))}
    </div>
  );
}
```

`web/components/Comments.tsx`:

```tsx
import type { Comment, Member, PostDetail } from "../../src/types";
import { dayLabel, timeLabel } from "../format";
import { MemberChip } from "./MemberChip";

export function Comments({ post, members }: { post: PostDetail; members: Member[] }) {
  return (
    <section className="space-y-3">
      <h2 className="text-sm font-semibold uppercase tracking-wide text-zinc-500">Comments ({post.comments.length})</h2>
      {post.comments.length === 0 && <p className="text-sm text-zinc-500">No comments yet. Be the first.</p>}
      <ul className="space-y-3">
        {post.comments.map((c) => (
          <CommentItem key={c.id} comment={c} member={members.find((m) => m.id === c.member_id)} />
        ))}
      </ul>
    </section>
  );
}

function CommentItem({ comment, member }: { comment: Comment; member?: Member }) {
  return (
    <li className="rounded-xl border border-zinc-200 bg-white p-3 dark:border-zinc-800 dark:bg-zinc-900">
      <div className="flex items-center gap-2 text-xs text-zinc-500">
        {member && <MemberChip member={member} filled />}
        <span className="font-medium text-zinc-800 dark:text-zinc-200">{member?.name ?? comment.member_id}</span>
        <span>
          {dayLabel(comment.created_at)} {timeLabel(comment.created_at)}
          {comment.updated_at !== comment.created_at && " · edited"}
        </span>
      </div>
      <p className="mt-2 whitespace-pre-wrap text-sm">{comment.body}</p>
    </li>
  );
}
```

- [ ] **Step 2: Create the pages**

`web/pages/PostPage.tsx`:

```tsx
import type { ReactNode } from "react";
import { Link, useParams } from "react-router";
import { useMembers, usePost } from "../api";
import { Comments } from "../components/Comments";
import { OriginBadge } from "../components/OriginBadge";
import { StatusRow } from "../components/StatusRow";
import { dayLabel, timeLabel } from "../format";

function Section({ title, children, highlight }: { title: string; children: ReactNode; highlight?: boolean }) {
  return (
    <section
      className={`rounded-xl p-4 ${
        highlight
          ? "border border-amber-300 bg-amber-50 dark:border-amber-700 dark:bg-amber-950/40"
          : "border border-zinc-200 bg-white dark:border-zinc-800 dark:bg-zinc-900"
      }`}
    >
      <h2 className="text-xs font-semibold uppercase tracking-wide text-zinc-500">{title}</h2>
      <p className="mt-1 leading-relaxed">{children}</p>
    </section>
  );
}

export function PostPage() {
  const id = Number(useParams().id);
  const post = usePost(id);
  const members = useMembers().data ?? [];
  if (post.isPending) return <p className="text-zinc-500">Loading…</p>;
  if (post.isError)
    return (
      <p>
        Post not found.{" "}
        <Link to="/" className="underline">
          Back home
        </Link>
      </p>
    );
  const p = post.data;
  return (
    <article className="space-y-5">
      <header className="space-y-2">
        <div className="flex flex-wrap items-center gap-2 text-xs text-zinc-500">
          <span>
            {dayLabel(p.created_at)} {timeLabel(p.created_at)} · {p.source}
          </span>
          <OriginBadge post={p} members={members} />
        </div>
        <h1 className="text-2xl font-bold leading-tight">{p.title}</h1>
        <a href={p.url} target="_blank" rel="noreferrer" className="inline-block break-all text-sm text-blue-600 hover:underline dark:text-blue-400">
          {p.url} ↗
        </a>
        {p.tags.length > 0 && (
          <div className="flex gap-1">
            {p.tags.map((t) => (
              <span key={t} className="rounded-full bg-zinc-100 px-2 py-0.5 text-[11px] text-zinc-600 dark:bg-zinc-800 dark:text-zinc-300">
                #{t}
              </span>
            ))}
          </div>
        )}
      </header>
      <Section title="What it is">{p.summary}</Section>
      <Section title="Why it matters for us">{p.why_it_matters}</Section>
      <Section title="Discussion question" highlight>
        {p.discussion_question}
      </Section>
      <StatusRow post={p} members={members} />
      <Comments post={p} members={members} />
    </article>
  );
}
```

`web/pages/BoardPage.tsx`:

```tsx
import { Link } from "react-router";
import { useBoard } from "../api";
import { MemberChip } from "../components/MemberChip";
import { dayLabel } from "../format";
import { cellSymbol, statusLabel } from "../status";

export function BoardPage() {
  const board = useBoard();
  if (!board.data) return <p className="text-zinc-500">Loading…</p>;
  const { members, posts, counts } = board.data;
  return (
    <div className="space-y-4">
      <h1 className="text-lg font-semibold">Team board · last 7 days</h1>
      <div className="overflow-x-auto rounded-xl border border-zinc-200 bg-white dark:border-zinc-800 dark:bg-zinc-900">
        <table className="w-full text-sm">
          <thead>
            <tr className="text-xs text-zinc-500">
              <th className="p-3 text-left font-medium">Post</th>
              {members.map((m) => (
                <th key={m.id} className="p-2 font-medium">
                  <div className="flex flex-col items-center gap-1">
                    <MemberChip member={m} filled />
                    <span className="tabular-nums">
                      {counts[m.id]?.done ?? 0}/{counts[m.id]?.total ?? 0}
                    </span>
                  </div>
                </th>
              ))}
            </tr>
          </thead>
          <tbody>
            {posts.map((p) => (
              <tr key={p.id} className="border-t border-zinc-100 dark:border-zinc-800">
                <td className="p-3">
                  <Link to={`/posts/${p.id}`} className="font-medium hover:underline">
                    {p.title}
                  </Link>
                  <div className="text-xs text-zinc-500">{dayLabel(p.created_at)}</div>
                </td>
                {members.map((m) => (
                  <td key={m.id} className="p-2 text-center text-base" title={`${m.name}: ${statusLabel(p.status[m.id])}`}>
                    {cellSymbol(p.status[m.id])}
                  </td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>
      <p className="text-xs text-zinc-500">✓ commented · – read, nothing to add · ★ shared it · blank: not yet</p>
    </div>
  );
}
```

`web/pages/Archive.tsx`:

```tsx
import { useFeed, useMembers } from "../api";
import { PostList } from "../components/PostList";

export function Archive() {
  const feed = useFeed("archive");
  const members = useMembers().data ?? [];
  return (
    <div className="space-y-4">
      <div>
        <h1 className="text-lg font-semibold">Archive</h1>
        <p className="text-sm text-zinc-500">Older than 7 days. Still open for comments, not counted on the board.</p>
      </div>
      {feed.isPending ? (
        <p className="text-zinc-500">Loading…</p>
      ) : (
        <PostList posts={feed.data ?? []} members={members} empty="Nothing archived yet." />
      )}
    </div>
  );
}
```

- [ ] **Step 3: Add the routes**

In `web/App.tsx`, add the imports:

```tsx
import { Archive } from "./pages/Archive";
import { BoardPage } from "./pages/BoardPage";
import { PostPage } from "./pages/PostPage";
```

and replace the `<Routes>` block with:

```tsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/posts/:id" element={<PostPage />} />
  <Route path="/board" element={<BoardPage />} />
  <Route path="/archive" element={<Archive />} />
  <Route path="*" element={<Navigate to="/" replace />} />
</Routes>
```

- [ ] **Step 4: Typecheck and look at it**

Run: `bun test && bun run typecheck` → all pass, clean.

With `bun run dev` running (re-seed with `bun run seed` if needed), check in the browser:
- Clicking a card opens the post page: meta line, title, link opens in a new tab, tags, the three sections (discussion question highlighted), member status row, comments with names and times.
- `/board`: 14 rows, 6 member columns with `done/total`, ✓ – ★ symbols match the post pages.
- `/archive`: 4 older posts.
- `/posts/9999` shows "Post not found".

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add post page, team board and archive

Closes #<issue>"
```

---

### Task 9: Comments, "nothing to add" and Submit

**Files:**
- Create: `src/api/engagement.ts`, `web/components/MarkButton.tsx`, `web/pages/Submit.tsx`
- Modify: `src/api/posts.ts` (preview + publish), `src/api/app.ts` (mount engagement routes), `web/components/Comments.tsx` (full replacement), `web/pages/PostPage.tsx` (add MarkButton), `web/App.tsx` (add `/submit`)
- Test: `tests/api-write.test.ts`

**Interfaces:**
- Consumes: `addComment`, `getComment`, `updateComment`, `deleteComment`, `setMark`, `clearMark`, `getPost`, `insertPost`, `findPostIdByUrl`, `DuplicateUrlError`, `parsePostJson`, `memberGuard`, `readJson`, client `api.*`.
- Produces routes:
  - `POST /api/posts/preview` `{ text }` → 200 `{ post }` | 400 `{ error }` | 409 `{ error, existingId }`
  - `POST /api/posts` `{ text }` → 201 `Post` (origin `member`, `submitted_by` = cookie member) | 400 | 409 `{ error, existingId }`
  - `POST /api/posts/:id/comments` `{ body }` → 201 `Comment` | 400 | 404
  - `PATCH /api/comments/:id` `{ body }` → 200 `Comment` | 400 | 403 | 404
  - `DELETE /api/comments/:id` → 204 | 403 | 404
  - `PUT /api/posts/:id/mark` → 204 | 404; `DELETE /api/posts/:id/mark` → 204

- [ ] **Step 1: Write the failing tests**

`tests/api-write.test.ts`:

```ts
import { expect, test } from "bun:test";
import { createApi } from "../src/api/app";
import { config } from "../src/config";
import { insertPost } from "../src/db/posts";
import { EXAMPLE_POST } from "../src/schema/post";
import { samplePost, testDb } from "./helpers";

const NOW = new Date("2026-10-02T12:00:00Z");

function setup() {
  const db = testDb();
  const app = createApi({ db, config, now: () => NOW });
  const post = insertPost(db, samplePost(), { origin: "agent", submittedBy: null, runId: null, now: NOW });
  const call = (method: string, path: string, member: string, body?: unknown) =>
    app.request(path, {
      method,
      headers: { Cookie: `member=${member}` },
      body: body === undefined ? undefined : JSON.stringify(body),
    });
  const status = async (member: string) =>
    ((await (await call("GET", `/api/posts/${post.id}`, member)).json()) as { status: Record<string, string | null> }).status[member];
  return { db, app, post, call, status };
}

test("commenting makes you caught up; empty comments are rejected; missing post is 404", async () => {
  const { post, call, status } = setup();
  expect((await call("POST", `/api/posts/${post.id}/comments`, "ben", { body: "   " })).status).toBe(400);
  const res = await call("POST", `/api/posts/${post.id}/comments`, "ben", { body: " Useful! " });
  expect(res.status).toBe(201);
  expect(((await res.json()) as { body: string }).body).toBe("Useful!");
  expect(await status("ben")).toBe("commented");
  expect((await call("POST", "/api/posts/999/comments", "ben", { body: "x" })).status).toBe(404);
});

test("only the author can edit or delete a comment", async () => {
  const { post, call, status } = setup();
  const created = (await (await call("POST", `/api/posts/${post.id}/comments`, "ben", { body: "First" })).json()) as { id: number };
  expect((await call("PATCH", `/api/comments/${created.id}`, "henry", { body: "Hacked" })).status).toBe(403);
  expect((await call("DELETE", `/api/comments/${created.id}`, "henry")).status).toBe(403);
  const edited = await call("PATCH", `/api/comments/${created.id}`, "ben", { body: "First, edited" });
  expect(edited.status).toBe(200);
  expect(((await edited.json()) as { body: string }).body).toBe("First, edited");
  expect((await call("DELETE", `/api/comments/${created.id}`, "ben")).status).toBe(204);
  expect(await status("ben")).toBeNull();
  expect((await call("DELETE", `/api/comments/${created.id}`, "ben")).status).toBe(404);
});

test("nothing-to-add mark can be set and undone", async () => {
  const { post, call, status } = setup();
  expect((await call("PUT", `/api/posts/${post.id}/mark`, "pablo")).status).toBe(204);
  expect(await status("pablo")).toBe("nothing");
  expect((await call("DELETE", `/api/posts/${post.id}/mark`, "pablo")).status).toBe(204);
  expect(await status("pablo")).toBeNull();
  expect((await call("PUT", "/api/posts/999/mark", "pablo")).status).toBe(404);
});

test("preview validates pasted JSON without saving", async () => {
  const { call } = setup();
  const ok = await call("POST", "/api/posts/preview", "henry", { text: JSON.stringify(EXAMPLE_POST) });
  expect(ok.status).toBe(200);
  expect(((await ok.json()) as { post: unknown }).post).toEqual(EXAMPLE_POST);
  const bad = await call("POST", "/api/posts/preview", "henry", { text: "{nope" });
  expect(bad.status).toBe(400);
  const feed = (await (await call("GET", "/api/feed", "henry")).json()) as unknown[];
  expect(feed).toHaveLength(1);
});

test("publishing creates a member post you are caught up on; duplicates return 409 with the existing id", async () => {
  const { post, call, status } = setup();
  const res = await call("POST", "/api/posts", "henry", { text: "```json\n" + JSON.stringify(EXAMPLE_POST) + "\n```" });
  expect(res.status).toBe(201);
  const created = (await res.json()) as { id: number; origin: string; submitted_by: string };
  expect(created.origin).toBe("member");
  expect(created.submitted_by).toBe("henry");
  const dup = await call("POST", "/api/posts", "ben", { text: JSON.stringify({ ...EXAMPLE_POST, url: `${post.url}?utm_source=x` }) });
  expect(dup.status).toBe(409);
  expect(((await dup.json()) as { existingId: number }).existingId).toBe(post.id);
  const dupPreview = await call("POST", "/api/posts/preview", "ben", { text: JSON.stringify({ ...EXAMPLE_POST, url: post.url }) });
  expect(dupPreview.status).toBe(409);
  const detail = (await (await call("GET", `/api/posts/${created.id}`, "henry")).json()) as { status: Record<string, string | null> };
  expect(detail.status.henry).toBe("author");
  expect(await status("henry")).toBeNull(); // the agent post is still unread for henry
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `bun test tests/api-write.test.ts`
Expected: FAIL with 404s (routes do not exist yet).

- [ ] **Step 3: Implement the API**

`src/api/engagement.ts`:

```ts
import { Hono } from "hono";
import { z } from "zod";
import { addComment, clearMark, deleteComment, getComment, setMark, updateComment } from "../db/engagement";
import { getPost } from "../db/posts";
import { type ApiDeps, type Env, memberGuard, readJson } from "./common";

const CommentBody = z.object({ body: z.string().trim().min(1, "Comment is empty").max(5000) });

export function engagementRoutes({ db, config, now = () => new Date() }: ApiDeps) {
  const r = new Hono<Env>();
  const member = memberGuard(config);

  r.post("/posts/:id{[0-9]+}/comments", member, async (c) => {
    const postId = Number(c.req.param("id"));
    if (!getPost(db, postId)) return c.json({ error: "Post not found" }, 404);
    const parsed = CommentBody.safeParse(await readJson(c));
    if (!parsed.success) return c.json({ error: z.prettifyError(parsed.error) }, 400);
    return c.json(addComment(db, postId, c.get("memberId"), parsed.data.body, now()), 201);
  });

  r.patch("/comments/:id{[0-9]+}", member, async (c) => {
    const comment = getComment(db, Number(c.req.param("id")));
    if (!comment) return c.json({ error: "Comment not found" }, 404);
    if (comment.member_id !== c.get("memberId")) return c.json({ error: "You can only edit your own comments" }, 403);
    const parsed = CommentBody.safeParse(await readJson(c));
    if (!parsed.success) return c.json({ error: z.prettifyError(parsed.error) }, 400);
    return c.json(updateComment(db, comment.id, parsed.data.body, now()));
  });

  r.delete("/comments/:id{[0-9]+}", member, (c) => {
    const comment = getComment(db, Number(c.req.param("id")));
    if (!comment) return c.json({ error: "Comment not found" }, 404);
    if (comment.member_id !== c.get("memberId")) return c.json({ error: "You can only delete your own comments" }, 403);
    deleteComment(db, comment.id);
    return c.body(null, 204);
  });

  r.put("/posts/:id{[0-9]+}/mark", member, (c) => {
    const postId = Number(c.req.param("id"));
    if (!getPost(db, postId)) return c.json({ error: "Post not found" }, 404);
    setMark(db, postId, c.get("memberId"), now());
    return c.body(null, 204);
  });

  r.delete("/posts/:id{[0-9]+}/mark", member, (c) => {
    clearMark(db, Number(c.req.param("id")), c.get("memberId"));
    return c.body(null, 204);
  });

  return r;
}
```

In `src/api/posts.ts`, change the imports to:

```ts
import { Hono } from "hono";
import { DuplicateUrlError, findPostIdByUrl, insertPost } from "../db/posts";
import { getBoard, getFeed, getPostDetail } from "../feed";
import { parsePostJson, teammatePrompt } from "../schema/post";
import { type ApiDeps, type Env, memberGuard, readJson } from "./common";
```

and add these routes before `return r;`:

```ts
  r.post("/posts/preview", member, async (c) => {
    const { text } = await readJson(c);
    const parsed = parsePostJson(typeof text === "string" ? text : "");
    if (!parsed.ok) return c.json({ error: parsed.error }, 400);
    const existingId = findPostIdByUrl(db, parsed.value.url);
    if (existingId !== null) return c.json({ error: `Already posted as post #${existingId}`, existingId }, 409);
    return c.json({ post: parsed.value });
  });

  r.post("/posts", member, async (c) => {
    const { text } = await readJson(c);
    const parsed = parsePostJson(typeof text === "string" ? text : "");
    if (!parsed.ok) return c.json({ error: parsed.error }, 400);
    try {
      const post = insertPost(db, parsed.value, { origin: "member", submittedBy: c.get("memberId"), runId: null, now: now() });
      return c.json(post, 201);
    } catch (e) {
      if (e instanceof DuplicateUrlError) return c.json({ error: e.message, existingId: e.existingId }, 409);
      throw e;
    }
  });
```

In `src/api/app.ts`, add `import { engagementRoutes } from "./engagement";` and mount it after the post routes:

```ts
  app.route("/", engagementRoutes(deps));
```

Run: `bun test` → all pass.

- [ ] **Step 4: Implement the UI**

Replace `web/components/Comments.tsx` with:

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { type FormEvent, useState } from "react";
import type { Comment, Member, PostDetail } from "../../src/types";
import { api } from "../api";
import { dayLabel, timeLabel } from "../format";
import { useMe } from "../me";
import { MemberChip } from "./MemberChip";

const textareaClass =
  "w-full rounded-lg border border-zinc-300 bg-white p-2 text-sm outline-none focus:border-zinc-500 dark:border-zinc-700 dark:bg-zinc-900";
const buttonClass =
  "rounded-lg bg-zinc-900 px-3 py-1.5 text-sm font-medium text-white disabled:opacity-40 dark:bg-zinc-100 dark:text-zinc-900";

export function Comments({ post, members }: { post: PostDetail; members: Member[] }) {
  const me = useMe();
  return (
    <section className="space-y-3">
      <h2 className="text-sm font-semibold uppercase tracking-wide text-zinc-500">Comments ({post.comments.length})</h2>
      {post.comments.length === 0 && <p className="text-sm text-zinc-500">No comments yet. Be the first.</p>}
      <ul className="space-y-3">
        {post.comments.map((c) => (
          <CommentItem key={c.id} comment={c} member={members.find((m) => m.id === c.member_id)} mine={c.member_id === me} />
        ))}
      </ul>
      <CommentForm postId={post.id} />
    </section>
  );
}

function CommentItem({ comment, member, mine }: { comment: Comment; member?: Member; mine: boolean }) {
  const qc = useQueryClient();
  const [editing, setEditing] = useState(false);
  const [draft, setDraft] = useState(comment.body);
  const [confirmDelete, setConfirmDelete] = useState(false);
  const save = useMutation({
    mutationFn: () => api.editComment(comment.id, draft),
    onSuccess: () => {
      setEditing(false);
      return qc.invalidateQueries();
    },
  });
  const remove = useMutation({ mutationFn: () => api.deleteComment(comment.id), onSuccess: () => qc.invalidateQueries() });
  return (
    <li className="rounded-xl border border-zinc-200 bg-white p-3 dark:border-zinc-800 dark:bg-zinc-900">
      <div className="flex items-center gap-2 text-xs text-zinc-500">
        {member && <MemberChip member={member} filled />}
        <span className="font-medium text-zinc-800 dark:text-zinc-200">{member?.name ?? comment.member_id}</span>
        <span>
          {dayLabel(comment.created_at)} {timeLabel(comment.created_at)}
          {comment.updated_at !== comment.created_at && " · edited"}
        </span>
        {mine && !editing && (
          <span className="ml-auto flex gap-2">
            <button type="button" onClick={() => setEditing(true)} className="hover:underline">
              edit
            </button>
            {confirmDelete ? (
              <button type="button" onClick={() => remove.mutate()} className="font-medium text-rose-600 hover:underline">
                confirm delete
              </button>
            ) : (
              <button type="button" onClick={() => setConfirmDelete(true)} className="hover:underline">
                delete
              </button>
            )}
          </span>
        )}
      </div>
      {editing ? (
        <div className="mt-2 space-y-2">
          <textarea value={draft} onChange={(e) => setDraft(e.target.value)} rows={3} className={textareaClass} />
          <div className="flex justify-end gap-2">
            <button type="button" onClick={() => setEditing(false)} className="text-sm text-zinc-500">
              cancel
            </button>
            <button type="button" disabled={!draft.trim() || save.isPending} onClick={() => save.mutate()} className={buttonClass}>
              Save
            </button>
          </div>
        </div>
      ) : (
        <p className="mt-2 whitespace-pre-wrap text-sm">{comment.body}</p>
      )}
    </li>
  );
}

function CommentForm({ postId }: { postId: number }) {
  const qc = useQueryClient();
  const [body, setBody] = useState("");
  const add = useMutation({
    mutationFn: () => api.addComment(postId, body),
    onSuccess: () => {
      setBody("");
      return qc.invalidateQueries();
    },
  });
  const submit = (e: FormEvent) => {
    e.preventDefault();
    if (body.trim()) add.mutate();
  };
  return (
    <form onSubmit={submit} className="space-y-2">
      <textarea
        value={body}
        onChange={(e) => setBody(e.target.value)}
        onKeyDown={(e) => {
          if (e.key === "Enter" && (e.metaKey || e.ctrlKey)) e.currentTarget.form?.requestSubmit();
        }}
        rows={3}
        placeholder="What do you think? Would we use this? (⌘+Enter to send)"
        className={textareaClass}
      />
      {add.error && <p className="text-sm text-rose-600">{add.error.message}</p>}
      <div className="flex justify-end">
        <button type="submit" disabled={!body.trim() || add.isPending} className={buttonClass}>
          Comment
        </button>
      </div>
    </form>
  );
}
```

`web/components/MarkButton.tsx`:

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";
import type { PostDetail } from "../../src/types";
import { api } from "../api";
import { useMe } from "../me";

/** "Read, nothing to add". Hidden once you commented or if you shared the post. */
export function MarkButton({ post }: { post: PostDetail }) {
  const me = useMe();
  const qc = useQueryClient();
  const set = useMutation({ mutationFn: () => api.setMark(post.id), onSuccess: () => qc.invalidateQueries() });
  const clear = useMutation({ mutationFn: () => api.clearMark(post.id), onSuccess: () => qc.invalidateQueries() });
  const status = post.status[me];
  if (status === "commented" || status === "author") return null;
  if (status === "nothing") {
    return (
      <button type="button" onClick={() => clear.mutate()} className="text-sm text-zinc-500 hover:underline">
        ✓ Marked as read, nothing to add · undo
      </button>
    );
  }
  return (
    <button
      type="button"
      onClick={() => set.mutate()}
      disabled={set.isPending}
      className="rounded-lg border border-zinc-300 px-3 py-1.5 text-sm hover:bg-zinc-100 dark:border-zinc-700 dark:hover:bg-zinc-800"
    >
      Read, nothing to add
    </button>
  );
}
```

In `web/pages/PostPage.tsx`, add `import { MarkButton } from "../components/MarkButton";` and replace `<StatusRow post={p} members={members} />` with:

```tsx
      <div className="flex flex-wrap items-center justify-between gap-3">
        <StatusRow post={p} members={members} />
        <MarkButton post={p} />
      </div>
```

`web/pages/Submit.tsx`:

```tsx
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { useState } from "react";
import { Link, useNavigate } from "react-router";
import type { PostInput } from "../../src/schema/post";
import { ApiError, api } from "../api";

export function Submit() {
  const prompt = useQuery({ queryKey: ["submit-prompt"], queryFn: api.submitPrompt, staleTime: Infinity, refetchInterval: false });
  const [text, setText] = useState("");
  const [preview, setPreview] = useState<PostInput | null>(null);
  const [copied, setCopied] = useState(false);
  const navigate = useNavigate();
  const qc = useQueryClient();
  const check = useMutation({ mutationFn: () => api.previewPost(text), onSuccess: (r) => setPreview(r.post), onError: () => setPreview(null) });
  const publish = useMutation({
    mutationFn: () => api.publishPost(text),
    onSuccess: async (post) => {
      await qc.invalidateQueries();
      navigate(`/posts/${post.id}`);
    },
  });
  const error = publish.error ?? check.error;
  const existingId = error instanceof ApiError ? (error.body.existingId as number | undefined) : undefined;

  const copy = async () => {
    await navigator.clipboard.writeText(prompt.data?.prompt ?? "");
    setCopied(true);
    setTimeout(() => setCopied(false), 1500);
  };

  return (
    <div className="space-y-6">
      <div>
        <h1 className="text-lg font-semibold">Share something</h1>
        <p className="text-sm text-zinc-500">
          1. Copy the prompt into your AI and add the link. 2. Paste the JSON it gives back. 3. Preview and publish.
        </p>
      </div>
      <section className="space-y-2">
        <div className="flex items-center justify-between">
          <h2 className="text-sm font-semibold uppercase tracking-wide text-zinc-500">Prompt</h2>
          <button type="button" onClick={copy} className="rounded-lg border border-zinc-300 px-3 py-1 text-sm dark:border-zinc-700">
            {copied ? "Copied!" : "Copy prompt"}
          </button>
        </div>
        <pre className="max-h-64 overflow-auto rounded-xl bg-zinc-900 p-4 text-xs text-zinc-100">{prompt.data?.prompt ?? "Loading…"}</pre>
      </section>
      <section className="space-y-2">
        <h2 className="text-sm font-semibold uppercase tracking-wide text-zinc-500">Paste the JSON</h2>
        <textarea
          value={text}
          onChange={(e) => {
            setText(e.target.value);
            setPreview(null);
            check.reset();
            publish.reset();
          }}
          rows={10}
          placeholder='{ "title": "…", "url": "https://…", … }'
          className="w-full rounded-lg border border-zinc-300 bg-white p-2 font-mono text-xs dark:border-zinc-700 dark:bg-zinc-900"
        />
        {error && (
          <div className="whitespace-pre-wrap rounded-lg bg-rose-50 p-3 text-sm text-rose-700 dark:bg-rose-950/40 dark:text-rose-300">
            {error.message}
            {existingId !== undefined && (
              <>
                {" "}
                <Link to={`/posts/${existingId}`} className="underline">
                  Open it
                </Link>
              </>
            )}
          </div>
        )}
        <div className="flex justify-end gap-2">
          <button
            type="button"
            disabled={!text.trim() || check.isPending}
            onClick={() => check.mutate()}
            className="rounded-lg border border-zinc-300 px-3 py-1.5 text-sm disabled:opacity-40 dark:border-zinc-700"
          >
            Preview
          </button>
          <button
            type="button"
            disabled={!preview || publish.isPending}
            onClick={() => publish.mutate()}
            className="rounded-lg bg-zinc-900 px-3 py-1.5 text-sm font-medium text-white disabled:opacity-40 dark:bg-zinc-100 dark:text-zinc-900"
          >
            Publish
          </button>
        </div>
      </section>
      {preview && (
        <section className="space-y-2 rounded-xl border border-zinc-200 bg-white p-4 dark:border-zinc-800 dark:bg-zinc-900">
          <div className="text-xs text-zinc-500">
            Preview · {preview.source}
            {preview.tags.length > 0 && ` · ${preview.tags.map((t) => `#${t}`).join(" ")}`}
          </div>
          <h3 className="text-lg font-semibold">{preview.title}</h3>
          <p className="text-sm">{preview.summary}</p>
          <p className="text-sm text-zinc-600 dark:text-zinc-400">{preview.why_it_matters}</p>
          <p className="text-sm font-medium">💬 {preview.discussion_question}</p>
        </section>
      )}
    </div>
  );
}
```

In `web/App.tsx`, add `import { Submit } from "./pages/Submit";` and the route `<Route path="/submit" element={<Submit />} />` before the `*` route.

- [ ] **Step 5: Typecheck and look at it**

Run: `bun test && bun run typecheck` → all pass, clean.

With `bun run dev`, check in the browser:
- On a post you have not engaged with: "Read, nothing to add" marks it (chip fills, NEW disappears on home); "undo" reverts.
- Add a comment (button and ⌘+Enter); the mark button disappears; edit and two-step delete work on your own comments only; deleting your only comment makes the post unread again.
- `/submit`: copy the prompt, paste the example JSON from the prompt with a new URL, Preview shows the card, Publish opens the new post with a "found by <you>" badge. Pasting a seeded URL shows "Already posted as post #N" with an "Open it" link. Pasting garbage shows the JSON error.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: add comments, nothing-to-add marks and member submissions

Closes #<issue>"
```

---

### Task 10: Scheduler, runs log and Agent page (fake agent)

**Files:**
- Create: `src/agent/types.ts`, `src/agent/fake.ts`, `src/db/runs.ts`, `src/runs-view.ts`, `src/scheduler.ts`, `src/api/runs.ts`, `web/pages/Runs.tsx`
- Modify: `src/api/common.ts` (typed `scheduler`), `src/api/app.ts` (mount runs routes), `src/server.ts` (start scheduler), `web/App.tsx` (add `/runs`)
- Test: `tests/scheduler.test.ts`, `tests/api-runs.test.ts`

**Interfaces:**
- Consumes: `dueSlots`, `localNow`, `isWorkday`, `recencyHours`, `windowStart`, `PostInput`, `insertPost`, `recentPostRefs`, `Config`.
- Produces:
  - `src/agent/types.ts`: `type AgentContext = { today: string; slotTime: string | null; recencyHours: number; recentPosts: { title: string; url: string }[] }`; `type Agent = (ctx: AgentContext) => Promise<unknown>`
  - `fakeAgent: Agent` from `src/agent/fake.ts`
  - `src/db/runs.ts`: `startRun(db, r: { slotDate: string | null; slotTime: string | null; trigger: "schedule" | "manual"; now: Date }): number`; `finishRun(db, id, r: { status: "success" | "failed"; error?: string | null; postId?: number | null; now: Date }): void`; `failInterruptedRuns(db, now): number`; `slotHistory(db, slotDate): Map<string, SlotHistory>`; `listRecentRuns(db, limit?): Run[]`
  - `getRunsView(db, config, now): RunsView` from `src/runs-view.ts`
  - `createScheduler(deps: { db; agent: Agent; config: Config; now?: () => Date; log?: (msg: string) => void }): Scheduler` with `Scheduler = { tick(): Promise<void>; runNow(): Promise<void> | null; start(): () => void; isBusy(): boolean }`
  - Routes: `GET /api/runs` → `RunsView`; `POST /api/runs` → 202 `{ started: true }` | 409 `{ error }`

- [ ] **Step 1: Write the failing tests**

`tests/scheduler.test.ts`:

```ts
import { expect, test } from "bun:test";
import type { Agent, AgentContext } from "../src/agent/types";
import { config as baseConfig } from "../src/config";
import { startRun } from "../src/db/runs";
import { getFeed } from "../src/feed";
import { getRunsView } from "../src/runs-view";
import { createScheduler } from "../src/scheduler";
import { members, samplePost, testDb } from "./helpers";

const config = { ...baseConfig, retryAfterMs: 5 * 60_000, maxAttemptsPerSlot: 3 };

function clockAt(iso: string) {
  let t = new Date(iso).getTime();
  return { now: () => new Date(t), advance: (ms: number) => (t += ms) };
}

test("tick runs every due slot once, in order, and later runs see earlier posts", async () => {
  const db = testDb();
  const clock = clockAt("2026-09-30T13:30:00Z"); // Wed 14:30 London
  const seen: AgentContext[] = [];
  const agent: Agent = async (ctx) => {
    seen.push(ctx);
    return samplePost();
  };
  const s = createScheduler({ db, agent, config, now: clock.now });
  await s.tick();
  expect(seen.map((c) => c.slotTime)).toEqual(["10:00", "11:00", "14:00"]);
  expect(seen.map((c) => c.recentPosts.length)).toEqual([0, 1, 2]);
  expect(seen[0]!.today).toBe("2026-09-30");
  expect(seen[0]!.recencyHours).toBe(48);
  await s.tick();
  expect(seen).toHaveLength(3);
  expect(getFeed(db, members, "recent", clock.now(), 7).every((p) => p.origin === "agent" && p.run_id !== null)).toBe(true);
});

test("does nothing on weekends", async () => {
  const db = testDb();
  let calls = 0;
  const s = createScheduler({ db, agent: async () => (calls++, samplePost()), config, now: clockAt("2026-10-03T15:00:00Z").now });
  await s.tick();
  expect(calls).toBe(0);
  expect(getRunsView(db, config, new Date("2026-10-03T15:00:00Z")).slots.every((v) => v.state === "skipped")).toBe(true);
});

test("a failing slot is retried after 5 minutes, at most 3 attempts", async () => {
  const db = testDb();
  const clock = clockAt("2026-09-30T09:30:00Z"); // Wed 10:30 London, only 10:00 due
  let calls = 0;
  const agent: Agent = async () => {
    calls++;
    throw new Error("search failed");
  };
  const s = createScheduler({ db, agent, config, now: clock.now });
  await s.tick();
  await s.tick();
  expect(calls).toBe(1);
  for (let i = 0; i < 4; i++) {
    clock.advance(5 * 60_000);
    await s.tick();
  }
  expect(calls).toBe(3);
  const slot = getRunsView(db, config, clock.now()).slots[0]!;
  expect(slot).toMatchObject({ time: "10:00", state: "failed", attempts: 3, error: "search failed" });
});

test("invalid agent output fails the run with a readable error", async () => {
  const db = testDb();
  const clock = clockAt("2026-09-30T09:30:00Z");
  const s = createScheduler({ db, agent: async () => ({ title: "x" }), config, now: clock.now });
  await s.tick();
  const slot = getRunsView(db, config, clock.now()).slots[0]!;
  expect(slot.state).toBe("failed");
  expect(slot.error).toStartWith("Invalid agent output");
});

test("a duplicate URL fails the run", async () => {
  const db = testDb();
  const clock = clockAt("2026-09-30T10:30:00Z"); // 10:00 and 11:00 due
  const post = samplePost();
  const s = createScheduler({ db, agent: async () => post, config, now: clock.now });
  await s.tick();
  const view = getRunsView(db, config, clock.now());
  expect(view.slots.map((v) => v.state)).toEqual(["success", "failed", "pending", "pending", "pending"]);
  expect(view.slots[1]!.error).toContain("Already posted");
});

test("runs left running by a previous process are marked interrupted and retried", async () => {
  const db = testDb();
  const clock = clockAt("2026-09-30T09:30:00Z");
  startRun(db, { slotDate: "2026-09-30", slotTime: "10:00", trigger: "schedule", now: new Date("2026-09-30T09:00:00Z") });
  const s = createScheduler({ db, agent: async () => samplePost(), config, now: clock.now });
  expect(getRunsView(db, config, clock.now()).recent.at(-1)).toMatchObject({ status: "failed", error: "interrupted" });
  await s.tick();
  expect(getRunsView(db, config, clock.now()).slots[0]).toMatchObject({ state: "success", attempts: 2 });
});

test("runNow runs a manual run and refuses while busy; tick waits for it", async () => {
  const db = testDb();
  const clock = clockAt("2026-09-30T09:30:00Z");
  const gate = Promise.withResolvers<void>();
  let calls = 0;
  const agent: Agent = async () => {
    calls++;
    await gate.promise;
    return samplePost();
  };
  const s = createScheduler({ db, agent, config, now: clock.now });
  const run = s.runNow();
  expect(run).not.toBeNull();
  expect(s.isBusy()).toBe(true);
  expect(s.runNow()).toBeNull();
  await s.tick(); // skipped while busy
  expect(calls).toBe(1);
  gate.resolve();
  await run;
  expect(s.isBusy()).toBe(false);
  const recent = getRunsView(db, config, clock.now()).recent;
  expect(recent[0]).toMatchObject({ trigger: "manual", status: "success", slot_time: null });
});
```

`tests/api-runs.test.ts`:

```ts
import { expect, test } from "bun:test";
import { createApi } from "../src/api/app";
import { config } from "../src/config";
import { createScheduler } from "../src/scheduler";
import { samplePost, testDb } from "./helpers";

const NOW = new Date("2026-09-30T09:30:00Z");
const as = (member: string, method = "GET"): RequestInit => ({ method, headers: { Cookie: `member=${member}` } });

test("GET /api/runs shows today's slots; POST starts a manual run; second POST while busy is 409", async () => {
  const db = testDb();
  const gate = Promise.withResolvers<void>();
  const scheduler = createScheduler({
    db,
    config,
    now: () => NOW,
    agent: async () => {
      await gate.promise;
      return samplePost();
    },
  });
  const app = createApi({ db, config, now: () => NOW, scheduler });

  const view = (await (await app.request("/api/runs", as("ben"))).json()) as { today: string; slots: { time: string; state: string }[] };
  expect(view.today).toBe("2026-09-30");
  expect(view.slots.map((s) => s.state)).toEqual(["pending", "pending", "pending", "pending", "pending"]);

  expect((await app.request("/api/runs", as("ben", "POST"))).status).toBe(202);
  expect((await app.request("/api/runs", as("ben", "POST"))).status).toBe(409);
  gate.resolve();
  expect((await app.request("/api/runs")).status).toBe(401);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `bun test tests/scheduler.test.ts tests/api-runs.test.ts`
Expected: FAIL, modules not found.

- [ ] **Step 3: Implement storage, view and scheduler**

`src/agent/types.ts`:

```ts
export type AgentContext = {
  today: string; // YYYY-MM-DD in the configured timezone
  slotTime: string | null; // "HH:MM", or null for a manual run
  recencyHours: number;
  recentPosts: { title: string; url: string }[];
};

/** Finds one post. Returns raw output; the scheduler validates it with PostInput. */
export type Agent = (ctx: AgentContext) => Promise<unknown>;
```

`src/agent/fake.ts`:

```ts
import type { Agent } from "./types";

/** Development stand-in for the Claude agent: waits a moment and returns a unique placeholder post. */
export const fakeAgent: Agent = async (ctx) => {
  await Bun.sleep(1500);
  const id = crypto.randomUUID().slice(0, 8);
  return {
    title: `Fake agent post ${id} (${ctx.slotTime ?? "manual"})`,
    url: `https://example.com/fake/${id}`,
    source: "example.com",
    summary: "Placeholder produced by the fake agent so the scheduler can be tested without Claude.",
    why_it_matters: "Lets us exercise the full posting flow in development.",
    discussion_question: "Does the scheduler flow look right to you?",
    tags: ["fake"],
    published_at: ctx.today,
  };
};
```

`src/db/runs.ts`:

```ts
import type { Database } from "bun:sqlite";
import type { Run } from "../types";

const RUN_COLUMNS = "id, slot_date, slot_time, trigger, status, error, started_at, finished_at, post_id";

export function startRun(
  db: Database,
  r: { slotDate: string | null; slotTime: string | null; trigger: "schedule" | "manual"; now: Date },
): number {
  const row = db
    .query<{ id: number }, { slotDate: string | null; slotTime: string | null; trigger: string; at: string }>(
      `INSERT INTO runs (slot_date, slot_time, trigger, status, started_at)
       VALUES ($slotDate, $slotTime, $trigger, 'running', $at) RETURNING id`,
    )
    .get({ slotDate: r.slotDate, slotTime: r.slotTime, trigger: r.trigger, at: r.now.toISOString() });
  return row!.id;
}

export function finishRun(
  db: Database,
  id: number,
  r: { status: "success" | "failed"; error?: string | null; postId?: number | null; now: Date },
): void {
  db.query(
    "UPDATE runs SET status = $status, error = $error, post_id = $postId, finished_at = $at WHERE id = $id",
  ).run({ id, status: r.status, error: r.error ?? null, postId: r.postId ?? null, at: r.now.toISOString() });
}

/** Runs still marked running belong to a process that died; mark them failed so they can be retried. */
export function failInterruptedRuns(db: Database, now: Date): number {
  return db
    .query("UPDATE runs SET status = 'failed', error = 'interrupted', finished_at = $at WHERE status = 'running'")
    .run({ at: now.toISOString() }).changes;
}

export type SlotHistory = {
  attempts: number;
  success: boolean;
  running: boolean;
  postId: number | null;
  lastStartedAt: string;
  lastError: string | null;
};

export function slotHistory(db: Database, slotDate: string): Map<string, SlotHistory> {
  const rows = db
    .query<Run, { d: string }>(
      `SELECT ${RUN_COLUMNS} FROM runs WHERE trigger = 'schedule' AND slot_date = $d ORDER BY id`,
    )
    .all({ d: slotDate });
  const map = new Map<string, SlotHistory>();
  for (const run of rows) {
    if (!run.slot_time) continue;
    const h = map.get(run.slot_time) ?? {
      attempts: 0,
      success: false,
      running: false,
      postId: null,
      lastStartedAt: run.started_at,
      lastError: null,
    };
    h.attempts += 1;
    h.lastStartedAt = run.started_at;
    if (run.status === "success") {
      h.success = true;
      h.postId = run.post_id;
    }
    if (run.status === "running") h.running = true;
    if (run.status === "failed") h.lastError = run.error;
    map.set(run.slot_time, h);
  }
  return map;
}

export function listRecentRuns(db: Database, limit = 20): Run[] {
  return db.query<Run, { limit: number }>(`SELECT ${RUN_COLUMNS} FROM runs ORDER BY id DESC LIMIT $limit`).all({ limit });
}
```

`src/runs-view.ts`:

```ts
import type { Database } from "bun:sqlite";
import type { Config } from "./config";
import { listRecentRuns, slotHistory } from "./db/runs";
import { isWorkday, localNow } from "./domain/time";
import type { RunsView, SlotView } from "./types";

export function getRunsView(db: Database, config: Config, now: Date): RunsView {
  const local = localNow(now, config.timeZone);
  const workday = isWorkday(local);
  const history = slotHistory(db, local.date);
  const slots = config.slots.map((time): SlotView => {
    const h = history.get(time);
    const state: SlotView["state"] = !workday
      ? "skipped"
      : !h
        ? "pending"
        : h.success
          ? "success"
          : h.running
            ? "running"
            : "failed";
    return { time, state, attempts: h?.attempts ?? 0, post_id: h?.postId ?? null, error: h?.lastError ?? null };
  });
  return { today: local.date, isWorkday: workday, slots, recent: listRecentRuns(db, 20) };
}
```

`src/scheduler.ts`:

```ts
import type { Database } from "bun:sqlite";
import { z } from "zod";
import type { Agent } from "./agent/types";
import type { Config } from "./config";
import { insertPost, recentPostRefs } from "./db/posts";
import { failInterruptedRuns, finishRun, slotHistory, startRun } from "./db/runs";
import { dueSlots, localNow, recencyHours, windowStart } from "./domain/time";
import { PostInput } from "./schema/post";

export type Scheduler = {
  /** Run every due, unfinished slot of today, one at a time. No-op while another run is in progress. */
  tick(): Promise<void>;
  /** Start a manual run. Returns null if a run is already in progress. */
  runNow(): Promise<void> | null;
  /** Tick now and every minute. Returns a stop function. */
  start(): () => void;
  isBusy(): boolean;
};

const errorMessage = (e: unknown) =>
  e instanceof z.ZodError ? `Invalid agent output: ${z.prettifyError(e)}` : e instanceof Error ? e.message : String(e);

export function createScheduler(deps: {
  db: Database;
  agent: Agent;
  config: Config;
  now?: () => Date;
  log?: (msg: string) => void;
}): Scheduler {
  const { db, agent, config } = deps;
  const now = deps.now ?? (() => new Date());
  const log = deps.log ?? (() => {});
  let busy = false;

  const interrupted = failInterruptedRuns(db, now());
  if (interrupted) log(`marked ${interrupted} interrupted run(s) as failed`);

  async function exclusive(fn: () => Promise<void>): Promise<void> {
    busy = true;
    try {
      await fn();
    } finally {
      busy = false;
    }
  }

  function pendingSlots(at: Date) {
    const due = dueSlots(at, config.timeZone, config.slots);
    if (due.length === 0) return [];
    const history = slotHistory(db, due[0]!.date);
    return due.filter(({ time }) => {
      const h = history.get(time);
      if (!h) return true;
      if (h.success || h.running || h.attempts >= config.maxAttemptsPerSlot) return false;
      return at.getTime() - Date.parse(h.lastStartedAt) >= config.retryAfterMs;
    });
  }

  async function runOnce(slot: { date: string; time: string } | null) {
    const at = now();
    const runId = startRun(db, {
      slotDate: slot?.date ?? null,
      slotTime: slot?.time ?? null,
      trigger: slot ? "schedule" : "manual",
      now: at,
    });
    log(`run #${runId} started (${slot ? `${slot.date} ${slot.time}` : "manual"})`);
    try {
      const raw = await agent({
        today: localNow(at, config.timeZone).date,
        slotTime: slot?.time ?? null,
        recencyHours: recencyHours(at, config.timeZone),
        recentPosts: recentPostRefs(db, windowStart(at, 30)),
      });
      const input = PostInput.parse(raw);
      db.transaction(() => {
        const post = insertPost(db, input, { origin: "agent", submittedBy: null, runId, now: now() });
        finishRun(db, runId, { status: "success", postId: post.id, now: now() });
      })();
      log(`run #${runId} posted: ${input.title}`);
    } catch (e) {
      finishRun(db, runId, { status: "failed", error: errorMessage(e), now: now() });
      log(`run #${runId} failed: ${errorMessage(e)}`);
    }
  }

  async function tick() {
    if (busy) return;
    await exclusive(async () => {
      for (const slot of pendingSlots(now())) await runOnce(slot);
    });
  }

  return {
    tick,
    runNow() {
      if (busy) return null;
      return exclusive(() => runOnce(null));
    },
    start() {
      const run = () => tick().catch((e) => log(`tick failed: ${errorMessage(e)}`));
      void run();
      const timer = setInterval(run, 60_000);
      return () => clearInterval(timer);
    },
    isBusy: () => busy,
  };
}
```

Note: `exclusive` sets `busy = true` synchronously before its first `await`, so `runNow()` followed immediately by `isBusy()` returns `true`.

- [ ] **Step 4: Implement the API and wire the server**

In `src/api/common.ts`, add `import type { Scheduler } from "../scheduler";` and change the `scheduler` field of `ApiDeps` to:

```ts
  scheduler?: Scheduler;
```

`src/api/runs.ts`:

```ts
import { Hono } from "hono";
import { getRunsView } from "../runs-view";
import type { Scheduler } from "../scheduler";
import { type ApiDeps, type Env, memberGuard } from "./common";

export function runRoutes({ db, config, now = () => new Date() }: ApiDeps, scheduler: Scheduler) {
  const r = new Hono<Env>();
  const member = memberGuard(config);

  r.get("/runs", member, (c) => c.json(getRunsView(db, config, now())));

  r.post("/runs", member, (c) => {
    const run = scheduler.runNow();
    if (!run) return c.json({ error: "A run is already in progress" }, 409);
    run.catch(() => {}); // failures are recorded on the run itself
    return c.json({ started: true }, 202);
  });

  return r;
}
```

In `src/api/app.ts`, add `import { runRoutes } from "./runs";` and after the engagement routes:

```ts
  if (deps.scheduler) app.route("/", runRoutes(deps, deps.scheduler));
```

Replace `src/server.ts` with:

```ts
import index from "../web/index.html";
import { fakeAgent } from "./agent/fake";
import { createApi } from "./api/app";
import { config } from "./config";
import { openDb } from "./db/db";
import { createScheduler } from "./scheduler";

const db = openDb(config.dbPath, config.members);
const scheduler = createScheduler({
  db,
  agent: fakeAgent,
  config,
  log: (msg) => console.log(`[scheduler] ${msg}`),
});
const api = createApi({ db, config, scheduler });

const server = Bun.serve({
  port: config.port,
  development: process.env.NODE_ENV !== "production",
  routes: {
    "/api/*": api.fetch,
    "/*": index,
  },
});

if (config.schedulerEnabled) scheduler.start();
console.log(
  `Dev Sharing running at ${server.url} (db: ${config.dbPath}, agent: ${config.agent}, scheduler: ${config.schedulerEnabled ? "on" : "off"})`,
);
```

- [ ] **Step 5: Implement the Agent page**

`web/pages/Runs.tsx`:

```tsx
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { Link } from "react-router";
import type { SlotView } from "../../src/types";
import { api } from "../api";
import { dayLabel, timeLabel } from "../format";

const STATE_STYLE: Record<SlotView["state"], string> = {
  pending: "bg-zinc-100 text-zinc-600 dark:bg-zinc-800 dark:text-zinc-300",
  running: "bg-blue-100 text-blue-700 dark:bg-blue-950 dark:text-blue-300",
  success: "bg-emerald-100 text-emerald-700 dark:bg-emerald-950 dark:text-emerald-300",
  failed: "bg-rose-100 text-rose-700 dark:bg-rose-950 dark:text-rose-300",
  skipped: "bg-zinc-100 text-zinc-400 dark:bg-zinc-800 dark:text-zinc-500",
};

const duration = (start: string, end: string | null) =>
  end ? `${Math.round((Date.parse(end) - Date.parse(start)) / 1000)}s` : "…";

export function Runs() {
  const runs = useQuery({ queryKey: ["runs"], queryFn: api.runs, refetchInterval: 5_000 });
  const qc = useQueryClient();
  const runNow = useMutation({ mutationFn: api.runNow, onSettled: () => qc.invalidateQueries({ queryKey: ["runs"] }) });
  if (!runs.data) return <p className="text-zinc-500">Loading…</p>;
  const { today, isWorkday, slots, recent } = runs.data;
  const busy = recent.some((r) => r.status === "running");
  return (
    <div className="space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-lg font-semibold">Agent · {today}</h1>
        <button
          type="button"
          disabled={busy || runNow.isPending}
          onClick={() => runNow.mutate()}
          className="rounded-lg bg-zinc-900 px-3 py-1.5 text-sm font-medium text-white disabled:opacity-40 dark:bg-zinc-100 dark:text-zinc-900"
        >
          {busy ? "Running…" : "Run now"}
        </button>
      </div>
      {runNow.error && <p className="text-sm text-rose-600">{runNow.error.message}</p>}
      {!isWorkday && <p className="text-sm text-zinc-500">Weekend: no scheduled runs today.</p>}
      <ul className="grid gap-2 sm:grid-cols-5">
        {slots.map((s) => (
          <li key={s.time} className="rounded-xl border border-zinc-200 bg-white p-3 dark:border-zinc-800 dark:bg-zinc-900">
            <div className="font-mono text-lg">{s.time}</div>
            <span className={`mt-1 inline-block rounded-full px-2 py-0.5 text-xs ${STATE_STYLE[s.state]}`}>
              {s.state}
              {s.attempts > 1 && ` · ${s.attempts} tries`}
            </span>
            {s.post_id && (
              <Link to={`/posts/${s.post_id}`} className="mt-1 block text-xs text-blue-600 hover:underline dark:text-blue-400">
                view post
              </Link>
            )}
            {s.error && s.state !== "success" && (
              <p className="mt-1 line-clamp-3 text-xs text-rose-600" title={s.error}>
                {s.error}
              </p>
            )}
          </li>
        ))}
      </ul>
      <section className="space-y-2">
        <h2 className="text-sm font-semibold uppercase tracking-wide text-zinc-500">Recent runs</h2>
        <div className="overflow-x-auto rounded-xl border border-zinc-200 bg-white dark:border-zinc-800 dark:bg-zinc-900">
          <table className="w-full text-sm">
            <tbody>
              {recent.map((r) => (
                <tr key={r.id} className="border-t border-zinc-100 first:border-t-0 dark:border-zinc-800">
                  <td className="p-2 text-zinc-500">#{r.id}</td>
                  <td className="p-2 whitespace-nowrap">
                    {dayLabel(r.started_at)} {timeLabel(r.started_at)}
                  </td>
                  <td className="p-2">{r.slot_time ?? "manual"}</td>
                  <td className="p-2">
                    <span className={`rounded-full px-2 py-0.5 text-xs ${STATE_STYLE[r.status]}`}>{r.status}</span>
                  </td>
                  <td className="p-2 tabular-nums text-zinc-500">{duration(r.started_at, r.finished_at)}</td>
                  <td className="p-2 text-xs">
                    {r.post_id ? (
                      <Link to={`/posts/${r.post_id}`} className="text-blue-600 hover:underline dark:text-blue-400">
                        post #{r.post_id}
                      </Link>
                    ) : (
                      <span className="line-clamp-2 text-rose-600" title={r.error ?? ""}>
                        {r.error}
                      </span>
                    )}
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </section>
    </div>
  );
}
```

In `web/App.tsx`, add `import { Runs } from "./pages/Runs";` and the route `<Route path="/runs" element={<Runs />} />` before the `*` route.

- [ ] **Step 6: Run tests, typecheck and look at it**

Run: `bun test && bun run typecheck` → all pass, clean.

With `bun run dev`: on a weekday after 10:00 London time the scheduler immediately catches up the due slots with fake posts (server log shows `[scheduler] run #N posted: …`). `/runs` shows slot states, "Run now" adds a manual fake post within ~2 s, the button shows "Running…" meanwhile, and the new post appears on Home. Restarting the server while a run is in progress marks it `interrupted` and retries it 5 minutes later.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: add scheduler with catch-up and retries, runs log and Agent page

Closes #<issue>"
```

---

### Task 11: Real Claude agent

**Files:**
- Create: `src/agent/prompt.ts`, `src/agent/claude.ts`, `scripts/agent-smoke.ts`
- Modify: `src/server.ts` (choose agent from config)
- Test: `tests/agent.test.ts`

**Interfaces:**
- Consumes: `AgentContext`, `Agent`, `postJsonSchema`, `PostInput`, `config`.
- Produces:
  - `buildAgentPrompt(ctx: AgentContext): string`
  - `type SpawnArgs = { cmd: string[]; stdin: string; cwd: string; timeoutMs: number }`; `type SpawnResult = { exitCode: number | null; stdout: string; stderr: string; timedOut: boolean }`; `type Spawner = (a: SpawnArgs) => Promise<SpawnResult>`
  - `bunSpawner: Spawner`; `parseClaudeResult(stdout: string): unknown`; `claudeCommand(): string[]`; `createClaudeAgent(opts: { cwd: string; timeoutMs: number; model?: string; spawner?: Spawner }): Agent`

- [ ] **Step 1: Write the failing tests**

`tests/agent.test.ts`:

```ts
import { expect, test } from "bun:test";
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { bunSpawner, claudeCommand, createClaudeAgent, parseClaudeResult, type SpawnArgs } from "../src/agent/claude";
import { buildAgentPrompt } from "../src/agent/prompt";
import type { AgentContext } from "../src/agent/types";
import { EXAMPLE_POST, postJsonSchema } from "../src/schema/post";

const ctx: AgentContext = {
  today: "2026-09-28",
  slotTime: "10:00",
  recencyHours: 72,
  recentPosts: [{ title: "Old story", url: "https://example.com/old" }],
};
const result = (extra: Record<string, unknown>) =>
  JSON.stringify({ type: "result", subtype: "success", is_error: false, result: "", ...extra });

test("prompt includes the date, weekday, slot, recency, covered posts and the one-item rule", () => {
  const p = buildAgentPrompt(ctx);
  expect(p).toContain("2026-09-28 (Monday)");
  expect(p).toContain("slot 10:00");
  expect(p).toContain("last 72 hours");
  expect(p).toContain("- Old story — https://example.com/old");
  expect(p).toContain("exactly one");
  expect(p).toContain("open the url with WebFetch");
  expect(buildAgentPrompt({ ...ctx, recentPosts: [], slotTime: null })).toContain("- (nothing yet)");
});

test("claudeCommand isolates the run and never uses --bare", () => {
  const cmd = claudeCommand();
  expect(cmd.slice(0, 4)).toEqual(["claude", "-p", "--model", "claude-opus-5-5"]);
  const flag = (name: string) => cmd[cmd.indexOf(name) + 1];
  expect(flag("--tools")).toBe("WebSearch,WebFetch");
  expect(flag("--allowedTools")).toBe("WebSearch,WebFetch");
  expect(flag("--output-format")).toBe("json");
  expect(flag("--setting-sources")).toBe("");
  expect(cmd).toContain("--strict-mcp-config");
  expect(cmd).toContain("--no-session-persistence");
  expect(cmd).not.toContain("--bare");
  expect(JSON.parse(flag("--json-schema")!)).toEqual(postJsonSchema());
});

test("agent passes command, prompt and cwd to the spawner and returns structured_output", async () => {
  const cwd = mkdtempSync(join(tmpdir(), "ds-agent-"));
  let call: SpawnArgs | undefined;
  const agent = createClaudeAgent({
    cwd,
    timeoutMs: 1000,
    spawner: async (a) => {
      call = a;
      return { exitCode: 0, stdout: result({ structured_output: EXAMPLE_POST }), stderr: "", timedOut: false };
    },
  });
  expect(await agent(ctx)).toEqual(EXAMPLE_POST);
  expect(call!.cmd).toEqual(claudeCommand());
  expect(call!.stdin).toBe(buildAgentPrompt(ctx));
  expect(call!.cwd).toBe(cwd);
  expect(call!.timeoutMs).toBe(1000);
});

test("agent errors: timeout, non-zero exit, Claude error, missing output, non-JSON", async () => {
  const cwd = mkdtempSync(join(tmpdir(), "ds-agent-"));
  const agentWith = (r: Partial<{ exitCode: number | null; stdout: string; stderr: string; timedOut: boolean }>) =>
    createClaudeAgent({
      cwd,
      timeoutMs: 1000,
      spawner: async () => ({ exitCode: 0, stdout: "", stderr: "", timedOut: false, ...r }),
    });
  await expect(agentWith({ timedOut: true, exitCode: null })(ctx)).rejects.toThrow("timed out after 1s");
  await expect(agentWith({ exitCode: 1, stderr: "not logged in" })(ctx)).rejects.toThrow("not logged in");
  await expect(agentWith({ stdout: result({ subtype: "error_during_execution", is_error: true }) })(ctx)).rejects.toThrow(
    "error_during_execution",
  );
  await expect(agentWith({ stdout: result({}) })(ctx)).rejects.toThrow("no structured_output");
  await expect(agentWith({ stdout: "oops" })(ctx)).rejects.toThrow("not JSON");
});

test("parseClaudeResult returns structured_output", () => {
  expect(parseClaudeResult(result({ structured_output: { a: 1 } }))).toEqual({ a: 1 });
});

test("bunSpawner pipes stdin and kills on timeout", async () => {
  expect(await bunSpawner({ cmd: ["cat"], stdin: "hello", cwd: tmpdir(), timeoutMs: 5000 })).toEqual({
    exitCode: 0,
    stdout: "hello",
    stderr: "",
    timedOut: false,
  });
  const slow = await bunSpawner({ cmd: ["sleep", "5"], stdin: "", cwd: tmpdir(), timeoutMs: 100 });
  expect(slow.timedOut).toBe(true);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `bun test tests/agent.test.ts`
Expected: FAIL, modules not found.

- [ ] **Step 3: Implement**

`src/agent/prompt.ts`:

```ts
import type { AgentContext } from "./types";

export function buildAgentPrompt(ctx: AgentContext): string {
  const weekday = new Date(`${ctx.today}T12:00:00Z`).toLocaleDateString("en-GB", { weekday: "long", timeZone: "UTC" });
  const covered = ctx.recentPosts.length
    ? ctx.recentPosts.map((p) => `- ${p.title} — ${p.url}`).join("\n")
    : "- (nothing yet)";
  return `You are the news editor for a small team of software engineers at a startup.
Today is ${ctx.today} (${weekday})${ctx.slotTime ? `, slot ${ctx.slotTime} London time` : ""}.

Use web search to find the single best item to share with the team right now.

What we want:
- Software engineering, startups, and the big AI events everyone in tech is talking about.
- Dev tooling and workflows first: coding agents (Claude Code, Cursor, Codex and similar), MCP, new developer practices.
- A major model release or industry shift beats a minor tool update.

Skip:
- Listicles, hype pieces, crypto-AI.
- Research papers, unless the whole industry is talking about one.
- Roundups and newsletters. If a roundup mentions something good, find and link the primary source.

Rules:
- It must be published in the last ${ctx.recencyHours} hours.
- It must not repeat a story we already covered (list below), even from a different URL.
- url must be the primary source: official blog, release notes, repository, or the original reporting.
- Before answering, open the url with WebFetch and confirm it loads and matches the title.
- Return exactly one item: the best one available.

How to write the fields:
- summary: what it is, in 2-3 plain sentences.
- why_it_matters: why software engineers at a startup should care, in 1-3 sentences.
- discussion_question: one open question that invites each teammate to share their own take.
- tags: up to 3 short lowercase tags.
- published_at: the source's publish date as YYYY-MM-DD, or null if unknown.

Already covered (last 30 days):
${covered}
`;
}
```

`src/agent/claude.ts`:

```ts
import { mkdirSync } from "node:fs";
import { postJsonSchema } from "../schema/post";
import { buildAgentPrompt } from "./prompt";
import type { Agent } from "./types";

export type SpawnArgs = { cmd: string[]; stdin: string; cwd: string; timeoutMs: number };
export type SpawnResult = { exitCode: number | null; stdout: string; stderr: string; timedOut: boolean };
export type Spawner = (args: SpawnArgs) => Promise<SpawnResult>;

export const bunSpawner: Spawner = async ({ cmd, stdin, cwd, timeoutMs }) => {
  const proc = Bun.spawn({ cmd, cwd, stdin: new Blob([stdin]), stdout: "pipe", stderr: "pipe" });
  let timedOut = false;
  const timer = setTimeout(() => {
    timedOut = true;
    proc.kill();
  }, timeoutMs);
  const [stdout, stderr, exitCode] = await Promise.all([
    new Response(proc.stdout).text(),
    new Response(proc.stderr).text(),
    proc.exited,
  ]);
  clearTimeout(timer);
  return { exitCode: timedOut ? null : exitCode, stdout, stderr, timedOut };
};

/**
 * Headless Claude Code, isolated from the owner's personal setup:
 * no MCP servers, no user/project settings (so no plugins or hooks), only web tools.
 * Never add --bare: it switches auth to ANTHROPIC_API_KEY instead of the logged-in subscription.
 */
export function claudeCommand(model = "claude-opus-5-5"): string[] {
  return [
    "claude",
    "-p",
    "--model",
    model,
    "--tools",
    "WebSearch,WebFetch",
    "--allowedTools",
    "WebSearch,WebFetch",
    "--output-format",
    "json",
    "--json-schema",
    JSON.stringify(postJsonSchema()),
    "--no-session-persistence",
    "--setting-sources",
    "",
    "--strict-mcp-config",
  ];
}

export function parseClaudeResult(stdout: string): unknown {
  let data: { subtype?: string; is_error?: boolean; result?: unknown; structured_output?: unknown };
  try {
    data = JSON.parse(stdout);
  } catch {
    throw new Error(`Claude output was not JSON: ${stdout.slice(0, 200)}`);
  }
  if (data.subtype !== "success" || data.is_error) {
    throw new Error(`Claude run failed (${data.subtype ?? "unknown"}): ${String(data.result ?? "").slice(0, 300)}`);
  }
  if (data.structured_output == null) throw new Error("Claude returned no structured_output");
  return data.structured_output;
}

export function createClaudeAgent(opts: { cwd: string; timeoutMs: number; model?: string; spawner?: Spawner }): Agent {
  const spawner = opts.spawner ?? bunSpawner;
  return async (ctx) => {
    mkdirSync(opts.cwd, { recursive: true });
    const r = await spawner({ cmd: claudeCommand(opts.model), stdin: buildAgentPrompt(ctx), cwd: opts.cwd, timeoutMs: opts.timeoutMs });
    if (r.timedOut) throw new Error(`Claude timed out after ${Math.round(opts.timeoutMs / 1000)}s`);
    if (r.exitCode !== 0) {
      throw new Error(`Claude exited with code ${r.exitCode}: ${(r.stderr || r.stdout).trim().slice(0, 300)}`);
    }
    return parseClaudeResult(r.stdout);
  };
}
```

In `src/server.ts`, add `import { createClaudeAgent } from "./agent/claude";` and replace `agent: fakeAgent,` with:

```ts
  agent:
    config.agent === "fake"
      ? fakeAgent
      : createClaudeAgent({ cwd: "data/agent-cwd", timeoutMs: config.agentTimeoutMs }),
```

`scripts/agent-smoke.ts`:

```ts
// One real agent call with today's context, printed. Usage: bun scripts/agent-smoke.ts
import { createClaudeAgent } from "../src/agent/claude";
import { config } from "../src/config";
import { localNow, recencyHours } from "../src/domain/time";
import { PostInput } from "../src/schema/post";

const now = new Date();
const agent = createClaudeAgent({ cwd: "data/agent-cwd", timeoutMs: config.agentTimeoutMs });
const started = Date.now();
const raw = await agent({
  today: localNow(now, config.timeZone).date,
  slotTime: null,
  recencyHours: recencyHours(now, config.timeZone),
  recentPosts: [],
});
console.log(JSON.stringify(PostInput.parse(raw), null, 2));
console.log(`took ${Math.round((Date.now() - started) / 1000)}s`);
```

- [ ] **Step 4: Run tests and typecheck**

Run: `bun test && bun run typecheck`
Expected: all pass, clean.

- [ ] **Step 5: Real smoke test**

Run: `bun scripts/agent-smoke.ts`
Expected: a valid post about a real, recent AI/SWE story from a primary source, and `took Ns`. Record N in the commit message.

Check the link mechanically: `curl -sIL -o /dev/null -w '%{http_code}\n' <url>` → `200` (some sites answer 403 to curl; then open it in a browser). Confirm the page matches the title and is recent.

- [ ] **Step 6: Isolation check**

Run from `data/agent-cwd`:

```bash
cd data/agent-cwd && echo 'List the names of every skill and slash command available to you, then say whether your context contains any text injected by hooks or plugins (yes/no). Be brief.' | claude -p --model claude-opus-5-5 --tools "" --no-session-persistence --setting-sources "" --strict-mcp-config
```

Expected: no plugin skills listed (built-in slash commands are fine) and the answer "No". (Verified once on 2026-10-02 with Claude Code 2.1.287; re-run here to confirm.) If plugin skills or hook text appear, stop and find the flag that disables them before continuing.

- [ ] **Step 7: End-to-end through the app**

Run: `SCHEDULER=off DB_PATH=data/dev.db bun src/server.ts` (real agent, dev DB, no automatic slot runs), open `/runs`, press "Run now". Expected: state `running`, then within a few minutes a `success` row linking to a new agent post with all sections filled. Stop the server.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: add headless Claude Code agent with isolated config

Smoke run took <N>s.

Closes #<issue>"
```

---

### Task 12: Hosting, README and final check

**Files:**
- Create: `README.md`
- Modify: `docs/superpowers/specs/2026-10-02-dev-sharing-design.md` (record the ngrok findings in the Hosting section)

**Interfaces:**
- Consumes: everything.
- Produces: a README a teammate can follow; `bun start` + `ngrok http 3000` documented and verified.

- [ ] **Step 1: Verify production mode**

Run: `SCHEDULER=off bun start` (real DB and agent, but no automatic runs during this check; `caffeinate -i` keeps the Mac awake while it runs).
In another terminal:

```bash
curl -s localhost:3000/api/members | head -c 80; echo
CSS=$(curl -s localhost:3000/ | grep -oE 'href="[^"]+\.css"' | head -1 | cut -d'"' -f2); curl -s "localhost:3000$CSS" | grep -c 'rounded-xl'
pgrep -fl caffeinate
```

Expected: members JSON; a count ≥ 1 (Tailwind built in production); a `caffeinate -i bun src/server.ts` process.

- [ ] **Step 2: Verify ngrok**

With `SCHEDULER=off bun start` running:

```bash
ngrok http 3000 --log stdout --log-format logfmt | grep -m1 -o 'url=https://[^ ]*'
```

Open the printed URL in a browser that has never visited it. Record:
1. whether ngrok shows a warning/interstitial page before the app (and what it says),
2. stop ngrok, start it again: is the URL the same?

If the URL changes, check the ngrok dashboard (Domains) for a free static domain; if one exists, re-run with `ngrok http 3000 --url=<that-domain>` and confirm the URL is stable across restarts. Write the exact working command into the README.

- [ ] **Step 3: Write the README**

`README.md` (fill the two bracketed findings from Step 2 with what you observed):

````markdown
# Dev Sharing

A tiny web app for a small dev team to stay on the same page about AI and software engineering.

- An AI agent (headless Claude Code) posts **one** curated item at **10:00, 11:00, 14:00, 15:00 and 16:00** London time, Monday to Friday.
- Everyone reads it and **comments**, or marks **"read, nothing to add"**.
- The **board** shows who is caught up on the last 7 days. Older posts move to the archive.
- Teammates can share their own finds through **Submit**: copy the prompt into your own AI, paste back the JSON.

## Run it

Requirements: [Bun](https://bun.sh) ≥ 1.3, [Claude Code](https://claude.com/claude-code) logged in, [ngrok](https://ngrok.com) with an authtoken.

```bash
bun install
bun start            # http://localhost:3000, keeps the Mac awake while running
<ngrok command from Step 2>   # share the https URL with the team
```

Run `bun start` from a normal terminal (not a background service): Claude Code reads its login from the macOS keychain.
If the laptop sleeps, missed slots of the current day run when it wakes up.

ngrok notes: [warning page behaviour observed in Step 2] · [URL stability observed in Step 2]

## Develop

```bash
bun run seed   # fresh data/dev.db with fake posts
bun run dev    # dev DB + fake agent, auto-restart, hot reload
bun test
bun run typecheck
bun scripts/agent-smoke.ts   # one real agent call, printed
```

## Configure

- Members: `config/members.json` (name, 2-letter code, colour).
- Schedule, window and retries: `src/config.ts`.
- Environment: `PORT` (3000), `DB_PATH` (`data/dev-sharing.db`), `AGENT` (`claude` or `fake`), `SCHEDULER` (`off` disables automatic runs), `APP_TIME_ZONE` (`Europe/London`).

## How the agent runs

`claude -p` with Opus 5.5, web search and fetch only, JSON-schema output, and no user settings, plugins, hooks or MCP servers (`--setting-sources "" --strict-mcp-config`). Each run gets the posts from the last 30 days so it does not repeat a story. Failed runs retry up to 3 times, 5 minutes apart; see the **Agent** page.

Design: `docs/superpowers/specs/2026-10-02-dev-sharing-design.md`
````

- [ ] **Step 4: Record findings in the spec**

In the spec's Hosting section, replace the "**To verify in the hosting issue:** …" sentence with the observed facts (warning page yes/no, URL stable yes/no, working command).

- [ ] **Step 5: Final check**

Run: `bun test && bun run typecheck`
Expected: all tests pass, typecheck clean.

Grep for leaks: `git grep -nE 'ANTHROPIC_API_KEY=|authtoken:|sk-ant-' -- . ':!bun.lock'` → no matches. Also grep for the owner's employer and customer names (ask the owner; never write them into the repo) → no matches.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "docs: add README and record hosting findings

Closes #<issue>"
```
