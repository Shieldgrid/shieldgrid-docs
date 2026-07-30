# Phase 3 — Velociraptor Spike Notes

## Confirmed VQL Capabilities (on local Mx9 instance)

Investigated via `multipass exec Mx9 -- docker exec velociraptor-docker-velociraptor-server-1 /bin/velociraptor ...`

### Available Isolation Artifacts

| Artifact | OS | Mechanism |
|---|---|---|
| `Linux.Remediation.Quarantine` | Linux | `nftables` table (`vrr_quarantine_table`) |
| `Windows.Remediation.Quarantine` | Windows | IPsec policy via `netsh ipsec` |
| `Windows.Remediation.QuarantineMonitor` | Windows | Periodic re-application of quarantine (event query) |

### Trigger Mechanism

Actions are dispatched using the `collect_client` VQL **function** (not plugin):

```sql
SELECT collect_client(
    client_id='C.2c537895848a98c2',
    artifacts='Linux.Remediation.Quarantine',
    env=dict(RemovePolicy='N')
) AS flow_id
FROM scope()
```

**Note:** `collect_client` is a VQL *function* (used in `SELECT`), not a plugin (used in `FROM`).

### Flow Status Polling

```sql
SELECT state FROM flows(client_id='C.xxx', flow_id='F.yyy')
```

Flow `state` values observed:
- `RUNNING` — still executing
- `FINISHED` — artifact completed successfully
- `ERROR` — artifact execution failed

### Client Discovery

Active clients visible via:
```sql
SELECT client_id, os_info.hostname, last_seen_at FROM clients()
```

The Mx9 VM client (`C.2c537895848a98c2`) was confirmed active with `last_seen_at` < 1 minute.

### Server Config Location (inside Mx9 Docker)

```
/etc/velociraptor/server.config.yaml
```

The Velociraptor server runs inside Docker on the Mx9 Multipass VM, exposed on ports 8000 (frontend/GUI) and 8001 (gRPC API).

## Rejected Alternatives

- **Process kill** (`Server.Utils.KillClient`) — considered too disruptive for first action, and also kills the Velociraptor agent itself
- **DNS sinkholing** (`Windows.Remediation.Sinkhole`) — Windows-only and requires careful rollback management
