# C4 — Lock & transaction footprint (with real EXPLAIN at scale)

- **Rule C4:** "Hold locks and transactions only for what needs them."
- **PR:** onyxsecurity/onyx#13392 · **Ticket:** PRDCT-14240
- **HEAD:** `74bded8213` (`74bded82133c1ac5da9b5f3ff0d5b47f95e8cbfc`)
- **Date:** 2026-09-20
- **Environment:** Tilt kind cluster `kind-slate`, ns `onyxd`, platform Postgres via `127.0.0.1:5432`.
- **Tenant / scale seeded:** schema `onyx_security`; one DesktopAgent **device family** `app_definition_id = 'seed_endpoint_family'` seeded to **30,000 asset rows** (ids 1000001–1030000) plus 30,000 matching `asset_inventory` rows. 30,000 is deliberately **above both** the 25,000-row bulk-update chunk threshold (`_STATUS_UPDATE_CHUNK_ROWS`) and the 5,000-member reproject size-gate (`INVENTORY_GROUP_SIZE_THRESHOLD`), so every gate/branch is exercised, not a two-row toy case. Seed rows were removed after capture (tenant restored to 205 assets); the seed SQL below is fully reproducible.

This extends the already-proven C4 statement inventory + no-non-DB-work sweep with the **lock-footprint** for the transaction(s) the diff touches.

---

## 1. The statements (quoted at HEAD)

### 1a. `bulk_update_asset_status_by_app_definition_ids` — the hot-path status writer
`backend_python/src/common/repository/asset.py:2084`. One set-based `UPDATE ... SET status = CASE ... END, updated_at = now()` over the whole matched app-definition family, preceded by an **indexed pre-count** (existence + write-cost metric), and — past 25,000 matched rows — carved into ascending-id range chunks that **stay in one caller transaction**:

```python
# pre-count (indexed existence set; also the write-cost metric)
counts_stmt = select(Asset.app_definition_id, func.count()).where(
    Asset.app_definition_id.in_(app_def_ids)).group_by(Asset.app_definition_id)

status_case = case(*[(Asset.app_definition_id == app_def_id, status.value)
                     for app_def_id, status in updates], else_=Asset.status)

def range_update_stmt(lower, upper):
    stmt = update(Asset).where(Asset.app_definition_id.in_(matched_ids)).values(
        status=status_case, updated_at=func.now())
    if lower is not None: stmt = stmt.where(Asset.id > lower)
    if upper is not None: stmt = stmt.where(Asset.id <= upper)
    return stmt

if sum(counts[a] for a in matched_ids) <= chunk_rows:   # chunk_rows = 25_000
    await self.db_session.execute(range_update_stmt(None, None))
else:
    numbered = select(Asset.id, func.row_number().over(order_by=Asset.id).label("rn")) \
        .where(Asset.app_definition_id.in_(matched_ids)).subquery()
    boundary_stmt = select(numbered.c.id).where(numbered.c.rn % chunk_rows == 0).order_by(numbered.c.id)
    boundaries = [row.id for row in (await self.db_session.execute(boundary_stmt)).all()]
    lower = None
    for upper in [*boundaries, None]:
        await self.db_session.execute(range_update_stmt(lower, upper))
        lower = upper
await self.db_session.flush()
```

### 1b. `reproject_edited_scalars(group_keys=app_def_ids)` — in-transaction read-model refresh
`backend_python/src/common/repository/asset_inventory/service.py:1519`. **Size-gated**: groups at/over 5,000 members are dropped from the immediate arm and converge via the periodic projection sweep; only under-threshold groups run synchronously. The actual write is `build_scalar_update` → `build_block_update(SCALAR, GROUP_KEYS)` = a diff-on-write `UPDATE asset_inventory ... FROM (project(SCALAR, GROUP_KEYS)) block_proj WHERE asset_inventory.asset_id = block_proj.asset_id AND (<per-column IS DISTINCT FROM>) RETURNING asset_id`.

```python
if group_keys:
    counts = await self.group_edit_member_counts(session, group_keys)
    group_keys, deferred = _partition_new_groups_by_size(group_keys, counts, size_threshold)  # 5000
    if deferred:
        logger.info("Group scalar reproject deferred to the periodic projection sweep ...")
    if not group_keys:
        return 0
return await self.reproject_edited_blocks(session, blocks={ProjectionBlock.SCALAR},
                                          group_keys=group_keys, asset_ids=asset_ids)
```

