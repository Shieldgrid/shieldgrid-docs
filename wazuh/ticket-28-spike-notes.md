# Ticket 28 — Wazuh Manager API + Velociraptor Query Spike Notes

Goal: connect `shieldgrid-core` to the **Wazuh Manager API** (agent inventory + operational data, distinct from the existing indexer-only alerts connector) and surface **Velociraptor queries/artifacts** from the UI. All connectivity below was verified live against the running environment on 2026-07-31.

## Environment (verified)

All three backends run inside the Multipass VM **Mx9** (10.244.222.140):

| Target | URL | Auth | Status |
|---|---|---|---|
| Wazuh Manager API | `https://10.244.222.140:55000` | JWT via Basic-auth login | ✅ Working |
| Wazuh Indexer (OpenSearch) | `https://10.244.222.140:9200` | Basic (admin) | ✅ Working |
| Velociraptor Server | `https://10.244.222.140:8001` (gRPC mTLS) | client cert (api_client.yaml) | ✅ Working |

## 1. Wazuh Manager API — the interesting bit

Wazuh is deployed via Helm in k3s (ns `wazuh`, `wazuh/wazuh-manager:4.14.4`), exposed through the LoadBalancer `wazuh-wazuh-helm` (55000 → master pod).

**Version 4.14 ships the *new* API framework.** Classic Wazuh 4.x docs (`/api/login`, `/api/agents`, ...) do **not** apply:

| Probe | Result |
|---|---|
| `POST /api/login` | ❌ 404 |
| `GET /api/agents` | ❌ 404 |
| `GET /agents/004` | ❌ 404 (agent detail path differs) |
| `POST /security/user/authenticate` (Basic) | ✅ 200 → `{"data": {"token": "<JWT>"}}` |
| `GET /agents` (Bearer) | ✅ 200 → agent list |
| `GET /groups`, `GET /rules`, `GET /security/users` | ✅ 200 |

**Auth flow:** `POST /security/user/authenticate` with HTTP Basic (`wazuh-wui` + apiCred password from the helm chart). Response shape is `{"data": {...}, "error": 0}`; errors are `{"title": ..., "detail": ...}`. Token is an ES512 JWT. Implementation must **cache the token and re-authenticate on 401** (the manager also invalidates tokens).

**Agent data (live):** 2 agents registered — `000` (manager master) and `004` (`Ju-nine-Ngu-154d5`, Wazuh v4.14.5, active). The 004 agent is the host machine's agent.

**Implication for the connector:** the new-API route table must be confirmed against the running manager during the implementation spike (some classic endpoints still exist, some moved). Do not blindly port old docs.

## 2. Wazuh Indexer (OpenSearch) — already connected

`shieldgrid-core`'s existing `WazuhConnector` is indexer-only and **healthy** (verified via `GET /health`): it reads `wazuh-alerts-*` (cluster green; biggest daily index ~293k docs). Reachable from the host via the node IP + `WAZUH_INSECURE_TLS=true` (self-signed cert). No infra change needed here.

## 3. Velociraptor — already connected

Server `ghcr.io/velocidex/velociraptor-server:0.77.1` runs in Docker on Mx9, ports 8000-8001 (API) + 8889 (GUI). The core connector is healthy and already implements: gRPC VQL streaming (`run_query`, currently private), `fetch_alerts` (`Artifact.Custom.Server.Alerts`), and isolate/unisolate actions with pre-flight checks (target exists, last seen < 10 min, dispatch via `collect_client`, flow polling). See `velociraptor/ticket-21-spike-notes.md`.

Gap: `run_query` is not exposed through the API — there is no way for the UI or an analyst to run VQL or list artifacts/clients today.

## 4. Current core wiring

- `src/config.rs` — has `opensearch_*` and `velociraptor_api_client_yaml`; **no Wazuh manager config**.
- `.env` already carries `WAZUH_MANAGER_URL`, `WAZUH_MANAGER_USERNAME`, `WAZUH_MANAGER_PASSWORD`, `WAZUH_INSECURE_TLS` — present but **not wired into `Config`/`main.rs`**.
- `src/routes/mod.rs` — `/health`, `/api/v1/alerts`, `/auth/*`, `/audit`, `/cases*`. No operational (query/inventory) endpoints.

## 5. Proposed implementation (pending ticket approval)

### A. Wazuh Manager connector (`connectors/wazuh.rs` or a sibling)
1. Add `wazuh_manager_url`, `wazuh_manager_username`, `wazuh_manager_password` to `Config` (fail-fast via `require_var`, redacted in `Debug`).
2. Token cache: authenticate once, reuse, refresh on 401 (manager tokens are short-lived).
3. Support (spike first, then build): `GET /agents` (inventory: id/name/ip/os/status/version), agent detail, `GET /groups`, `GET /rules` (+ CDB lists if useful). Keep the new-API path table in one place.
4. Keep the existing indexer-based `fetch_alerts` untouched — the manager connector is for **operations/inventory**, the indexer for **alert search**.

### B. Expose Velociraptor queries through the API
1. Make `run_query` accessible (public wrapper with row cap + timeout already present in the gRPC call).
2. New routes (all JWT + RBAC + audit-logged):
   - `GET /api/v1/velociraptor/clients` — client inventory (hostname, os, last_seen).
   - `GET /api/v1/velociraptor/artifacts` — curated, safe artifact list w/ parameters.
   - `POST /api/v1/velociraptor/query` — free-form VQL (`{vql, client_id?}`); **admin-only**, row cap 500, 30 s timeout, result capped + audit-logged (Rule 5/6/7).

### C. Web UI (later phase)
- Velociraptor query page: client picker → artifact picker (with params) **and** a free VQL box (admin-only); results as a table + CSV download.
- Wazuh agent inventory page: status badges, drill-down to agent detail.

## 6. Open decisions (need owner sign-off)

1. **Free-form VQL**: admin-only (recommended) or any analyst?
2. **Results persistence**: transient display only, or attach query results to a case?
3. **Build order**: Wazuh manager inventory first, or Velociraptor query UI first?
4. **MCP surface**: also expose these as `shieldgrid-mcp` tools (e.g. `run_vql_query`, `list_agents`), or UI-only for now?

## 7. Notes / gotchas

- **Never paste the real manager password into chat/logs** — it lives in `wazuh-helm/charts/wazuh/values.yaml` (`apiCred`) and in `shieldgrid-core/.env`; reference them via env vars.
- Wazuh 4.14's API is a moving target between minor versions — verify endpoint shapes against the live manager, don't trust docs alone.
- Velociraptor VQL runs on the server side and is sandboxed by Velociraptor itself, but arbitrary VQL is still high-impact (admin gate + audit), per Rule 3/7.
