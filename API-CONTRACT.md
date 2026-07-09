# FPVibe — API Federation Contract

This document defines the JSON API contract between FPVibe federation tools.
It is the binding spec for cross-tool communication. Each tool implements its
side; consumers degrade gracefully when endpoints are unreachable.

Status: Draft v1.0 · companion to ARCHITECTURE.md

---

## 1. Conventions

### Base URLs

Each tool is discovered via environment variables. All URLs in this document
are relative to the base URL.

| Tool | Env var | Default | Example |
|------|--------|---------|---------|
| fpv-inventory | `INVENTORY_URL` | (unset = standalone) | `http://fpv-inventory:8000` |
| flowchart | `SESSIONS_URL` | (unset = standalone) | `http://flowchart:3000` |
| fpvibe-spots | `SPOTS_URL` | (unset = standalone) | `http://fpvibe-spots:7000` |

When a URL env var is unset or empty, the consumer tool operates without that
federation link. No calls are made; fallback behavior activates.

### Content type

All responses are `application/json; charset=utf-8`.

### Errors

Standard HTTP status codes:
- `200` — success
- `404` — entity not found
- `500` — server error (consumer should retry or degrade)

Error body (when the server can produce one):
```json
{ "error": "Part not found" }
```

Consumers must handle missing/empty/unset gracefully. A `404` on a cross-tool
fetch is not a crash — it means the referenced entity no longer exists.

### No authentication

No auth headers, no tokens, no sessions. Tools are local-only behind Runtipi.

---

## 2. Health endpoints

Every tool must expose:

```
GET /api/health
```

Response:
```json
{
  "status": "ok",
  "version": "1.0.0",
  "name": "fpv-inventory"
}
```

The `name` field identifies which tool is responding (useful for debugging
misconfigured env vars). The `version` is the tool's package version, used to
detect incompatible API versions when debugging.

---

## 3. fpv-inventory API

### 3.1 GET /api/builds

Returns top-level flyable assemblies. A "build" is a part where `parent_id IS NULL`
and `type = "craft"`.

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `status` | string | Filter by status: `unused`, `in-use`, `broken`, `retired`, `lost` |
| `q` | string | Case-insensitive name search |

**Response (200):**
```json
[
  {
    "id": 5,
    "name": "LionBee",
    "type": "craft",
    "status": "in-use",
    "quantity": 1,
    "specs": "FC: BetaFPV F4 1S\nMotors: 0702 30000KV\nFrame: Meteor65",
    "notes": "Power loops still mushy on exit",
    "photo_path": "5-1719504000.jpg"
  },
  {
    "id": 7,
    "name": "Air65III",
    "type": "craft",
    "status": "in-use",
    "quantity": 1,
    "specs": "FC: Super BF F4 AIO\nMotors: 0802 27000KV\nFrame: Air65",
    "notes": "",
    "photo_path": null
  }
]
```

**Empty array when no builds exist.** Not an error.

### 3.2 GET /api/builds/:id

Returns a single build with its component tree (one level deep — the BOM).

**Path parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `id` | integer | Part ID of the build |

