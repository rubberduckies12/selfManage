# Personal Ops System — Architecture & Build Plan

> Solo-founder control center focused on speed, clarity, and AI-augmented execution.

---

## 1) North Star & Principles
- **Crystal visibility:** at-a-glance Command Deck for *today, week, pipeline, money*.
- **Frictionless capture:** 1-click/1-tap quick capture (text/voice), zero required fields.
- **Linked everything:** tasks ↔ projects ↔ tags ↔ money ↔ review notes.
- **Local-first feel:** optimistic UI, fast filters, keyboard shortcuts.
- **AI assists, not dictates:** summaries, planning prompts, and capture → structured tasks.

---

## 2) High-Level Architecture
```
┌──────────────────────────────────────────────────────────────────┐
│ React (CRA) + Tailwind                                           │
│  • Routes: /, /projects, /tasks, /money, /review                 │
│  • State: React Query (server cache) + Zustand (UI/ephemeral)    │
│  • Components: Deck widgets, Kanban, Tables, Editors             │
└──────────────▲───────────────────────────────┬───────────────────┘
               │                               │ REST/WS
               │                               │
      ┌────────┴────────┐                ┌──────┴─────────────────┐
      │ Node.js (Express│                │ Background Worker(s)   │
      │ / Fastify API)  │                │  • Schedulers (cron)   │
      │  • Auth (JWT)   │                │  • Alpaca sync         │
      │  • REST + WS    │                │  • Crypto balances     │
      │  • OpenAI proxy │                │  • Price snapshots     │
      └────────▲────────┘                │  • Weekly digest build │
               │                         └─────────▲──────────────┘
               │                                     │
         ┌─────┴───────┐                     ┌───────┴─────────┐
         │ MongoDB     │                     │ External APIs    │
         │  • Core data│                     │  • Alpaca (stocks)
         │  • Events   │                     │  • Exchange(s)   │
         │  • Caches   │                     │  • OpenAI (GPT/WS)
         └─────────────┘                     └──────────────────┘
```

**Why this split:** front-end stays snappy; API thin & predictable; workers isolate third‑party jitter. Use a shared `@types` package for DTOs.

---

## 3) Data Model (MongoDB Collections)
> All `_id` are ObjectId; `uid` is authenticated user id (single-user but future‑proofed). Use compound indexes for common filters.

### 3.1 Users (for completeness)
```
users {
  _id, email, name, avatarUrl, tz, currency, createdAt, updatedAt,
  integrations: { alpaca?: {...}, exchanges?: [...], openaiProxy?: {...} }
}
```
Indexes: `email` unique.

### 3.2 Projects
```
projects {
  _id, uid, name, code, description,
  status: 'active'|'paused'|'archived',
  area?: 'SaaS'|'Investing'|'Job'|'Personal'|string,
  priority: 1..5,
  dates: { start?: Date, due?: Date, archivedAt?: Date },
  metrics?: { roiEstimate?: Number, effort?: Number },
  tags: [string],
  counters: { openTasks: Number, blockedTasks: Number },
  createdAt, updatedAt
}
```
Indexes: `{uid:1,status:1,priority:-1}`, text index on `name, description, code`.

### 3.3 Tasks
```
tasks {
  _id, uid, projectId?, title, note?,
  status: 'inbox'|'next'|'doing'|'blocked'|'waiting'|'done'|'cancelled',
  priority: 'low'|'med'|'high'|'urgent',
  estimate?: Number, // hours
  actual?: Number,
  due?: Date, start?: Date,
  recurrence?: { rule: 'RRULE string', next: Date },
  tags: [string],
  links?: [{ type:'url'|'doc'|'commit'|'pr', url, label? }],
  checklist?: [{ _id, text, doneAt?: Date }],
  relations?: { parentId?: ObjectId, blockedBy?: [ObjectId] },
  capture?: { source:'quick'|'voice'|'ai'|'import', raw?: string },
  completedAt?, createdAt, updatedAt
}
```
Indexes: `{uid:1,status:1,due:1}`, `{uid:1,projectId:1,status:1}`, text index on `title,note,tags`.

### 3.4 Tags (optional helper)
```
tags { _id, uid, key, color?, createdAt }
```
Index: `{uid:1,key:1}` unique.