### 1c. `bulk_update_access_control_rules` — the commit span the diff touches
`backend_python/src/crud_service/api/routes/access_control.py:161`. The **single request transaction** that composes both statements above (plus audit writes) and commits once. Ordered: split MCP vs non-MCP → MCP access-control upserts → **`bulk_update_asset_status_by_app_definition_ids(non_mcp_updates)`** → **`reproject_edited_scalars(db_session, group_keys=app_def_ids)`** → representative lookup for audit + one ASSET-UPDATED audit row per updated asset → **single commit**. Agent-issue recompute is fired **fire-and-forget after commit** in its own sessions (see 1d):

```python
non_mcp_results_dict = await asset_repo.bulk_update_asset_status_by_app_definition_ids(non_mcp_updates)  # :209
...
await inventory_read_model_repo.reproject_edited_scalars(db_session, group_keys=app_def_ids)             # :250
...
assets_by_app_def_id = await AssetRepository(db_session).get_by_app_definition_ids(app_def_ids)          # :268 (audit names)
```

### 1d. `get_asset_id_batch_by_app_definition_ids` — post-commit per-page keyset read
`backend_python/src/common/repository/asset.py:361`. One bounded keyset page, each resolved in its **own short session**, driven by `_recompute_agent_issues_after_sanction_change` (`access_control.py:826`), so no read transaction (and no xmin horizon) is held across the Temporal RPCs:

```python
stmt = (select(Asset.id)
        .where(Asset.app_definition_id.in_(app_definition_ids),
               Asset.is_deleted == False, Asset.id > after_id)
        .order_by(Asset.id).limit(batch_size))          # batch_size = 1000
# caller loop (access_control.py): each page in its OWN session, transaction closed before start_workflow
async with get_authenticated_db_session(session_maker) as session:
    asset_id_page = await AssetRepository(session).get_asset_id_batch_by_app_definition_ids(
        scope_app_def_ids, after_id=last_id, batch_size=AGENT_ISSUE_RECALC_BATCH_SIZE)
```

### 1e. Tenant migration UPDATE on `issue_definitions` (rev `a1b7c3e5d9f2`)
`backend_python/src/db_migrations/versions/tenant/sql/reenable_revived_unsanctioned_agent_issue.sql`. Bounded to **at most one row per tenant** (the single catalog definition, reached through the unique index on `identifier`), scoped to `is_enabled=false` only → idempotent:

```sql
UPDATE issue_definitions
SET is_enabled = true
WHERE identifier = 'unsanctioned-agent-deployed'
  AND is_enabled = false;
```

---

## 2. Seed evidence (real psql)

```
$ PGPASSWORD=postgres psql -h 127.0.0.1 -U postgres -d postgres
-- ids explicit at 1_000_000+ (assets.id is GENERATED BY DEFAULT AS IDENTITY, max existing id = 600)
INSERT INTO onyx_security.assets
  (id, name, unique_identifier, type, sub_type, status, app_definition_id, surface,
   is_deleted, total_risk_score, created_at, updated_at)
SELECT 1000000+g, 'Seed Endpoint Agent '||g, 'seed-endpoint::'||g,
       'AssetTypeDesktopAgent','AssetSubTypeAIAgent','AssetStatusDiscovered',
       'seed_endpoint_family','Desktop',false,10.0,now(),now()
FROM generate_series(1,30000) g;
-- matching read-model rows (asset_inventory requires only asset_id/is_representative/created_at/updated_at)
INSERT INTO onyx_security.asset_inventory
  (asset_id, is_representative, name, type, sub_type, surface, status, app_definition_id,
   total_risk_score, is_deleted, created_at, updated_at)
SELECT 1000000+g, (g=1), 'Seed Endpoint Agent '||g, 'AssetTypeDesktopAgent','AssetSubTypeAIAgent',
       'Desktop','AssetStatusDiscovered','seed_endpoint_family',10.0,false,now(),now()
FROM generate_series(1,30000) g;

       what       | count
------------------+-------
 assets_seeded    | 30000
 inventory_seeded | 30000

$ ... ANALYZE onyx_security.assets; ANALYZE onyx_security.asset_inventory;
 count |   min   |   max
-------+---------+---------
 30000 | 1000001 | 1030000
```

(`assets` has **no FK constraints**; `status` is a plain varchar with 15 indexes — 6 of them index `status`/`app_definition_id`/`group_key`, so a status write is non-HOT and must maintain those indexes. `STORED_ASSET_STATUSES = {SANCTIONED, UNSANCTIONED, DISCOVERED}`.)

Cleanup after capture: `DELETE FROM onyx_security.asset_inventory WHERE app_definition_id='seed_endpoint_family';` then the same on `assets` → tenant restored to 205 assets.

