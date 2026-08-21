# Shieldgrid × CoPilot Feature Parity — Implementation Plan

## Executive Summary

This document maps CoPilot's feature set to Shieldgrid and defines a phased implementation plan. Shieldgrid already has strong foundations (Wazuh + Velociraptor connectors, case management, AI analyst, threat intel, actions), but needs expansion to match CoPilot's breadth.

**Current Shieldgrid Status vs CoPilot:**
- ✅ Built: Wazuh connector, Velociraptor connector, Case management, AI analyst, Threat intel, Action execution, Audit logging, Auth/RBAC, Detection rules, MITRE matrix
- 🔜 Missing: Network connectors, additional integrations, active response automation, multi-tenancy, customer portal, advanced features

---

## Phase 1: Active Response & Automation (Weeks 1-2)

### 1.1 Wazuh Active Response Integration
**CoPilot Feature:** `active_response` module — triggers Wazuh AR commands (firewall-drop, netsh-command, etc.)

**What to build in Shieldgrid:**
- Extend `WazuhConnector` with `push_action()` implementation (currently returns "not supported")
- Add Wazuh manager API calls: `POST /active-response` endpoint
- New action templates: `wazuh_firewall_drop`, `wazuh_netsh_block`, `wazuh_custom_script`
- DB migration: `wazuh_active_response` table to track AR command history

**Files to modify:**
- `shieldgrid-core/src/connectors/wazuh.rs` — implement `push_action()`
- `shieldgrid-core/src/services/actions.rs` — add Wazuh AR dispatch logic
- `shieldgrid-core/migrations/` — new migration for AR tracking

### 1.2 Shuffle Workflow Integration
**CoPilot Feature:** `shuffle` connector — triggers Shuffle SOAR workflows

**What to build:**
- New connector: `shieldgrid-core/src/connectors/shuffle.rs`
- Implement `Connector` trait for Shuffle
- API integration: trigger workflows via Shuffle REST API
- Webhook receiver: receive workflow results back into cases
- New action template: `trigger_shuffle_workflow`

**Files to create:**
- `shieldgrid-core/src/connectors/shuffle.rs`
- `shieldgrid-core/src/routes/shuffle.rs` (webhook receiver)
- `shieldgrid-core/src/models/shuffle.rs`

### 1.3 Scheduled Actions & Automation
**CoPilot Feature:** `schedulers` module — automated tasks

**What to build:**
- Background scheduler service (cron-like)
- Schedule recurring actions (e.g., "every hour, check for new critical alerts and trigger Shuffle workflow")
- DB schema: `action_schedules` table

---

## Phase 2: AI Analyst & Threat Intel Enhancements (Weeks 2-3)

### 2.1 Enhanced AI Analyst
**CoPilot Feature:** `ai_analyst` module — autonomous investigation

**Current Shieldgrid:** Basic triage and posture summary

**Enhancements:**
- LLM integration (OpenAI/Anthropic) for natural language analysis
- Auto-triage: batch process alerts with AI recommendations
- Alert correlation: AI groups related alerts into incidents
- Response recommendations with confidence scores
- Chat interface for analyst-AI collaboration

**Files to modify:**
- `shieldgrid-core/src/services/ai_analyst.rs` — add LLM integration
- `shieldgrid-core/src/routes/ai_analyst.rs` — new endpoints for chat, batch triage
- New: `shieldgrid-core/src/services/llm.rs` (LLM client abstraction)

### 2.2 Threat Intel Expansion
**CoPilot Feature:** `threat_intel` module — multi-source intel

**Current Shieldgrid:** VirusTotal, EPSS

**Enhancements:**
- Add AlienVault OTX integration
- Add AbuseIPDB integration
- Add MISP connector for threat feeds
- Threat intel dashboard with trend analysis
- IOC correlation across cases

---

## Phase 3: Network Connectors (Weeks 3-4)

### 3.1 Syslog Receiver Service
**CoPilot Feature:** `network_connectors` — Fortinet, Palo Alto, Cisco

**What to build:**
- Syslog UDP/TCP listener service
- Parse CEF/LEEF syslog formats
- Normalize network alerts into `NormalizedAlert`
- New connector: `shieldgrid-core/src/connectors/syslog.rs`

**Files to create:**
- `shieldgrid-core/src/connectors/syslog.rs`
- `shieldgrid-core/src/services/syslog_parser.rs`

### 3.2 Device-Specific Connectors
**Implement parsers for:**
- Fortinet FortiGate (CEF format)
- Palo Alto PAN-OS (CEF format)
- Cisco ASA (Syslog format)
- Each parser maps device-specific fields to normalized alert schema

---

## Phase 4: Additional Integrations (Weeks 4-5)

### 4.1 Graylog Integration
**CoPilot Feature:** `graylog` connector