### 3.5 Reviews (Weekly/Daily notes & AI outputs)
```
reviews {
  _id, uid,
  type: 'weekly'|'daily',
  period: { weekISO: '2025-W37', start: Date, end: Date },
  inputs: { highlights: [string], lowlights:[string], notes?: string },
  metrics: { tasksDone: Number, tasksCreated: Number, focusPct?: Number,
             revenueWeek?: Number, pnlWeek?: Number },
  ai: { summary?: string, priorities?: [string], risks?: [string], prompts?: [string] },
  createdAt
}
```
Index: `{uid:1,'period.weekISO':1}` unique.

### 3.6 Money: Accounts & Transactions
```
accounts {
  _id, uid, type:'job'|'saas'|'brokerage'|'crypto'|'bank'|'other',
  name, provider?: 'alpaca'|'binance'|'coinbase'|'manual',
  external?: { id?: string, meta?: object },
  currency: 'GBP'|'USD'|'EUR'|string,
  visibility: 'dashboard'|'hidden',
  createdAt, updatedAt
}

transactions {
  _id, uid, accountId, ts: Date,
  type:'income'|'expense'|'transfer'|'dividend'|'interest'|'trade',
  amount: Number, currency,
  symbol?: string, qty?: Number, price?: Number, // for trades
  tags?: [string], note?, projectId?,
  source:'manual'|'alpaca'|'exchange'|'calc',
  createdAt, updatedAt
}
```
Indexes: `{uid:1,accountId:1,ts:-1}`, `{uid:1,type:1,ts:-1}`, `{uid:1,symbol:1,ts:-1}`.

### 3.7 Portfolio (Positions & Snapshots)
```
positions {
  _id, uid, accountId, symbol, assetType:'equity'|'crypto',
  qty: Number, avgPrice: Number, currency,
  last: { price: Number, ts: Date },
  createdAt, updatedAt
}

price_snapshots {
  _id, uid, symbol, assetType, ts: Date, price: Number, currency
}
```
Indexes: `{uid:1,accountId:1,symbol:1}` unique; snapshots `{uid:1,symbol:1,ts:-1}`.

### 3.8 Activity Events (audit & analytics)
```
events {
  _id, uid, ts: Date, actor: 'user'|'system'|'ai',
  type: 'task.created'|'task.done'|'project.created'|'txn.imported'|'review.generated'|string,
  ref: { collection:'tasks'|'projects'|'transactions'|'reviews', id: ObjectId },
  meta?: object
}
```
Index: `{uid:1,ts:-1}`.

### 3.9 Files (optional)
```
files { _id, uid, name, size, mime, storageKey, linkedTo?: {type:'task'|'review'|..., id} }
```

---

## 4) API Design (Express/Fastify)
**Auth:** email magic link or local; issue short-lived JWT + refresh. Single user now → still keep `uid` in queries.

### 4.1 REST Endpoints (core)
- **Projects**  
  `GET /api/projects?status=active&tag=...`  
  `POST /api/projects`  
  `PATCH /api/projects/:id`  
  `DELETE /api/projects/:id` (soft-delete → archived)

- **Tasks**  
  `GET /api/tasks?status=next,doing&projectId=&dueFrom=&dueTo=&q=&sort=due`  
  `POST /api/tasks`  
  `PATCH /api/tasks/:id`  
  `POST /api/tasks/:id/checklist` (append)  
  `POST /api/tasks/bulk` (reorder/status moves)  
  `POST /api/tasks/inbox/capture` (quick/voice → AI structuring)

- **Tags**  
  `GET /api/tags` `POST /api/tags`

- **Money**  
  `GET /api/accounts` `POST /api/accounts`  
  `GET /api/transactions?type=income&from=&to=&symbol=`  
  `POST /api/transactions` (manual)  
  `POST /api/import/alpaca` (on-demand sync)  
  `POST /api/import/crypto`  
  `GET /api/positions` `GET /api/valuation/today`

- **Reviews**  
  `POST /api/reviews/weekly/generate?week=2025-W37`  
  `GET /api/reviews/weekly/:weekISO`  
  `POST /api/reviews/notes` (append highlights/lowlights)

- **Events** `GET /api/events?type=&from=`

**Standards:** cursor pagination (`?cursor=`), `If-None-Match` ETags, 201/204 semantics.

---

