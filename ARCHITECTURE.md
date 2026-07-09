# FPVibe — Architecture Spec v1.0

Status: Draft · supersedes FPVIBE.md v0.3
Audience: implementing agents (Claude Code, opencode, Hermes) and future-Cori

---

## 1. TL;DR

FPVibe is a federation of single-purpose, self-hosted FPV tools. Each tool is an independent Docker container with its own SQLite database and web UI. Tools integrate via lightweight JSON APIs — never via a shared database, shared filesystem, or shared git substrate.

**The federation contract is an API schema, not a file format.**

- Each tool owns its data, its database, its deploy lifecycle.
- Each tool exposes `GET /api/<entity>` for entities other tools reference.
- Each tool discovers siblings via env vars (`INVENTORY_URL`, `SESSIONS_URL`).
- Cross-tool calls are read-only, optional, and always degradable.
- No tool writes to another tool's database. Ever.
- No auth, no multi-tenancy. Local-only behind Runtipi's reverse proxy.

---

## 2. What Changed from v0.3

| v0.3 premise | v1.0 reality |
|---|---|
| Git is the single source of truth (YAML substrate) | Each tool owns its own SQLite database, edited through its own web UI |
| Commit-on-save; tools are views over text files | Normal database writes; tools are the editors |
| `fpv:` URN scheme + resolver service | Direct HTTP URLs (`http://fpv-inventory:8000/api/builds/3`) |
| Forward-auth proxy / tailnet perimeter auth | No auth. Local-only behind Runtipi. |
| Shared theme tokens file (`fpvibe-theme.css`) | Optional. Each tool uses similar CSS. No shared file required for n=1. |
| Meta-compose orchestration | Per-tool Runtipi entries. No meta-compose needed. |
| Conformance checklist with 8 gates | Simplified to 5 gates (§6). |

**What survived:** federation over monolith, promote-don't-pre-build, independent deployability, graceful degradation, one-job-per-tool.

---

## 3. Current Repos and Maturity

| Repo | Stack | Maturity | Database |
|------|-------|----------|----------|
| `FPVibe/flowchart` | Hono/Node + vanilla JS PWA | Working app, CI, multi-agent dev | SQLite (`better-sqlite3`) |
| `FPVibe/fpv-inventory` | Deno + server-rendered HTML | Functional, basic | SQLite (Deno `sqlite`) |
| `FPVibe/fpv-tools` | Vanilla JS, static PWA | Mature, public on Pages | Client-side only (igow.db via sql.js, localStorage) |
| `FPVibe/fpvibe.github.io` | — | Empty | — |
| `FPVibe/docs` | — | This document | — |

**flowchart** is the most mature: full JSON REST API, 25KB seed data, session/progress/tricks/equipment/denver/plans/export-import. Known gap: offline writes fail (ISSUE-offline-first.md).

**fpv-inventory** is functional but has no JSON API — pure server-rendered HTML form POSTs. Hierarchical parts (parent_id = assemblies), status, quantity, type, specs, photos, full history log. README is still template boilerpaste.

**fpv-tools** is a static PWA with three tools (CLI Merge, Rate Profile, IGOW Reference). Client-side only. Does not participate in server-side federation.

---

## 4. Architecture

```
┌─────────────────────────────────────────────────────┐
│  Runtipi (Docker, local-only, no auth)              │
│                                                     │
│  ┌──────────────┐       ┌──────────────────┐       │
│  │ flowchart    │       │ fpv-inventory    │       │
│  │ :3000        │       │ :8000            │       │
│  │              │       │                  │       │
│  │ SQLite:      │       │ SQLite:          │       │
│  │  sessions    │       │  parts           │       │
│  │  packs       │       │  part_history    │       │
│  │  tricks      │       │                  │       │
│  │  crashes     │       │ JSON API (add):  │       │
│  │  reviews     │       │  GET /api/builds │       │
│  │  equipment   │       │  GET /api/parts  │       │
│  │              │       │                  │       │
│  │ JSON API:    │  ───► │                  │       │
│  │  GET /api/   │ fetch │                  │       │
│  │  sessions    │ builds│                  │       │
│  └──────────────┘       └──────────────────┘       │
│         │                       │                  │
│         │  optional reverse      │                  │
│         │  (last flown)          │                  │
│         ◄───────────────────────│                  │
│                                                     │
│  ┌──────────────────────────────────────────┐      │
│  │ fpv-tools (static, GitHub Pages)         │      │
│  │ Client-side only, no server federation   │      │
│  └──────────────────────────────────────────┘      │
│                                                     │
│  Discovery: INVENTORY_URL / SESSIONS_URL env vars  │
│  No shared database, no shared filesystem           │
│  No auth, no multi-tenancy                          │
└─────────────────────────────────────────────────────┘
```

