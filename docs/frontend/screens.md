# Shieldgrid Web — Screens & API Consumption

This document describes the function and backend API consumption of each frontend route.

---

### `/login`
- **Purpose:** Operator authentication screen.
- **Endpoints Consumed:**
  - `POST /api/v1/auth/login` (exchanges email/password for JWT token)

---

### `/dashboard`
- **Purpose:** Central operational overview. Displays summary metrics, connector health, and the live Core Orbit node graph.
- **Endpoints Consumed:**
  - `GET /health` (connector health statuses)
  - `GET /api/v1/alerts` (recent alert metrics)
  - `GET /api/v1/cases` (open case count)

---

### `/alerts`
- **Purpose:** Merged alert triage queue for security events. Supports client-side filtering by connector, severity, and status.
- **Endpoints Consumed:**
  - `GET /api/v1/alerts` (supports `?since=` parameter)

---

### `/cases`
- **Purpose:** Incident management queue. Displays active cases and allows analysts to open new incident cases.
- **Endpoints Consumed:**
  - `GET /api/v1/cases` (list all cases)
  - `POST /api/v1/cases` (create new case)

---

### `/cases/:id`
- **Purpose:** Detailed investigation workspace. Allows updating case status, assigning analysts, viewing linked alert telemetry, and attaching/detaching alerts.
- **Endpoints Consumed:**
  - `GET /api/v1/cases/:id` (case details)
  - `PATCH /api/v1/cases/:id` (update status/assignment)
  - `GET /api/v1/cases/:id/alerts` (list attached alert IDs)
  - `POST /api/v1/cases/:id/alerts` (attach alert to case)
  - `DELETE /api/v1/cases/:id/alerts/:alert_id` (detach alert from case)

---

### `/connectors`
- **Purpose:** Admin-only view showing status and diagnostic error reasons for integrated connectors (Wazuh, Velociraptor).
- **Endpoints Consumed:**
  - `GET /health`

---

### `/audit`
- **Purpose:** Admin-only audit trail displaying system activity logs.
- **Endpoints Consumed:**
  - `GET /api/v1/audit`