## 5) Background Jobs
Use a separate worker process (BullMQ or node-cron) with a Redis queue.

- **Alpaca Sync (stocks):** every 15m on weekdays. Fetch positions, orders, and account activities → normalize to `transactions` & `positions`.
- **Crypto Sync:** hourly. Pull balances & fills from chosen exchange(s). Fallback: manual CSV import.
- **Price Snapshots:** every 15m market hours (equity) + every 1h (crypto). Update `positions.last` and write `price_snapshots` for charts.
- **Weekly Digest Builder:** Sunday 17:00 local. Precompute metrics (tasks done/created, revenue, P&L) → store in `reviews.metrics`.
- **AI Summaries:** after digest metrics, call GPT to produce `reviews.ai` content.

---

## 6) AI Integration (GPT + Whisper)
**Capture to Task:**
- Endpoint: `POST /api/tasks/inbox/capture` with `{ text | audio }`.
- Pipeline:
  1) (optional) Whisper → text
  2) GPT structured extraction → `{ title, project?, tags[], due?, priority?, checklist[] }`
  3) Create `task(status='inbox')` with `capture.source='voice'|'ai'` + store raw.

**Weekly Review Generation:**
- Inputs: `events` + `transactions` + task deltas + calendar (optional).
- Prompt skeleton:
```
SYSTEM: You are an execution coach. Output concise bullets.
USER DATA: { metrics JSON }
TASK LOG: { tasks done/created/blocked }
MONEY: { pnl, income by source }
REQUEST: 1) 5-bullet summary 2) Top 5 priorities for next week 3) Risks & mitigations 4) One focus theme.
```
- Output persisted to `reviews.ai`.

**Daily Planning (optional):** morning `GET /api/reviews/daily/generate` using "today's capacity" and due tasks.

---

## 7) Frontend: Routes, Layout, and Components
**Tech:** CRA + Tailwind + React Router. Data via React Query; UI state via Zustand. Keyboard shortcuts (Cmd+K, "q" for quick capture, "n" new task).

### 7.1 Command Deck `/`
Cards in responsive grid (2–4 columns):
- **Today Focus**: tasks with `status in (next,doing)` sorted by `priority,due` (inline complete/drag to Done).
- **Quick Capture**: input + mic button → `capture` endpoint.
- **Projects Radar**: list top active projects with open/blocked counts.
- **This Week Timeline**: due tasks & milestones.
- **Money Snapshot**: totals by account + sparkline of 30‑day net.
- **Signals**: blocked tasks, overdue, low momentum projects.
- **Weekly Review Banner**: if Sunday eve/Monday morning.

### 7.2 Projects `/projects`
- **Kanban**: columns = `inbox, next, doing, blocked, waiting, done` (drag‑drop). Swimlanes per project or a single project view.
- **List**: dense table with filters (project, tag, due, status) + saved views.
- **Project Drawer**: metrics, description, tags, quick add task, progress, last 10 events.

### 7.3 Tasks `/tasks`
- Power table: multi-select, bulk edit (status, due, tags), natural language quick-add ("tomorrow 3pm", "#saas @urgent").
- Smart filters: "Hotlist" (due ≤ 3 days OR priority ≥ high), "Backlog", "Waiting".

### 7.4 Money `/money`
- **Accounts** list with balances and visibility toggle.
- **Income** tab (transactions type income/dividend/interest) with source breakdown.
- **Portfolio** tab: positions with last price, P&L, allocation pie; line chart from `price_snapshots`.
- **Add Transaction** modal (manual income/expense/transfer).
- **Import**: Alpaca on-demand; crypto CSV.

### 7.5 Weekly Review `/review`
- Left: inputs (highlights, lowlights, numbers).  
- Right: AI summary, priorities, risks.  
- Button: "Commit plan" → creates next-week priority tasks in `next` with `projectId` & tags.

---

## 8) UX Details for Speed
- **Cmd+K Launcher:** quick nav + create anything.
- **Customizable columns** in Kanban & Tables (persist per user).
- **Inline edits** everywhere; debounce saves.
- **Optimistic mutations** with error toasts + rollback.
- **Accessible keyboard nav** (j/k select, x complete, e edit).

---

