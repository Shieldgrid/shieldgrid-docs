# Shieldgrid Web: Information Architecture & Route Design

This document outlines the proposed frontend routes for `shieldgrid-web`. The design is informed by analyzing comparable SOC platform workflows (like SOCFortress CoPilot) but rigorously scoped to Shieldgrid's existing data model (`User`, `Connector`, `Alert`, `Case`, `Action`, `AuditLog`) and currently implemented backend endpoints.

## Route List

### `/login`
**Purpose:** Serves as the authentication entry point for operators and administrators. Users provide credentials to obtain a session/token, which authorizes them to interact with the platform.
**Consumes:**
- `POST /api/v1/auth/login`

### `/dashboard`
**Purpose:** A high-level operational snapshot. This screen gives analysts immediate context on system health and workload, displaying active connector statuses (e.g., "Wazuh down") alongside high-level metrics derived from recent alerts and open cases. 
**Consumes:**
- `GET /health` (for per-connector health signals)
- `GET /api/v1/alerts` (to derive recent alert volume)
- `GET /api/v1/cases` (to derive open case counts)

### `/alerts`
**Purpose:** The centralized triage queue for security events. This screen displays a normalized, merged list of alerts from all registered connectors. Analysts can review event details and perform client-side filtering by connector, severity, or status to identify events that require escalation into cases.
**Consumes:**
- `GET /api/v1/alerts` (supports `?since=` for time-bound queries)

### `/cases`
**Purpose:** The incident management hub. This view provides a list of all active and historical cases, allowing operators to track the lifecycle of investigations. Analysts can create new cases directly from this view or click into a specific case for deeper investigation.
**Consumes:**
- `GET /api/v1/cases` (list all cases)
- `POST /api/v1/cases` (create a new case)

### `/cases/:id`g
**Purpose:** The detailed investigation workspace for a specific case. Here, analysts can collaborate, update the case status, assign it to a user, and manage evidence. It includes an interface to view linked alerts, attach new relevant alerts, or detach false positives. Future capabilities will allow analysts to trigger response `Action`s directly from this view.
**Consumes:**
- `GET /api/v1/cases/{id}` (fetch case details)
- `PATCH /api/v1/cases/{id}` (update status or assignment)
- `GET /api/v1/cases/{id}/alerts` (list attached alert IDs)
- `POST /api/v1/cases/{id}/alerts` (attach an alert to the case)
- `DELETE /api/v1/cases/{id}/alerts/{alert_id}` (remove an alert from the case)

### `/connectors`
**Purpose:** An admin-only interface for managing integrations. Operators can verify connectivity to third-party tools (e.g., Wazuh, Velociraptor) and view detailed error reasons if a connector is degraded or down.
**Consumes:**
- `GET /health` (extracts connector array containing `id`, `status`, and `reason`)

### `/audit`
**Purpose:** An admin-only system transparency view. This screen displays an immutable log of platform activities (such as case creation, status updates, or alert attachments), ensuring accountability for all user and system actions.
**Consumes:**
- `GET /api/v1/audit`
