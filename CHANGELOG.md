# FPVibe — Changelog

Breaking changes to the federation contract, and the decisions behind them.

The contract version is tracked in [API-CONTRACT.md §6.2](./API-CONTRACT.md).
Each tool's own version is its package version, returned by `/api/health`.

---

## v2.0 — Builds are first-class entities

**Date:** 2026-11-06
**Contract:** `API-CONTRACT.md` v1.0 → v2.0 (breaking)
**Tool version:** `fpv-inventory` 1.0.0 → 2.0.0
**Decision:** [FPVibe/docs#16](https://github.com/FPVibe/docs/issues/16)

### What changed

In v1.0, a build was a **part** with `type = "craft"`. The same `parts` table
held stock rows, installed rows, and assembly rows, and a build's components
were its children via `parent_id`. Relating stock to builds therefore required
matching rows on the **name string**, and "how many of this part do I own?"
could not be answered from a single row.

In v2.0, a build is its own entity with its own table and its own row. A
build's components live in a BOM join table (`build_parts`) that references
parts by id.

### Why

Three problems with the v1.0 model, in order of how much they cost:

1. **Allocation was a string join.** `allocation.ts` matched stock to
   installed rows on `(name, type)`. Renaming a motor silently dropped it out
   of its own allocation, and two different parts sharing a name were merged.
   The join is now on `part_id`.
2. **One table did three jobs.** Whether a row was stock, an installed copy,
   or an assembly was inferred from `type` and `parent_id`. Every consumer
   needed a `craft` special case.
3. **"How many do I have?" was unanswerable from a row.** `quantity` meant
   "on the shelf" for a stock row and "installed" for a child row, so the
   total owned was only available by summing rows across a name group.

The v1.0 model also had nowhere to store a BOM `role`, which was left as a
permanently-`null` field with an open question attached (§7.6).

### Breaking changes

| Endpoint | Change |
|----------|--------|
| `GET /api/builds` | Returns build objects (`class`, `created_at`, `updated_at`), not part objects with `type: "craft"`. |
| `GET /api/builds/:id` | Returns `components` in place of `children`. |
| `GET /api/builds/:id/bom` | `role` is now populated from the BOM join. |
| `GET /api/parts` | No `parent_id` field. `parent_id` query param removed. `type` no longer includes `craft`. |
| `GET /api/parts/:id` | No `parent_id` field. History entries use `from_build_id` / `to_build_id` in place of `from_parent_id` / `to_parent_id`. |
| `GET /api/parts/:id/allocation` | Computed from the BOM join rather than name matching. Response shape unchanged. |
| `GET /api/gear`, `/api/gear/:id` | Unchanged. |
| `GET /api/stock` | Unchanged in shape. `craft` no longer appears as a type. |

### Migration

Existing databases are migrated in place on startup:

- Each `type = "craft"` row becomes a row in `builds`.
- Each child of a craft becomes a `build_parts` row. The installed copy's
  `quantity` is folded back into the part's total, so a part's `quantity` is
  the total owned after migration.
- `parent_id` is dropped from `parts`; `craft` is dropped from the type set.

### Consumer impact

**None currently.** `flowchart` has no `INVENTORY_URL` client yet — the
federation link described in ARCHITECTURE.md §5.1 has not been implemented, so
no consumer reads `GET /api/builds` today. When that client is built, it must
target the v2.0 shape: a build dropdown needs `id` and `name` from
`GET /api/builds`, both of which are unchanged.

### Open questions closed

- **§7.6 — "Where does the BOM `role` field come from?"** Resolved: option (c).
  `role` is stored on the `build_parts` join row, set when a part is installed
  into a build. It can distinguish two instances of the same part type in one
  build (e.g. main and backup camera).
