# Changelog

Notable changes to the FPVibe architecture and API contract documents.

## 2026-11-10 — API-CONTRACT v2.0: first-class builds in fpv-inventory

**Breaking change** to the fpv-inventory API (fpv-inventory#84, shipped in
fpv-inventory#85). Builds are their own entities; contract bumped to v2.0.

- **Builds leave the parts table.** A build is no longer a part with
  `type = "craft"`. `GET /api/builds` and `GET /api/builds/:id` return build
  entities (`id`, `name`, `status`, `class`, `notes`, `specs`, `photo_path`,
  timestamps). Build ids are preserved across the migration, so
  `craft_inventory_id` cross-tool links keep working (§4).
- **`components` replaces `children`** on `GET /api/builds/:id`. Each entry is a
  `build_parts` allocation row (`id`, `part_id`, `name`, `type`, `quantity`,
  `role`, `notes`).
- **Allocation is a join on `part_id`** (§3.3, §3.6). The name+type grouping
  and its stability caveat are gone; a part's `quantity` is the total owned and
  is conserved.
- **`role` is real.** The §7.6 open question is resolved — the join table
  stores it (nullable).
- **Parts have no `parent_id`**; the `parent_id` filter on `GET /api/parts`
  and the `craft` part type are gone (§3.4).
- **History columns renamed**: `from_parent_id`/`to_parent_id` →
  `from_build_id`/`to_build_id`. Legacy `action: "moved"` entries can persist
  in migrated databases (§3.5).
- Tool version bumped to `2.0.0` (§2.1).

## 2026-11-13 — API-CONTRACT v2.1: storage location on parts in fpv-inventory

**Additive change** (fpv-inventory#72, shipped in the location PR). Semver
minor: existing consumers are unaffected.

- Parts gain an optional `location` field (nullable free text) recording
  physical storage, e.g. "Drawer 2 — small parts" (§3.4).
- `GET /api/parts` gains a `location` query parameter: case-insensitive
  substring filter on that field (§3.4).
- Parts gain `DELETE /api/parts/:id`: removes a part with its history; 409
  when the part is allocated to a build (§3.5.1).
- Tool version bumped to `2.1.0` (§2.1).