---

## 3. EXPLAIN (ANALYZE, BUFFERS) — hot paths at 30k scale

`SET search_path = onyx_security, public;` before each. Every `UPDATE` EXPLAIN was run inside `BEGIN … ROLLBACK` (real execution, not persisted).

### [1] Pre-count (indexed existence/cost count)
```
GroupAggregate  (cost=0.00..1286.57 rows=1) (actual time=8.076..8.077 rows=1 loops=1)
  Buffers: shared hit=834
  ->  Seq Scan on assets  (rows=30000) (actual time=0.021..6.744 rows=30000 loops=1)
        Filter: ((app_definition_id)::text = 'seed_endpoint_family'::text)
        Rows Removed by Filter: 205
 Execution Time: 8.116 ms
```
> Seq Scan here only because the seed family is ~99% of this tiny 30,205-row tenant. On a real multi-app tenant the planner uses `asset_app_definition_id` for a selective family.

### [2] Core bulk UPDATE — whole 30k family, single-statement form
```
Update on assets  (cost=0.00..1361.56 rows=0) (actual time=847.046..847.046 rows=0 loops=1)
  Buffers: shared hit=911567 read=6 dirtied=1308 written=1307
  ->  Seq Scan on assets  (rows=30000) (actual time=0.012..9.078 rows=30000 loops=1)
        Filter: ((app_definition_id)::text = 'seed_endpoint_family'::text)
        Rows Removed by Filter: 205
 Execution Time: 847.130 ms      -- 30,000 rows updated
```
> **~0.85 s / 30k rows** (~28 µs/row). `shared hit=911,567` buffers = the non-HOT index maintenance across the 15 indexes touching `status`. Consistent with the docstring's warm ~6 s / 209K on the largest tenants.

### [3] Chunk-boundary window (runs only when matched > 25,000)
```
Subquery Scan on numbered  (actual time=36.597..42.438 rows=1 loops=1)
  Filter: ((numbered.rn % '25000'::bigint) = 0)   Rows Removed by Filter: 29999
  ->  WindowAgg  (rows=30000) (actual time=0.111..40.464 rows=30000 loops=1)
        ->  Index Scan using base_applications_pkey on assets (rows=30000) (actual time=0.107..32.721)
 Execution Time: 42.465 ms
```
> Ascending-id **Index Scan** on the PK → deterministic boundary order (the invariant the code relies on for deterministic row-lock acquisition across writers).

### [4] One 25,000-row id-range chunk UPDATE (`id <= boundary`)
```
Update on assets  (actual time=799.315..799.316 rows=0 loops=1)
  Buffers: shared hit=849682 dirtied=1018 written=1018
  ->  Seq Scan on assets  (actual time=0.012..10.121 rows=25000 loops=1)
        Filter: ((id <= 1025000) AND ((app_definition_id)::text = 'seed_endpoint_family'::text))
 Execution Time: 799.341 ms      -- 25,000 rows updated
```
> Each chunk is one bounded `UPDATE` (≈0.8 s / 25k here) but all chunks share **one transaction** — so row locks accumulate (see §5).

### [5] `reproject_edited_scalars` → `build_scalar_update(GROUP_KEYS=['seed_endpoint_family'])`
Run via SQLAlchemy with the real asyncpg dialect + `schema_translate_map` (drives the true production statement), `BEGIN … ROLLBACK`:
```
Update on asset_inventory  (cost=1811914.50..3865286.88 rows=30000)
                           (actual time=2877.699..4254.376 rows=30000 loops=1)
  Buffers: shared hit=1352107 read=20 dirtied=2486 written=1719, temp read=1464 written=1464
  CTE base_assets
    ->  Hash Left Join (rows=30000) ... ->  Seq Scan on assets assets_1 (rows=30000)
         Filter: ((COALESCE(group_key, app_definition_id))::text = 'seed_endpoint_family' OR <model arm>)
    -- da_device_seen_at correlated subqueries: Aggregate ... (loops=30000) over device_sources/device_applications
  ->  Hash Left Join (rows=30000) ...   -- ~14 LEFT JOINs (agents, ai_providers, access_control x2,
        app_definitions, mcp_servers, desktop_agent_owner, employees x3, devices, host-device lateral)
  Filter: (asset_inventory.name IS DISTINCT FROM ... OR <36 per-column IS DISTINCT FROM guards>)
 Planning Time: 27.911 ms
 JIT: Functions: 349 ... Total 2687.049 ms
 Execution Time: 4537.250 ms      -- 30,000 rows written
```
> **~4.5 s** for a 30k-member group (of which ~2.7 s is JIT of 349 functions). This is exactly why `reproject_edited_scalars` is **size-gated at 5,000**: at this scale the group is **deferred to the periodic sweep** and does **not** run in the request transaction. The plan above is what the periodic projection sweep runs, not the synchronous request path at large scale.