### Data ownership

Each entity has exactly one authoritative owner. Other tools read via API; they never write.

| Entity | Owner | Authority |
|--------|-------|-----------|
| Session (flight log) | flowchart | flowchart is the only writer |
| Pack, Trick attempt, Crash, Review | flowchart | flowchart |
| Build / Craft (flyable assembly) | fpv-inventory | inventory is the only writer |
| Part (component, spare, consumable) | fpv-inventory | inventory |
| Gear (discrete serial/warranty asset) | fpv-inventory | inventory |
| Spot (flying location) | flowchart | flowchart (until promoted to own tool) |
| Training plan / Drill | flowchart | flowchart (until promoted) |

### Graceful degradation

Cross-tool calls are always read-only and always degradable. The pattern:

```
flowchart session form → craft dropdown:
  1. Try GET {INVENTORY_URL}/api/builds
  2. If success: populate dropdown, cache result in localStorage
  3. If INVENTORY_URL unset or fetch fails:
     a. Use cached builds from last successful fetch
     b. If no cache: fall back to a static enum or freetext input
  4. Show a badge: "✓ inventory" / "⚠ cached" / "⚠ offline"
  5. Session is always creatable regardless of inventory state
```

```
fpv-inventory build detail → "last flown" section:
  1. Try GET {SESSIONS_URL}/api/sessions?craft=<id>&limit=5
  2. If success: show recent sessions with dates
  3. If SESSIONS_URL unset or fetch fails: hide the section
  4. Build detail page is fully functional without it
```

**Rule: core functionality of a tool never breaks because a sibling is down.**
Cross-tool features degrade to cached data → fallback input → silent omission.
Never to a crash, never to a blocked save.

---

## 5. The Federation Contract

### 5.1 Discovery

Each tool reads sibling URLs from environment variables:

| Env var | Points to | Required? |
|---------|-----------|-----------|
| `INVENTORY_URL` | fpv-inventory base URL (e.g. `http://fpv-inventory:8000`) | No — unset = no federation |
| `SESSIONS_URL` | flowchart base URL (e.g. `http://flowchart:3000`) | No |
| `SPOTS_URL` | fpvibe-spots base URL (future) | No |

When unset, the tool operates standalone. Federation is opt-in per tool.

### 5.2 Required API endpoints

Each tool that owns an entity that other tools reference **must** expose a read-only JSON API:

**fpv-inventory must add:**

```
GET /api/builds
  → Top-level assemblies (parent_id IS NULL, type = "craft")
  → [{ id, name, status, type, quantity, specs, notes, photo_path }]

GET /api/builds/:id
  → Single build with its component tree
  → { id, name, status, ... , children: [{ id, name, type, quantity, status }] }

GET /api/parts
  → All parts (optional ?type= filter)
  → [{ id, name, status, type, quantity, parent_id }]

GET /api/parts/:id
  → Single part with history
  → { id, name, status, type, quantity, specs, notes, photo_path, parent_id, history: [...] }

GET /api/health
  → { status: "ok", version: "x.y.z" }
```

**flowchart already has:**

```
GET /api/sessions
  → Query: ?craft=<id>&limit=<n>
  → [{ id, date, location, platform, pack_count, crash_count }]

GET /api/sessions/:id
  → Full session with packs, tricks, crashes, review

GET /api/health
  → { status: "ok", version: "x.y.z" }
```

### 5.3 Link convention

Cross-tool links are plain HTTP URLs, not custom URN schemes:

- In flowchart UI: "Craft: [LionBee](http://fpv-inventory:8000/parts/5)" — direct link to inventory's build detail
- In inventory UI: "Last flown: [2026-07-04](http://flowchart:3000/#session=42)" — direct link to session detail

No resolver service. No `fpv:` scheme. The URL IS the link.

### 5.4 Versioning

Each tool's JSON API is versioned via the `version` field in `/api/health`.
The federation contract is this document. When an endpoint's response shape changes, bump the tool version and note the breaking change in the docs repo.