## 9) Suggested Tech Choices
- **Backend:** Fastify (perf) or Express + Zod for schema validation; Prisma (Mongo driver) or Mongoose.
- **Queues:** BullMQ + Redis.
- **Auth:** JWT (httpOnly cookie) + refresh; optional magic link.
- **OpenAI:** stream responses; keep prompts and outputs in `reviews` or `events` for traceability.
- **Charts:** Recharts.

---

## 10) Example Mongoose Schemas (trimmed)
```ts
// task.model.ts
const TaskSchema = new Schema({
  uid: { type: ObjectId, index: true, required: true },
  projectId: { type: ObjectId, ref: 'Project' },
  title: { type: String, required: true },
  note: String,
  status: { type: String, enum: ['inbox','next','doing','blocked','waiting','done','cancelled'], index: true, default: 'inbox' },
  priority: { type: String, enum: ['low','med','high','urgent'], index: true, default: 'med' },
  due: Date, start: Date,
  tags: [String],
  checklist: [{ _id:false, text:String, doneAt:Date }],
  relations: { parentId: ObjectId, blockedBy: [ObjectId] },
  capture: { source:String, raw:String },
  completedAt: Date
},{ timestamps:true });
TaskSchema.index({ uid:1, projectId:1, status:1 });
TaskSchema.index({ uid:1, status:1, due:1 });
export default model('Task', TaskSchema);
```

```ts
// review.generate.ts (pseudo)
const metrics = await buildWeeklyMetrics(uid, weekISO);
const prompt = templates.weeklySummary(metrics);
const ai = await openai.chat.completions.create({ model:'gpt-4o-mini', messages:[
  { role:'system', content:'You are an execution coach. Be concise.' },
  { role:'user', content: JSON.stringify(metrics) }
]});
await Reviews.updateOne({ uid, 'period.weekISO': weekISO }, { $set: { metrics, ai: parse(ai) }}, { upsert:true });
```

---

## 11) Initial DB Indexes
- `tasks`: `{uid:1,status:1,due:1}`, `{uid:1,projectId:1,status:1}`, text(`title,note,tags`).
- `projects`: `{uid:1,status:1,priority:-1}`.
- `transactions`: `{uid:1,accountId:1,ts:-1}`, `{uid:1,type:1,ts:-1}`.
- `positions`: `{uid:1,accountId:1,symbol:1}` unique.
- `reviews`: `{uid:1,'period.weekISO':1}` unique.

---

## 12) Security & Ops
- Secrets in `.env` + Vault in prod; never ship OpenAI key to client; route through server.
- Input validation everywhere (Zod), rate limit on capture & AI endpoints.
- Structured logging (pino) with request ids; error tracing (Sentry).
- Backups: nightly Mongo dump; test restores monthly.

---

## 13) Project Structure (Monorepo optional)
```
/ops-app
  /apps
    /web (CRA + Tailwind)
    /api (Express/Fastify)
    /worker (queues/cron)
  /packages
    /types (shared DTOs & enums)
    /ui (shared React components)
    /config (eslint, tsconfig)
```

---

## 14) Build Roadmap
**V0 (1–2 days)**
- Schemas for projects, tasks, accounts, transactions, reviews.
- Create/read/update for tasks & projects; basic Command Deck with Quick Capture.

**V1 (1–2 weeks)**
- Kanban + List views; saved filters.  
- Money: manual accounts & transactions; portfolio table.
- Weekly Review generator (metrics + GPT summary).

**V1.1**
- Alpaca import; crypto CSV import; price snapshots; dashboard KPIs.

**V1.2**
- Voice capture (Whisper); AI task structuring; daily planning.

**V2**
- Multi-device sync polish, performance passes, richer analytics.

---

## 15) Open Questions (nice-to-have later)
- Calendar integration (read-only) for deadlines in Command Deck?  
- SLA/due-time semantics and work-hours calendar?  
- Simple risk register per project?

---

## 16) Acceptance Criteria (MVP)
- Create task from keyboard, see it instantly on Command Deck, drag to Done.
- Add a project and assign tasks; see project counters update.
- Enter an income transaction and see Money Snapshot reflect it.
- Run Weekly Review → get AI summary & auto-created top 5 priorities.

---

**You’re ready to start scaffolding.** Start with the schemas & Task CRUD, then the Command Deck widgets, then the Weekly Review path. Keep it fun and *fast*. 🚀