### [6] Size-gate `group_edit_member_counts` (the count that decides defer)
```
HashAggregate  (actual time=11.530..11.531 rows=1 loops=1)
  Group Key: COALESCE(group_key, app_definition_id)
  ->  Seq Scan on assets  (rows=30000)
        Filter: ((app_definition_id)::text = 'seed_endpoint_family'
                 OR ((type)::text = 'AssetTypeHomegrown' AND (group_key)::text = 'seed_endpoint_family'))
 Execution Time: 11.567 ms
```
> Cheap (~12 ms). On a real tenant the `app_definition_id`/`group_key` index arms replace the seq scan; here the family dominates the table. This runs in the request path even when the reproject itself defers.

### [7] Keyset page `get_asset_id_batch_by_app_definition_ids` (after_id=0, batch=1000)
```
Limit  (cost=0.29..72.46 rows=1000) (actual time=0.103..0.399 rows=1000 loops=1)
  Buffers: shared hit=118
  ->  Index Scan using base_applications_pkey on assets (actual time=0.103..0.343 rows=1000)
        Index Cond: (id > 0)
        Filter: ((NOT is_deleted) AND (app_definition_id = 'seed_endpoint_family'))
 Execution Time: 0.438 ms
```
> **0.44 ms, 118 buffers** for a full 1,000-id page — peak memory one page, transaction closed before the Temporal RPC. Fleet size is unbounded but each read is bounded.

### [8] Tenant migration UPDATE on `issue_definitions`
```
Update on issue_definitions  (actual time=0.399..0.399 rows=0 loops=1)
  Buffers: shared read=2
  ->  Index Scan using issue_definitions_identifier_key on issue_definitions
        Index Cond: ((identifier)::text = 'unsanctioned-agent-deployed')
        Filter: (NOT is_enabled)   Rows Removed by Filter: 1
 Execution Time: 0.426 ms
```
> Single-row **Index Scan on the unique index** — no scan of the 92-row catalog. Matched **0 rows** here because this tenant's row is already `is_enabled=true` (the migration is idempotent / already applied). Worst case is 1 row.

---

## 4. Observed lock modes (real `pg_locks`, same backend, inside `BEGIN`)

### Bulk asset-status UPDATE (one 25k chunk) — `assets`
Table + **every index** taken at `RowExclusiveLock`; row-level exclusive on each matched tuple (one `transactionid` self-lock; per-tuple `xmax` on all 25,000 rows):
```
 relation | assets                            | RowExclusiveLock | t
 relation | base_applications_pkey            | RowExclusiveLock | t
 relation | asset_status / asset_status_id    | RowExclusiveLock | t
 relation | asset_app_definition_id           | RowExclusiveLock | t
 relation | asset_type_app_definition_id      | RowExclusiveLock | t
 relation | assets_effective_group_key ...    | RowExclusiveLock | t   (16 relations total)
 locktype=transactionid count=1   -- the txn's own id; matched rows carry its xmax as the row lock
```

### `reproject_edited_scalars` UPDATE — `asset_inventory` writer + read side
```
 asset_inventory            | RowExclusiveLock   -- writer (row-level exclusive on 30,000 updated rows)
 access_control             | AccessShareLock
 agents                     | AccessShareLock
 ai_providers               | AccessShareLock
 asset_connections          | AccessShareLock
 assets                     | AccessShareLock
 desktop_agent_applications | AccessShareLock
 desktop_agent_users        | AccessShareLock
 device_applications        | AccessShareLock
 device_sources             | AccessShareLock
 devices                    | AccessShareLock
 employees                  | AccessShareLock
 extension_blocked_domains  | AccessShareLock
 issue_detection_audit      | AccessShareLock
 mcp_servers                | AccessShareLock
 (+ public.app_definitions  | AccessShareLock)   -- public schema, outside the tenant-schema filter
```

### Migration UPDATE — `issue_definitions`
```
 issue_definitions                | RowExclusiveLock
 issue_definitions_identifier_key | RowExclusiveLock
 issue_definitions_pkey           | RowExclusiveLock
 -- at most one row-level lock
```

---

## 5. Lock-footprint table

Row estimates grounded in the EXPLAIN above; scale case = **one toggled DesktopAgent app-family spanning a large fleet** (docstring: 200K+ device rows on the largest tenants).