No automated compatibility checking. For n=1, breakage is caught when you use it.

---

## 6. Conformance — what makes a tool an FPVibe citizen

A tool is in the federation iff:

1. **One job.** Doing two things → it's two tools.
2. **Owns its data.** SQLite in its container. No shared database, no shared filesystem.
3. **Exposes a JSON read API** for entities other tools reference. (Stateless tools like fpv-tools: exempt.)
4. **Degrades gracefully** when siblings are down. Core function never depends on a sibling being reachable.
5. **Docker + Runtipi-compatible.** Single container, single port, non-root (UID 1000), `/api/health` endpoint, env-configured, volume for `/data`.

Optional but recommended: mobile-responsive, dark theme, PWA installable.

---

## 7. Entity Model

The entity model from v0.3 survives as a shared API schema reference. It's no longer a file format — it's what the JSON API responses look like.

### Build / Craft

A flyable aircraft. In inventory, it's a top-level part (parent_id IS NULL) with type = "craft" and child components (the BOM).

```json
{
  "id": 5,
  "name": "LionBee",
  "type": "craft",
  "status": "in-use",
  "quantity": 1,
  "specs": "FC: BetaFPV F4 1S\nMotors: 0702 30000KV\nFrame: Meteor65",
  "notes": "Power loops still mushy on exit",
  "photo_path": "5-1719504000.jpg",
  "parent_id": null,
  "children": [
    { "id": 8, "name": "0702 Motor", "type": "motor", "quantity": 4, "status": "in-use" },
    { "id": 12, "name": "BetaFPV F4 1S", "type": "fc", "quantity": 1, "status": "in-use" }
  ]
}
```

### Part

An inventory item — component, spare, or consumable. Same `parts` table, just not type = "craft" or not top-level.

### Session

A flight outing. Owned by flowchart.

```json
{
  "id": 42,
  "date": "2026-07-04",
  "location": "stoughton-field",
  "platform": "LionBee",
  "craft_inventory_id": 5,
  "pack_count": 6,
  "crash_count": 3,
  "session_type": "interleaved",
  "notes": "Power loops improving"
}
```

The `craft_inventory_id` field is the cross-tool link — it references an fpv-inventory part ID. It's optional; sessions created without inventory federation use the `platform` freetext field instead.

### Spot

A flying location. Currently a `location` enum in flowchart's session table. Promotes to its own tool only when airspace/compliance metadata outgrows a simple field.

### Training plan / Drill

Structured practice. Currently trick/drill seed data in flowchart. Promotes to its own tool only when progress tracking needs real structure.

---

## 8. Promotion

Spots and training start as fields/tables inside flowchart. Promote to their own citizens only when a concrete trigger fires:

- **Spot → fpvibe-spots:** when airspace/compliance metadata (LAANC, IAA/MySRS registration, EU constraints) outgrows a `location` enum field and wants real structure/validation.
- **Training → fpvibe-training:** when drills need genuine progress tracking, streaks, or scheduling rather than a reference checklist.

Promotion is cheap: add `SPOTS_URL` env var, stand up the new container, migrate the data. Existing sessions still reference the old location enum; new sessions can use the API. No link migration needed because there's no URN scheme — just update the env var.

---

## 9. Deploy shape (per tool)

```
Dockerfile:
  - single container, non-root (UID 1000)
  - SQLite file at /data/<tool>.db
  - /api/health endpoint

docker-compose.yml:
  - single service
  - one port
  - volume: /data
  - env: PORT, DB_PATH, INVENTORY_URL (or SESSIONS_URL), NODE_ENV/DENO_ENV

Runtipi:
  - config.json entry in app store
  - exposable: true
  - no_auth: true
```

### flowchart docker-compose.yml (current, add INVENTORY_URL)

```yaml
services:
  flowchart:
    build: .
    container_name: flowchart
    restart: unless-stopped
    ports:
      - "${APP_PORT:-3000}:3000"
    volumes:
      - ${APP_DATA_DIR:-./data}:/app/data
    environment:
      - NODE_ENV=production
      - DB_PATH=/app/data/flowchart.db
      - INVENTORY_URL=${INVENTORY_URL:-}
```

### fpv-inventory docker-compose.yml (updated)

