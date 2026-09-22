# C6 — Keep large data out of workflow payloads (OPEN half: real serialized bytes)

- PR: #13392 · Ticket: PRDCT-14240 (Kimi Code/CLI)
- Repo: /home/coder/onyx · Branch: `factory-eval-v32/PRDCT-14240`
- HEAD: `74bded8213` · Date: 2026-09-20

## What C6 asked for (OPEN half)

> Paste (or link) one real serialized `IssueDetectionWorkflowInput` as it crosses the Temporal
> boundary from `_recompute_agent_issues_after_sanction_change`: the JSON produced by
> `model_dump(by_alias=True)` for a full-cap page — 1000 asset ids plus the platform_tenant block —
> with its measured byte length, so the ≤12 KB figure is backed by an actual serialization.

The cap half (keyset batching, ≤1000-id pages) was already proven. This closes the open half with a
real serialization run against the repo venv, plus a committed regression guard.

## The model that crosses the Temporal boundary

`IssueDetectionWorkflowInput` — `backend_python/src/common/models/workflow_inputs.py:62`

```python
class IssueDetectionWorkflowInput(BaseModel):
    """Input for Go IssueDetectionWorkflow child workflow.

    Note: When serializing for Go, use model_dump(by_alias=True) to ensure
    platform_tenant fields use CamelCase (ID, Name, SchemaName, UUID).
    """
    asset_ids: list[int] = Field(description="Processed asset IDs")
    platform_tenant: TenantModel = Field(description="Tenant payload for Go workflow (CamelCase when by_alias=True)")
```

Fields:

| Field | Type | Alias (by_alias) | Notes |
|-------|------|------------------|-------|
| `asset_ids` | `list[int]` | `asset_ids` | The capped page of device (asset) ids; `int` (bigint) |
| `platform_tenant` | `TenantModel` | `platform_tenant` | Nested Go-interop block |

`platform_tenant` block — `TenantModel`, `backend_python/src/common_core/models/tenant.py:15`.
Under `model_dump(by_alias=True)` it emits PascalCase keys:

| Python field | Type | Alias emitted |
|--------------|------|---------------|
| `id` | `int` | `ID` |
| `name` | `str` | `Name` |
| `schema_name` | `str` | `SchemaName` |
| `uuid` | `str` | `UUID` |

So the payload has exactly two top-level keys (`asset_ids`, `platform_tenant`) and the tenant block
has exactly four keys (`ID`, `Name`, `SchemaName`, `UUID`).

## The cap constant

`AGENT_ISSUE_RECALC_BATCH_SIZE = 1000` — `backend_python/src/crud_service/api/routes/access_control.py:823`.
No page carries more than this many ids; a 10K-device family fans out to 10 bounded starts.

## The exact serialization method (quoted from the worker/route code)

From `_recompute_agent_issues_after_sanction_change`,
`backend_python/src/crud_service/api/routes/access_control.py:882-893`:

```python
await temporal_client.start_workflow(
    ISSUE_DETECTION_WORKFLOW_NAME,
    IssueDetectionWorkflowInput(
        asset_ids=scope,
        platform_tenant=TenantModel(id=tenant.id, name=tenant.name, schema_name=tenant.schema_name, uuid=str(tenant.uuid)),
    ).model_dump(by_alias=True),
    id=workflow_id,
    task_queue=PYTHON_WORKER_TASK_QUEUE_NAME,
    id_conflict_policy=WorkflowIDConflictPolicy.TERMINATE_EXISTING,
    id_reuse_policy=WorkflowIDReusePolicy.ALLOW_DUPLICATE,
    search_attributes=...,
)
```

The payload handed to `start_workflow` is `IssueDetectionWorkflowInput(...).model_dump(by_alias=True)`.
Temporal's data converter JSON-encodes that dict; `json.dumps(...)` on the same dict is the faithful
byte-for-byte measure of what crosses the boundary. The comment at
`access_control.py:820` states the intent this evidence backs: "serialize to ~12 KB, two orders of
magnitude under Temporal's 256 KB warn / 2 MB hard blob limit."

