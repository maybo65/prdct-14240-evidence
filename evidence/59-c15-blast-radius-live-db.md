# C15 — `unsanctioned-agent-deployed` revive: live blast-radius on a real tenant DB

**Stamp:** run on the factory e2e Tilt tenant (`kind-slate`, ns `onyxd`, tenant schema `onyx_security`), branch HEAD `fce925c99b` · PR #13392.
**For:** the C15 owner go/no-go. This supplies the **numbers** the re-audit asked for; the fleet-wide sign-off itself remains May's (bucket a) — I do not self-approve a fleet-wide default change.

> The re-audit asked (verbatim, paraphrased): count non-deleted assets of type DESKTOP_AGENT (or HOMEGROWN with sub_type AI_AGENT) whose status is not SANCTIONED (NULL included), grouped by app definition; plus the current `issue_definitions.is_enabled` for `unsanctioned-agent-deployed` before and after migration `a1b7c3e5d9f2`. Both below, run live. Pre-branch tenant data is acceptable per the re-audit.

## The migration under review
`backend_python/src/db_migrations/versions/tenant/2026_09_13_1600-a1b7c3e5d9f2_reenable_revived_unsanctioned_agent_issue.py` → runs `sql/reenable_revived_unsanctioned_agent_issue.sql`, whose entire DML is:
```sql
UPDATE issue_definitions
SET is_enabled = true
WHERE identifier = 'unsanctioned-agent-deployed'
  AND is_enabled = false;
```
Forward-only, idempotent, scoped to exactly the one catalog row still carrying the retirement tombstone (`is_enabled=false`). It flips a *retired* issue back on so tombstoned tenants detect it like a fresh tenant would.

## 1) Blast radius — assets the revived issue would newly flag (live query)

```sql
SELECT COALESCE(app_definition_id::text,'(none)') AS app_definition_id, type, sub_type,
       COALESCE(status,'(null)') AS status, count(*)
FROM onyx_security.assets
WHERE is_deleted = false
  AND ( type = 'AssetTypeDesktopAgent'
        OR (type='AssetTypeHomegrown' AND sub_type='AssetSubTypeAIAgent') )
  AND ( status IS NULL OR status <> 'AssetStatusSanctioned' )
GROUP BY app_definition_id, type, sub_type, status
ORDER BY count(*) DESC, app_definition_id;
```

Raw output:
```
      app_definition_id       |         type          |            sub_type             |         status          | count
------------------------------+-----------------------+---------------------------------+-------------------------+-------
 AppIDChatGPTDesktop          | AssetTypeDesktopAgent | AssetSubTypeChatbotsAndCopilots | AssetStatusDiscovered   |     1
 AppIDClaudeDesktop           | AssetTypeDesktopAgent | AssetSubTypeAIAgent             | AssetStatusDiscovered   |     1
 AppIDMicrosoftCopilotDesktop | AssetTypeDesktopAgent | AssetSubTypeChatbotsAndCopilots | AssetStatusDiscovered   |     1
 endpoint_claude-code         | AssetTypeDesktopAgent | AssetSubTypeAIAgent             | (null)                  |     1
 endpoint_codex               | AssetTypeDesktopAgent | AssetSubTypeAIAgent             | (null)                  |     1
 endpoint_kimi-cli            | AssetTypeDesktopAgent | AssetSubTypeAIAgent             | AssetStatusUnsanctioned |     1
(6 rows)
```
**Total in this tenant: 6 assets** across 6 app definitions. Note two are already `status=(null)` or `Discovered` on unmanaged endpoints, and one is the newly-added `endpoint_kimi-cli` asset itself (`Unsanctioned`). The revive turns "unsanctioned agent deployed" from silent to a live posture issue for exactly these rows.

## 2) `is_enabled` before → after the migration (live, ROLLED BACK — nothing persisted)

This tenant is **already `is_enabled=t`** (it was provisioned at/after the revive, so it inserted the definition with the column default `true` — the migration is a **no-op** here, `WHERE is_enabled=false` matches zero rows). That is the honest state of this box. To show the transition the migration performs on a **tombstoned** tenant, I simulated the tombstone and ran the exact migration DML inside a transaction I then `ROLLBACK`, so no state was mutated:

```sql
BEGIN;
SELECT is_enabled ...;                                   -- current(head-provisioned) => t
UPDATE issue_definitions SET is_enabled=false WHERE identifier='unsanctioned-agent-deployed';
SELECT is_enabled ...;                                   -- simulated-tombstone(before) => f
UPDATE issue_definitions SET is_enabled=true
  WHERE identifier='unsanctioned-agent-deployed' AND is_enabled=false;   -- the migration body
SELECT is_enabled ...;                                   -- after-migration => t
ROLLBACK;
```
Raw output:
```
           phase           | is_enabled
---------------------------+------------
 current(head-provisioned) | t
 simulated-tombstone(before)| f
 after-migration           | t
```
Confirms: on a tombstoned tenant `false → true`; on an already-enabled tenant it is an idempotent no-op. `ROLLBACK` left the real row untouched (`is_enabled=t`).

## Owner decision (bucket a → @maybo65)
The mechanism, scope, and blast-radius numbers are proven above. The **go/no-go on enabling this posture issue fleet-wide** — 6 assets flip to flagged in *this* tenant, and proportionally across the fleet — is a product default-change sign-off that belongs to the owner, not to me. I have not self-approved it.
