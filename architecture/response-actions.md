# Response Actions — Architecture & Operator Guide

## Overview

Phase 3 adds **Response Actions** to Shieldgrid — the ability to execute remediation commands against a live endpoint directly from the SOC interface. The first action implemented is network isolation via Velociraptor's `Linux.Remediation.Quarantine` artifact.

> **Safety principle:** Every action ticket is held to a higher bar than observation tickets. An action that is "mostly right" is a much bigger problem than a dashboard screen that is mostly right.

---

## Architecture

```
Analyst (browser)
      │
      │  POST /api/v1/cases/:id/actions
      ▼
shieldgrid-core (Axum)
      │
      │  1. Write audit_log (action_request)
      │  2. Validate target via VQL
      │  3. collect_client() → flow_id
      │  4. Poll flows() for up to 30s
      │  5. Write audit_log (action_success/failure/timeout)
      │
      ▼
Velociraptor Server (gRPC)
      │
      │  Linux.Remediation.Quarantine
      │  (nftables: drops all traffic except DNS + VR server)
      ▼
Target Endpoint (Velociraptor client)
```

---

## Connector Trait Extension

Every connector now implements:

```rust
async fn push_action(&self, action: ResponseAction) -> Result<ActionResult>;
```

| Connector   | Isolation Support | Notes                                  |
|-------------|-------------------|----------------------------------------|
| Velociraptor | ✅ Yes            | `Linux.Remediation.Quarantine`         |
| Wazuh        | ❌ No             | Returns `success: false` with detail   |

---

## API Reference

### Execute an Action

```
POST /api/v1/cases/:case_id/actions
Authorization: HttpOnly cookie (jwt_token)
Role: Admin required
Content-Type: application/json

{
  "connector_id": "velociraptor",
  "action_type": "isolate",
  "target_id": "C.2c537895848a98c2"
}
```

**`action_type` values:**
- `"isolate"` — Apply nftables quarantine (blocks all traffic except DNS + Velociraptor)
- `"unisolate"` — Remove quarantine rules, restore full network access

**Response:**
```json
{
  "success": true,
  "detail": "Action isolate completed successfully",
  "is_timeout": false,
  "timestamp": "2026-07-30T14:00:00Z"
}
```

**Outcome states:**

| `success` | `is_timeout` | Meaning |
|-----------|-------------|---------|
| `true`    | `false`     | Artifact ran and confirmed FINISHED |
| `false`   | `false`     | Definitive failure (invalid target, VQL error, artifact ERROR state) |
| `false`   | `true`      | Action was dispatched but endpoint did not confirm within 30s. Verify manually in Velociraptor. |

### List Case Actions

```
GET /api/v1/cases/:case_id/actions
Authorization: HttpOnly cookie (jwt_token)
Role: Admin required
```

Returns audit log entries for this case with `action` matching `action_%`.

---

## Safety Mechanisms

### 1. Pre-flight Target Validation

Before any artifact is dispatched, the backend queries Velociraptor:

```sql
SELECT os_info.hostname, last_seen_at
FROM clients(client_id='<target_id>')
```

**Abort conditions:**
- **0 rows returned** → target `client_id` does not exist on this Velociraptor server. Request rejected with 400-class error.
- **`last_seen_at` > 10 minutes ago** → client is considered offline. Request rejected synchronously rather than queuing an action to an unresponsive host.

### 2. Idempotency

`Linux.Remediation.Quarantine` creates (or replaces) a named `nftables` table (`vrr_quarantine_table`). Running isolate twice is safe — the second run replaces the same table. Running unisolate on a non-isolated host also safe — it simply finds no table to remove.

### 3. Audit Trail

Every action creates **two** audit log entries:

| Timing | `action` value | `target` format |
|--------|---------------|-----------------|
| Before execution | `action_request` | `case:<id>:target:<client_id>:type:<action>` |
| After execution | `action_success` / `action_failure` / `action_timeout` | `case:<id>:target:<client_id>:type:<action>:detail:<message>` |

This ensures a record exists even if the server crashes mid-execution.

### 4. Timeout Handling

The backend polls the Velociraptor flow status every 2 seconds for up to **30 seconds** (15 iterations). If the flow does not reach `FINISHED` or `ERROR` within this window:

- `success: false`, `is_timeout: true` is returned
- An `action_timeout` audit entry is written
- The UI displays: *"Action request sent, but outcome could not be verified. Verify manually in Velociraptor."*

The action is **not** marked as failed — the artifact may still be executing and complete successfully after the timeout.

---

## Extending to New Action Types

To add a new action type (e.g., `kill_process`):

1. **Add a new VQL mapping** in `push_action` in `velociraptor.rs`:
   ```rust
   "kill_process" => ("Windows.Remediation.KillProcess", "N"),
   ```

2. **Extend `ActionRequest.action_type`** in `models/action.rs` with documentation.

3. **Update the frontend type union** in `types.ts`:
   ```ts
   action_type: 'isolate' | 'unisolate' | 'kill_process';
   ```

4. **Update the modal** in `CaseDetailPage.tsx` to expose the new button.

---

## Windows Isolation (Future)

For Windows endpoints, replace the artifact with `Windows.Remediation.Quarantine`, which uses IPsec policy instead of nftables. The API payload is identical — only the connector-side mapping changes.

The OS can be determined automatically via a pre-flight `os_info.system` query on the client record, and the correct artifact selected without any API change.

---

## Testing the Action Endpoint (curl)

```bash
# 1. Log in and capture the session cookie
curl -c /tmp/sg_cookie.txt -X POST \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@shieldgrid.local","password":"admin"}' \
  http://localhost:3000/api/v1/auth/login

# 2. Get a case ID
curl -b /tmp/sg_cookie.txt http://localhost:3000/api/v1/cases | jq '.[0].id'

# 3. Execute an isolation (replace CASE_ID and CLIENT_ID)
curl -b /tmp/sg_cookie.txt -X POST \
  -H "Content-Type: application/json" \
  -d '{"connector_id":"velociraptor","action_type":"isolate","target_id":"C.2c537895848a98c2"}' \
  http://localhost:3000/api/v1/cases/CASE_ID/actions

# 4. Check action history for the case
curl -b /tmp/sg_cookie.txt \
  http://localhost:3000/api/v1/cases/CASE_ID/actions | jq .
```