**Response (200):**
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
  "created_at": "2026-01-15 10:30:00",
  "updated_at": "2026-07-04 18:22:00",
  "children": [
    {
      "id": 8,
      "name": "0702 Motor",
      "type": "motor",
      "status": "in-use",
      "quantity": 4,
      "specs": null,
      "notes": null
    },
    {
      "id": 12,
      "name": "BetaFPV F4 1S AIO",
      "type": "fc",
      "status": "in-use",
      "quantity": 1,
      "specs": "16MB blackbox, 5V BEC",
      "notes": "Betaflight 4.5.0"
    }
  ]
}
```

**404 when build not found:**
```json
{ "error": "Build not found" }
```

### 3.3 GET /api/parts

Returns all parts, optionally filtered by type.

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `type` | string | Filter by type: `motor`, `fc`, `esc`, `vtx`, `frame`, `camera`, `antenna`, `battery`, `craft`, `other` |
| `status` | string | Filter by status |
| `parent_id` | integer | Filter by parent (0 = top-level only) |
| `q` | string | Case-insensitive name search |

**Response (200):**
```json
[
  {
    "id": 8,
    "name": "0702 Motor",
    "type": "motor",
    "status": "in-use",
    "quantity": 4,
    "parent_id": 5,
    "specs": null,
    "notes": null,
    "photo_path": null
  }
]
```

### 3.4 GET /api/parts/:id

Returns a single part with its history.

**Response (200):**
```json
{
  "id": 8,
  "name": "0702 Motor",
  "type": "motor",
  "status": "in-use",
  "quantity": 4,
  "specs": null,
  "notes": null,
  "photo_path": null,
  "parent_id": 5,
  "created_at": "2026-01-15 10:30:00",
  "updated_at": "2026-07-01 14:00:00",
  "history": [
    {
      "id": 1,
      "part_id": 8,
      "action": "created",
      "from_parent_id": null,
      "to_parent_id": 5,
      "old_status": null,
      "new_status": null,
      "quantity_delta": null,
      "notes": null,
      "created_at": "2026-01-15 10:30:00"
    },
    {
      "id": 5,
      "part_id": 8,
      "action": "updated",
      "from_parent_id": null,
      "to_parent_id": null,
      "old_status": "unused",
      "new_status": "in-use",
      "quantity_delta": 0,
      "notes": null,
      "created_at": "2026-01-20 09:00:00"
    }
  ]
}
```

---

## 4. flowchart API

### 4.1 GET /api/sessions

Returns flight sessions, most recent first.

**Query parameters:**

| Param | Type | Description |
|-------|------|-------------|
| `craft` | integer | Filter by inventory craft ID (cross-tool link) |
| `craft_name` | string | Filter by platform name (legacy freetext match) |
| `limit` | integer | Max results (default: 50, max: 200) |

**Response (200):**
```json
[
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
  },
  {
    "id": 41,
    "date": "2026-07-03",
    "location": "home-room-A",
    "platform": "Air65III",
    "craft_inventory_id": 7,
    "pack_count": 4,
    "crash_count": 1,
    "session_type": "blocked-drill",
    "notes": "Orbit lines tightening"
  }
]
```

**`craft_inventory_id`** is the cross-tool link — an integer referencing an
fpv-inventory part ID. It is nullable: sessions created without inventory
federation use the `platform` freetext field only.

### 4.2 GET /api/sessions/:id

Returns a full session with packs, trick attempts, crashes, and review.

**Response (200):**
```json
{
  "id": 42,
  "date": "2026-07-04",
  "time_start": "14:00",
  "time_end": "16:30",
  "location": "stoughton-field",
  "platform": "LionBee",
  "craft_inventory_id": 5,
  "weather": "5-8mph",
  "session_type": "interleaved",
  "notes": "Power loops improving",
  "packs": [
    {
      "id": 101,
      "pack_number": 1,
      "voltage_start": 4.35,
      "voltage_end": 3.50,
      "focus": "Power loops",
      "crashes": 1,
      "notes": ""
    }
  ],
  "trick_attempts": [
    {
      "id": 201,
      "trick_id": 3,
      "trick_name": "Power Loop",
      "attempts": 8,
      "landed": 5,
      "notes": "Mushy on exit"
    }
  ],
  "crashes": [
    {
      "id": 301,
      "pack_number": 1,
      "trick_id": 3,
      "trick_name": "Power Loop",
      "failure_type": "throttle",
      "root_cause": "pilot",
      "action_item": "Blip throttle earlier on exit"
    }
  ],
  "review": {
    "good_1": "Orbit lines tightening",
    "good_2": "Power loop commitment better",
    "good_3": "",
    "improve_1": "Exit the power loop cleaner",
    "improve_2": "",
    "improve_3": "",
    "fatigue": "good",
    "frustration": "mild",
    "stopped_before_bad": true,
    "tomorrow_focus": "Power loop exit timing"
  }
}
```

**404 when session not found:**
```json
{ "error": "Session not found" }
```

### 4.3 Existing endpoints (unchanged)

These flowchart endpoints already exist and remain as-is:

- `GET /api/health` — health check
- `GET /api/tricks` — all tricks (reference data)
- `GET /api/tricks/drills/all` — drill list
- `GET /api/tricks/resources/all` — learning resources
- `GET /api/tricks/schedule/all` — 20-week schedule
- `GET /api/progress/stats` — overall statistics
- `GET /api/progress/mastery` — trick mastery overview
- `GET /api/progress/gates` — phase gate checks
- `GET /api/equipment` — equipment inventory (flowchart-local)
- `GET /api/data/export` — full JSON export

### 4.4 Required addition: `craft_inventory_id` on sessions

flowchart's `sessions` table needs a new nullable integer column:

```sql
ALTER TABLE sessions ADD COLUMN craft_inventory_id INTEGER;
```

This column stores the fpv-inventory part ID when a session is linked to an
inventory build. When null, the session uses the legacy `platform` freetext
field. Migration is additive — existing sessions are unaffected.

The `GET /api/sessions` response already includes this field in the shape
above. The `GET /api/sessions/:id` response includes it too.

---

## 5. Degradation patterns

### 5.1 Consumer: flowchart fetching builds from inventory

```typescript
// In flowchart's session form — fetching the craft dropdown

const INVENTORY_URL = Deno.env.get("INVENTORY_URL") ?? "";

