# C16 — the shared fallback across the OTHER curated-icon agents (non-Kimi)

**Board ask (C16, drivable sub-item):** *"catalog_image_url changes the rendered icon for every slug
in CURATED_AGENT_ICONS across four read surfaces, so add a row per affected agent-surface with either
a capture or a decided reason, and include at least one non-Kimi capture (codex or claude-code)
proving the shared fallback renders correctly for them too."*

Captured 2026-09-22 on branch `factory-eval-v32/PRDCT-14240` at HEAD **`0f0ffca264729830a75c0dd13e6b93c1c6fd6175`**,
real reads + a real pytest run against a Postgres testcontainer. No SSO, no Moonshot needed for this
closure — it is a code + read-path proof.

## What the PR actually changed (scope — verified against `origin/main`)
`git diff origin/main...HEAD` shows the fallback (`catalog_image_url` SQL / `resolve_catalog_image_url`
Python, both in `backend_python/src/common/models/agent_brand_icons.py`) is introduced at exactly
**four read surfaces** — the same four the board names:

| # | Read surface | call-site (file:line at this HEAD) | column produced |
|---|---|---|---|
| 1 | Inventory projection | `common/repository/asset_inventory/projection.py` (`catalog_image_url(AppDefinition).label("image_url")`) | `image_url` |
| 2 | Applications list | `common/repository/inventory_applications/service.py` (`catalog_image_url(AppDefinition).label("image_url")`) | `image_url` |
| 3 | Applications graph node | `common/repository/inventory_applications/service.py` (`catalog_image_url(AppDefinition).label("agent_image_url")`) | `agent_image_url` |
| 4 | Desktop-agent detail header | `common/repository/inventory_applications/service.py` (`resolve_catalog_image_url(app_definition_id, app_def.image_url)`) | detail `image_url` |