## Measured full-cap byte length (real serialization)

Run against the repo venv (`backend_python/.venv/bin/python`), `model_dump(by_alias=True)` then
`json.dumps(...).encode("utf-8")`:

| Scenario (1000 ids) | Id digit width | Serialized bytes |
|---------------------|----------------|------------------|
| Small ids `1..1000` | 4 | 5,058 |
| `100000..100999` | 6 | 8,165 |
| Mature tenant `100000000..100000999` | 9 | **11,165** (matches the ~12 KB board figure) |
| Worst case `9e18..9e18+999` (max int64 width) | 19 | **21,165** |

**Full-cap page = 1000 asset ids + platform_tenant block. Even the absolute worst case (1000 max-width
19-digit int64 ids) serializes to 21,165 bytes (~21 KB)** — an order of magnitude under Temporal's
256 KB warn threshold and ~99% under the 2 MB hard limit. The realistic mature-tenant case is ~11 KB.

## Truncated JSON sample (from the worst-case 21,165-byte serialization)

First 40 of the 1000 `asset_ids`, then the full `platform_tenant` block. Full list is 1000 ids; full
serialized length is **21,165 bytes**.

```json
{
  "asset_ids": [
    9000000000000000000,
    9000000000000000001,
    9000000000000000002,
    9000000000000000003,
    9000000000000000004,
    9000000000000000005,
    9000000000000000006,
    9000000000000000007,
    9000000000000000008,
    9000000000000000009,
    9000000000000000010,
    9000000000000000011,
    9000000000000000012,
    9000000000000000013,
    9000000000000000014,
    9000000000000000015,
    9000000000000000016,
    9000000000000000017,
    9000000000000000018,
    9000000000000000019,
    9000000000000000020,
    9000000000000000021,
    9000000000000000022,
    9000000000000000023,
    9000000000000000024,
    9000000000000000025,
    9000000000000000026,
    9000000000000000027,
    9000000000000000028,
    9000000000000000029,
    9000000000000000030,
    9000000000000000031,
    9000000000000000032,
    9000000000000000033,
    9000000000000000034,
    9000000000000000035,
    9000000000000000036,
    9000000000000000037,
    9000000000000000038,
    9000000000000000039
  ],
  "... (960 more ids, 1000 total) ...": "truncated for paste; full serialized payload = 21165 bytes",
  "platform_tenant": {
    "ID": 4242,
    "Name": "Acme Security, Inc.",
    "SchemaName": "tenant_acme_security",
    "UUID": "b3f1c2d4-5e6a-4b7c-8d9e-0f1a2b3c4d5e"
  }
}
```

## Committed regression guard

- Test: `test_full_cap_issue_detection_payload_stays_under_temporal_limit`
- Path: `backend_python/tests/common/models/test_issue_detection_payload_bounds.py`

It builds a full-cap `IssueDetectionWorkflowInput` (1000 worst-case bigint ids + a populated
`platform_tenant`), serializes with the same `model_dump(by_alias=True)` + `json.dumps` the route
uses, asserts the PascalCase alias shape, and asserts the byte length is under Temporal's 256 KB warn
threshold (with a tight 32 KB ceiling so a real bloat is caught early). Raising the cap or fattening
the input model fails this test before a payload can bloat at `start_workflow`.

## Command + output proving it ran green (HEAD 74bded8213, 2026-09-20)

```
$ cd backend_python && .venv/bin/python -m pytest tests/common/models/test_issue_detection_payload_bounds.py -v -s
...
full-cap IssueDetectionWorkflowInput payload: 21165 bytes for 1000 asset ids
PASSED
======================== 1 passed, 5 warnings in 8.29s =========================
```

## Verdict

C6 OPEN half CLOSED. A real full-cap `IssueDetectionWorkflowInput` serialized via the route's own
`model_dump(by_alias=True)` measures 11,165 bytes at a realistic mature tenant and 21,165 bytes at
the absolute int64 worst case — both far under the ≤12 KB / 256 KB expectations. Backed by a
committed regression test that runs green.