async function fetchBuilds(): Promise<Build[] | null> {
  if (!INVENTORY_URL) return null;  // federation disabled

  try {
    const res = await fetch(`${INVENTORY_URL}/api/builds`, {
      signal: AbortSignal.timeout(3000),  // 3s timeout — localhost should be instant
    });
    if (!res.ok) return null;
    const builds = await res.json();

    // Cache for offline fallback
    localStorage.setItem("fpvibe:builds", JSON.stringify(builds));
    return builds;
  } catch {
    // Network error, timeout, or JSON parse failure — degrade
    return null;
  }
}

function getCachedBuilds(): Build[] {
  try {
    return JSON.parse(localStorage.getItem("fpvibe:builds") ?? "[]");
  } catch {
    return [];
  }
}

// In the form:
const builds = await fetchBuilds();
const cachedBuilds = getCachedBuilds();

let status: "live" | "cached" | "offline";
let dropdownOptions: string[];

if (builds && builds.length >= 0) {
  status = "live";
  dropdownOptions = builds.map(b => b.name);
} else if (cachedBuilds.length > 0) {
  status = "cached";
  dropdownOptions = cachedBuilds.map(b => b.name);
} else {
  status = "offline";
  dropdownOptions = ["Air65III", "Mobula6", "Meteor75", "Rekon3", "5-inch", "sim-whoop", "sim-5inch", "custom"];
  // Fall back to existing static enum
}

// Show badge: "✓ inventory" / "⚠ cached (N min ago)" / "⚠ offline"
// Session is always creatable regardless of status
```

### 5.2 Consumer: inventory fetching sessions from flowchart

```typescript
// In fpv-inventory's build detail page — "last flown" section

const SESSIONS_URL = Deno.env.get("SESSIONS_URL") ?? "";

async function fetchRecentSessions(craftId: number): Promise<Session[] | null> {
  if (!SESSIONS_URL) return null;

  try {
    const res = await fetch(
      `${SESSIONS_URL}/api/sessions?craft=${craftId}&limit=5`,
      { signal: AbortSignal.timeout(3000) }
    );
    if (!res.ok) return null;
    return await res.json();
  } catch {
    return null;
  }
}

// In the build detail template:
const sessions = await fetchRecentSessions(part.id);
const lastFlownSection = sessions && sessions.length > 0
  ? renderSessionList(sessions)
  : "";  // empty string = section hidden entirely
```

### 5.3 Degradation contract

| Situation | Behavior |
|-----------|----------|
| Sibling URL env var unset | No calls made. Feature hidden or fallback used. |
| Sibling container down (connection refused) | Timeout after 3s. Use cache or fallback. Show "⚠ offline" badge. |
| Sibling returns 404 for specific entity | Reference is stale. Show "entity no longer exists" or hide reference. Do not crash. |
| Sibling returns 500 | Same as connection refused — degrade to cache/fallback. |
| Sibling returns valid but empty array | This is success with no data. Show "no builds found" / "no sessions found". Not an error. |
| Sibling returns malformed JSON | Treat as connection failure. Degrade. |

**Hard rule: a cross-tool fetch failure must never prevent a save.**
If flowchart can't reach inventory, the session is still created — it just
won't have a `craft_inventory_id` linked. If inventory can't reach flowchart,
the build detail page still renders — it just hides the "last flown" section.

---

## 6. Versioning

### 6.1 API versioning

Each tool's `/api/health` returns its `version`. The federation contract is
this document. When an endpoint's response shape changes in a breaking way:

1. Bump the tool version (semver).
2. Note the breaking change in the docs repo (`docs/CHANGELOG.md`).
3. Update this document with the new shape.

There is no automated version negotiation. For n=1, you'll notice breakage
when you use the feature. The `version` field in `/api/health` helps you
confirm which version is running when debugging.

### 6.2 Contract version

This document is versioned: `v1.0`. Future versions will add endpoints (additive)
or change response shapes (breaking). Additive changes don't require consumer
updates. Breaking changes are noted in the changelog.

---

## 7. Open API questions

1. **Should flowchart accept `craft_inventory_id` on POST /api/sessions?**
   Currently the session create endpoint accepts `platform` (string). Adding
   `craft_inventory_id` (integer, nullable) to the POST body is the natural
   extension. This is in the Phase 2 implementation scope.

2. **Should inventory expose stock-check aggregation via API?**
   `GET /api/stock` could return quantities aggregated by type/status. This
   would let flowchart show "motors: 12 available" when creating a session.
   Likely a Phase 4 addition.

3. **Should inventory expose a "create build from components" endpoint?**
   `POST /api/builds` accepting a name + array of part IDs to assemble as
   children. This supports the "from-the-bin builds" feature. Needs design
   — it's the first cross-tool write concern, but it stays within inventory
   (inventory writes to its own DB; flowchart just calls the API).

4. **Should there be a unified search across tools?**
   `GET /api/search?q=power+loop` across sessions, builds, tricks? Probably
   not worth it for n=1. Each tool's own search is sufficient.