**What to build:**
- New connector: query Graylog GELF API
- Fetch alerts, correlate with Wazuh
- Graylog dashboard integration

### 4.2 Grafana Integration
**CoPilot Feature:** `grafana` connector

**What to build:**
- Embed Grafana dashboards in Shieldgrid UI
- Grafana alerting integration
- Dashboard provisioning via API

### 4.3 InfluxDB Integration
**CoPilot Feature:** `influxdb` connector

**What to build:**
- Metrics collection from InfluxDB
- Performance dashboards
- Alert thresholds based on metrics

---

## Phase 5: Multi-tenancy & Customer Portal (Weeks 5-6)

### 5.1 Multi-tenant Architecture
**CoPilot Feature:** `customers`, `customer_provisioning`

**What to build:**
- Tenant isolation in PostgreSQL (row-level security)
- Tenant management API
- Tenant-scoped data access
- Billing/usage tracking

### 5.2 Customer Portal
**CoPilot Feature:** `customer_portal`

**What to build:**
- Separate frontend for customer users
- Limited visibility (own alerts/cases only)
- Case collaboration features

---

## Phase 6: Additional Security Integrations (Weeks 6-7)

### 6.1 EDR/SIEM Connectors
**CoPilot integrations:** CrowdStrike, Carbon Black, Darktrace, etc.

**What to build:**
- CrowdStrike connector (API integration)
- Carbon Black connector
- Darktrace connector
- Each follows the standard `Connector` trait

### 6.2 Email Security
**CoPilot integrations:** Mimecast, Sublime

**What to build:**
- Mimecast connector
- Sublime Security connector
- Email alert normalization

### 6.3 Cloud Security
**CoPilot integrations:** Duo, GitHub Audit, Bitdefender

**What to build:**
- Duo connector (MFA events)
- GitHub Audit log connector
- Bitdefender connector

---

## Phase 7: Advanced Features (Weeks 7-8)

### 7.1 Reporting & Analytics
**CoPilot Feature:** `performance`, `logs`

**What to build:**
- Report generation (PDF/CSV)
- Performance metrics dashboard
- Centralized log viewer
- SLA tracking

### 7.2 SSO & Advanced Auth
**CoPilot Feature:** SSO config

**What to build:**
- SAML/OIDC integration
- LDAP/AD connector
- SSO configuration UI

### 7.3 Notifications
**CoPilot Feature:** `notifications`

**What to build:**
- Email notifications
- Slack/Teams integration
- Webhook notifications
- Alert escalation policies

---

## Implementation Priority

### Immediate (Next 2-4 weeks):
1. **Week 1-2:** Phase 1 (Active Response & Automation)
   - Wazuh AR integration
   - Shuffle connector
   - Scheduled actions

2. **Week 2-3:** Phase 2 (AI Analyst & Threat Intel)
   - LLM integration for AI analyst
   - Additional threat intel sources

3. **Week 3-4:** Phase 3 (Network Connectors)
   - Syslog receiver
   - Fortinet/Palo Alto/Cisco parsers

### Short-term (1-2 months):
4. Phase 4 (Additional Integrations)
5. Phase 5 (Multi-tenancy)

### Medium-term (2-3 months):
6. Phase 6 (Security Integrations)
7. Phase 7 (Advanced Features)

---

## Technical Architecture Notes

### Connector Pattern (Already Established)
All new connectors follow the existing pattern:
```rust
#[async_trait]
impl Connector for NewConnector {
    fn id(&self) -> &'static str;
    async fn health_check(&self) -> HealthStatus;
    async fn fetch_alerts(&self, since: DateTime) -> Result<Vec<NormalizedAlert>>;
    async fn push_action(&self, action: ResponseAction) -> Result<ActionResult>;
}
```

### Database Migrations
Each phase requires new migrations. Follow existing pattern in `shieldgrid-core/migrations/`.

### Frontend Updates
Each backend feature needs corresponding React pages in `shieldgrid-web/src/pages/`.

---

## Success Metrics

1. **Feature Coverage:** Match 80% of CoPilot's backend modules
2. **Connector Count:** 10+ connectors (Wazuh, Velociraptor, Shuffle, Graylog, Grafana, InfluxDB, Fortinet, Palo Alto, Cisco, CrowdStrike)
3. **Response Time:** All API endpoints < 200ms
4. **Test Coverage:** > 70% code coverage for new features
5. **Documentation:** All new features documented in `shieldgrid-docs`

---

## Risk Mitigation

1. **Scope Creep:** Stick to phased approach, complete one phase before next
2. **Technical Debt:** Maintain existing code quality standards (AGENT_RULES.md)
3. **Integration Complexity:** Test each connector against real instances
4. **Performance:** Load test with realistic data volumes

---

## Next Steps

1. Review this plan and approve phases
2. Create GitHub issues for each phase
3. Start Phase 1 implementation
4. Set up development environments for testing