| Statement (txn) | Tables locked | Lock mode | Rows locked @ scale | Worst-case hold | Concurrent writers that block |
|---|---|---|---|---|---|
| **Pre-count** SELECT (1a) | `assets` (+ indexes read) | AccessShareLock | 0 rows write-locked | ms (§[1]: 8 ms/30k) | none (AccessShare conflicts only with AccessExclusive/DDL) |
| **Bulk status UPDATE** (1a) — the core writer | `assets` **table + all 15 indexes** | table `RowExclusiveLock`; **per-matched-row row-level exclusive** (tuple `xmax`) | = family size: **25,000 per chunk**, **all chunks' rows in ONE txn** → up to 200K+ | Whole `bulk_update_access_control_rules` txn: every chunk's row locks held from chunk 1 until the single commit. §[2]/[4]: ~0.8 s/25k warm → **~6 s / 209K** per docstring, colder in prod. | **scan-ingest `assets.status` upsert** blocks *only* on a device row it shares with this write (row-level `xmax` wait); table `RowExclusiveLock` does **not** conflict with another `RowExclusiveLock`, so upserts to *other* rows proceed. Ascending-id chunking makes lock acquisition order deterministic to avoid deadlock with concurrent writers. |
| **Chunk-boundary window** (1a) | `assets` (PK index scan) | AccessShareLock | 0 write | §[3]: 42 ms/30k | none |
| **`reproject_edited_scalars`** (1b) → `build_scalar_update` | **writes** `asset_inventory`; **reads** ~14 tenant tables + `public.app_definitions` | `asset_inventory` `RowExclusiveLock` + row-level on rewritten rows; reads `AccessShareLock` | up to **30k+ asset_inventory rows** (a group-scalar edit rewrites every member) | **Size-gated: at ≥5,000 members it is DEFERRED and does NOT run in the request txn** → contributes **zero** request-hold at large fleet scale (converges via the ~10-min sweep). Under 5,000 it runs in-txn: §[5] ~4.5 s/30k as the sweep plan (JIT-dominated). | **projection sweep** (full-tenant `build_upsert` on `asset_inventory`) contends row-level on overlapping rows — `build_upsert` orders `by asset_id ASC` to keep lock acquisition monotonic and avoid deadlock with this scoped write. No app-level status writer blocks (they write `assets`, not `asset_inventory`). |
| **Audit writes** (1c) | `asset` reps read (`get_by_app_definition_ids`) + ASSET-UPDATED audit rows | reads AccessShareLock; audit inserts `RowExclusiveLock` on the audit table | 1 rep/app read; 1 audit row per updated asset | Same commit as the bulk update → **extends the assets row-lock hold** by the audit lookup+insert time. | audit-table writers only (own table); does not widen the `assets` conflict set. |
| **Keyset page read** (1d) | `assets` (PK index scan) | AccessShareLock | 0 write | §[7]: **0.44 ms/1,000-id page**, each page in its **own short session, post-commit** → no xmin horizon held across Temporal RPCs | none — deliberately outside the sanction transaction |
| **Migration UPDATE** (1e) | `issue_definitions` + unique index + pkey | `RowExclusiveLock`; ≤1 row-level | **≤ 1 row** (unique-index reached, no scan) | §[8]: **0.43 ms**, one row, idempotent (0 rows when already enabled) | negligible — a single catalog row; a concurrent posture-policy toggle of the *same* identifier would row-wait, but a retired issue is not user-toggleable, so no real contender. |

### C4 conclusions
- **The transaction holds locks only for the write it needs.** The sanction commit span (`bulk_update_access_control_rules`) row-locks `assets` device rows + writes audit rows in one commit; it does **not** hold read transactions across Temporal RPCs (keyset recompute is post-commit, per-page, own sessions) and does **not** hold the fleet-sized read-model rewrite at scale (reproject is size-gated off ≥5,000 and deferred to the sweep).
- **Worst-case hold at large fleet scale = the bulk status write time only** (multi-chunk, single txn; ~6 s/209K warm per docstring), plus the small audit write — not the 4.5 s reproject, which is deliberately excluded from the request path at scale.
- **Contention is row-level, not table-level.** `RowExclusiveLock` (bulk update, reproject, audit) never blocks another `RowExclusiveLock`; scan-ingest and the projection sweep block only on the specific rows they share, and both write paths order by ascending id so acquisition is deterministic and deadlock-free.
- **The migration is a single indexed-row idempotent write** — no scan, no meaningful lock footprint.