```yaml
services:
  fpv-inventory:
    build: .
    container_name: fpv-inventory
    restart: unless-stopped
    ports:
      - "${APP_PORT:-8000}:8000"
    volumes:
      - ${APP_DATA_DIR:-./data}:/data
    environment:
      - DB_PATH=/data/fpv-inventory.db
      - PHOTOS_DIR=/data/photos
      - PORT=8000
      - SESSIONS_URL=${SESSIONS_URL:-}
```

---

## 10. Implementation priorities

### Phase 1: Inventory JSON API (unblocks federation)

1. Add JSON API routes to fpv-inventory alongside existing HTML routes
2. Add `GET /api/builds`, `GET /api/builds/:id`, `GET /api/parts`, `GET /api/parts/:id`, `GET /api/health`
3. Acceptance: `curl http://localhost:8000/api/builds` returns JSON array

### Phase 2: flowchart federation client

1. Add `INVENTORY_URL` env var support to flowchart
2. Replace hardcoded `platform` enum in session form with dynamic fetch from `{INVENTORY_URL}/api/builds`
3. Implement degradation: fetch → cache → fallback enum → freetext
4. Store `craft_inventory_id` (integer, nullable) on sessions that link to inventory builds
5. Acceptance: session form populates craft dropdown from inventory; if inventory is down, falls back gracefully

### Phase 3: Inventory reverse reference (optional)

1. Add `SESSIONS_URL` env var support to fpv-inventory
2. On build detail page, fetch recent sessions for that craft from flowchart
3. Acceptance: build detail shows "last flown" section; degrades to hidden when flowchart is down

### Phase 4: Inventory features (user-requested)

1. Stock check view: aggregate quantities by type/status across all parts
2. Repair plan entity: link a broken part to needed replacement parts + status tracking
3. From-the-bin builds: guided assembly flow — pick components from inventory, create a new craft with those parts as children
4. Acceptance: stock check shows "motors: 12 on hand, 4 allocated"; repair plan links broken motor to replacement; from-the-bin creates a new craft in inventory

### Phase 5: Documentation reconciliation

1. Replace FPVIBE.md v0.3 with this ARCHITECTURE.md in docs repo
2. Update fpvibe-context.md to reflect actual repo state (org exists, repos transferred, fpv-inventory has JSON API)
3. Fix fpv-inventory README (still template boilerpaste)
4. Migrate fpv-tools from cori.github.io to fpvibe.github.io

---

## 11. Guardrails (anti-patterns)

- **No shared database.** Each tool's SQLite is its own. Never mount the same file into two containers.
- **No shared filesystem / git substrate.** Data lives in databases, edited through web UIs.
- **No cross-tool writes.** Tools read siblings via API; they never write to a sibling's database.
- **No plugin framework.** No shared "core" library beyond conventions documented here.
- **No auth.** Local-only behind Runtipi. No login, no sessions, no user store.
- **No multi-tenancy.** Single user. No ownership fields, no ACLs.
- **No hard runtime dependencies.** Each tool boots and functions standalone. Federation is opt-in via env vars.
- **Keep tools single-job.** If one grows a second job, split it.
- **When in doubt, ship a small tool — not a bigger one.**

---

## 12. Open decisions

1. **Does flowchart's `equipment` table go away, or does it become session-event-only?** Inventory owns "what exists"; flowchart should own "what happened during a session." The equipment_log table (which already links to session_id) might become the canonical equipment-event record, while inventory owns the part itself. Decide before Phase 2.

2. **Gear (discrete serial/warranty assets):** Does inventory's `type` enum get a `gear` type, or does gear stay as a separate concept? Currently inventory has `type: "craft"` but no `gear` type. Add one, or keep gear as a top-level part with a `is_gear` flag?

3. **Tune profiles:** Where do Betaflight tune/rate profiles live? In inventory (as a field on a craft), in flowchart (as a session reference), or in a future fpvibe-tune tool? For now, tune data is in fpv-tools' rate-profile localStorage. No server-side home yet.

4. **fpv-tools → fpvibe.github.io migration:** The org Pages repo exists but is empty. fpv-tools still deploys to `cori.github.io/fpv-tools`. Worth migrating for namespace consistency, but breaks existing bookmarks/links.

5. **IGOW reference data:** Currently a static SQLite file in fpv-tools (`igow/igow.db`). If training promotes to its own tool, should this data move into flowchart's drill table, or stay client-side? Probably stay client-side — it's reference data, not session data.