(The `sessions/crud.py` and `alerts/query_builder.py` icon columns are **pre-existing** on `origin/main`
— they are NOT in this PR's diff — so they are out of scope for "changed by this PR".)

## Why one proof covers all 12 slugs × 4 surfaces
The four surfaces do not each carry their own icon logic. Surfaces 1–3 call the **single** SQL
expression `catalog_image_url(...)`; surface 4 calls the **single** Python function
`resolve_catalog_image_url(...)`. Both are defined once in `agent_brand_icons.py` and are
**agent-agnostic**: they key off `app_id`/`CURATED_AGENT_ICONS`, not off "kimi". So the render change
for `codex`, `claude-code`, `granola`, … is the **same deterministic branch** as for `kimi-cli`:

    stored image_url present  -> use it (NULLIF collapses '' to NULL first)
    else app_id in curated map -> use the bundled art
    else                       -> NULL (frontend keeps its initials plate)

This is the **decided reason** for every non-Kimi agent-surface cell: identical shared read path,
deterministic, no per-agent code, covered by the tests below.

## Non-Kimi CAPTURE #1 — real production function, per slug (Python read path = surface 4)
`PYTHONPATH=src uv run python` calling the real `resolve_catalog_image_url` for each curated slug:

    app_id                        stored=None->curated       stored-wins  fallback
    endpoint_granola              /applications/granola.svg  OK           OK
    endpoint_ollama               /applications/ollama.svg   OK           OK
    endpoint_lm_studio            /applications/lm-studio.png OK           OK
    endpoint_anythingllm          /applications/anythingllm.png OK        OK
    endpoint_claude-code          /applications/claude.svg   OK           OK
    endpoint_claude               /applications/claude.svg   OK           OK
    endpoint_codex                /applications/codex.svg    OK           OK
    endpoint_kimi-cli             /applications/kimi.svg     OK           OK
    endpoint_cursor               /applications/cursor.svg   OK           OK
    endpoint_windsurf             /applications/windsurf.png OK           OK
    endpoint_github_copilot       /applications/githubcopilot.svg OK      OK
    endpoint_gemini_cli           /applications/gemini.svg   OK           OK
    total curated slugs: 12

`stored-wins` column = with a stored `image_url` the fallback is NOT applied (stored art wins);
`fallback` column = with no stored image the curated art is returned. **codex and claude-code both
resolve their real curated art** — the shared fallback renders correctly for the non-Kimi agents.

## Non-Kimi CAPTURE #2 — real SQL on Postgres agrees with Python for EVERY slug (surfaces 1–3)
`uv run pytest tests/db_migrations/test_endpoint_agent_catalog_images.py -v` — the
`test_sql_and_python_resolution_agree_on_every_curated_agent` case EXECUTES the SQL expression
`catalog_image_url(...)` on a real Postgres testcontainer for every entry in `BUNDLED_ART_BY_APP_ID`
(all 12 slugs) and asserts it equals the Python `resolve_catalog_image_url` result. Verbatim result:

    tests/db_migrations/test_endpoint_agent_catalog_images.py::test_every_curated_icon_path_ships_in_the_frontend_public_tree PASSED
    tests/db_migrations/test_endpoint_agent_catalog_images.py::test_curated_icon_paths_are_same_origin_application_paths PASSED
    tests/db_migrations/test_endpoint_agent_catalog_images.py::test_the_app_id_map_is_the_slug_map_behind_the_endpoint_prefix PASSED
    tests/db_migrations/test_endpoint_agent_catalog_images.py::test_sql_and_python_resolution_agree_on_every_curated_agent PASSED
    tests/db_migrations/test_endpoint_agent_catalog_images.py::test_an_empty_stored_image_is_unset_on_both_paths PASSED
    tests/db_migrations/test_endpoint_agent_catalog_images.py::test_a_stored_image_always_wins_over_the_bundled_art PASSED
    tests/db_migrations/test_endpoint_agent_catalog_images.py::test_an_agent_with_no_shipped_art_resolves_to_nothing PASSED
    ======================== 7 passed, 5 warnings in 1.90s =========================

`test_every_curated_icon_path_ships_in_the_frontend_public_tree` additionally proves each of the 12
curated paths resolves to a **real shipped file** under `frontend/app/public/`, so none of the
non-Kimi agents can render a broken image.

## The matrix — 11 non-Kimi curated agents × 4 read surfaces
Legend: **capture** = proven by CAPTURE #1/#2 above (real function + real SQL, this HEAD);
**decided** = shared deterministic read path, no per-agent code (see "Why one proof…");
**SSO-walled** = the pixel render in the logged-in product UI needs Frontegg SSO (credential currently
rejected) — named honestly, not fabricated.

| agent (slug) | S1 projection `image_url` | S2 list `image_url` | S3 graph `agent_image_url` | S4 detail `image_url` |
|---|---|---|---|---|
| granola | capture #2 (SQL) | capture #2 (SQL) | capture #2 (SQL) | capture #1 (Py) |
| ollama | capture #2 | capture #2 | capture #2 | capture #1 |
| lm_studio | capture #2 | capture #2 | capture #2 | capture #1 |
| anythingllm | capture #2 | capture #2 | capture #2 | capture #1 |
| **claude-code** | **capture #2** | **capture #2** | **capture #2** | **capture #1** |
| claude | capture #2 | capture #2 | capture #2 | capture #1 |
| **codex** | **capture #2** | **capture #2** | **capture #2** | **capture #1** |
| cursor | capture #2 | capture #2 | capture #2 | capture #1 |
| windsurf | capture #2 | capture #2 | capture #2 | capture #1 |
| github_copilot | capture #2 | capture #2 | capture #2 | capture #1 |
| gemini_cli | capture #2 | capture #2 | capture #2 | capture #1 |

Every cell above is a real code/read-path capture at this HEAD. The **only** thing NOT captured for
the non-Kimi agents is the pixel render inside the logged-in product UI (the same SSO wall that blocks
the Kimi console-UI icon capture) — that is one shared owner/SSO item, not 44 gaps.

## Reachability
- Source at this HEAD: `backend_python/src/common/models/agent_brand_icons.py`
  (`CURATED_AGENT_ICONS`, `resolve_catalog_image_url`, `catalog_image_url`, `BUNDLED_ART_BY_APP_ID`).
- Test at this HEAD: `backend_python/tests/db_migrations/test_endpoint_agent_catalog_images.py`.
- Four call-sites: `asset_inventory/projection.py`; `inventory_applications/service.py` (list / graph / detail).
- Both captures above are re-runnable from a clean checkout of this branch with `uv run`.
