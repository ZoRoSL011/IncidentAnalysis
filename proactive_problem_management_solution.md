# Proactive Problem Management Automation Solution (ServiceNow Incident Export Driven)

## Assumptions for MVP
1. Incident exports are provided as CSV/XLSX with mostly consistent column names and date formats.
2. No direct write-back to ServiceNow in MVP; PRB workflow is maintained in-app first.
3. AI features use an LLM service with prompt templates and strict evidence-grounding logic in the backend.
4. Primary users are Problem Managers and technical managers with basic analytics literacy.
5. Data volume for MVP is up to ~1 million incidents/year; daily ingest batches are acceptable.
6. SLA fields may be missing in some exports; dashboard should degrade gracefully.

---

## 3.1 Product Requirements (PRD)

### Personas
1. **Problem Manager**
   - Owns proactive problem management lifecycle.
   - Needs fast identification of recurring issues and action plans.
2. **Service Desk Lead**
   - Monitors ticket quality and recurring operational issues.
   - Needs trend views by team/service and early warning spikes.
3. **Technical Team Manager**
   - Owns remediation execution for specific CIs/services.
   - Needs clear RCA evidence and actionable task tracking.
4. **Service Owner**
   - Accountable for business service reliability and risk.
   - Needs concise executive summaries, priorities, and progress.

### User Stories + Acceptance Criteria
1. **As a Problem Manager, I upload a CSV/XLSX export so analysis is automatic.**
   - AC1: System validates required columns and date formats before ingest.
   - AC2: Errors show exact row/field issues and downloadable rejection report.
   - AC3: Upload completion under 2 minutes for 100k rows.

2. **As a Service Desk Lead, I want trend dashboards to identify repeat issues quickly.**
   - AC1: Dashboard includes volume over time, top CIs/groups/services, cause/sub-cause, Pareto.
   - AC2: Filters update all charts within 3 seconds for standard queries.
   - AC3: Clicking a chart segment opens supporting incident list.

3. **As a Technical Team Manager, I need clustered repeat issues and RCA drafts.**
   - AC1: Clusters show confidence score and supporting incident numbers.
   - AC2: 5-Why draft cites incidents or says “Insufficient evidence.”
   - AC3: Manager can edit/save RCA and mitigation items.

4. **As a Service Owner, I need manager-ready reports by team.**
   - AC1: RCA report exportable as PDF and HTML.
   - AC2: Report includes evidence table (incident IDs + key fields).
   - AC3: Email sending supports multiple recipients and logs delivery status.

5. **As a Problem Manager, I need PRB workflow tracking in one place.**
   - AC1: Can create PRB and tasks linked to cluster/report.
   - AC2: Status workflow enforced: New → Under Investigation → RCA Completed → Mitigation In Progress → Monitoring → Resolved → Closed.
   - AC3: Audit log captures who changed status/fields and when.

### Non-Functional Requirements
- **Security**
  - Role-based access control (RBAC) by role/team.
  - Encryption in transit (TLS) and at rest for DB backups.
  - PII minimization and configurable data retention.
- **Performance**
  - 100k-row uploads process <2 minutes.
  - Dashboard interactions <3 seconds for common filter sets.
- **Auditability**
  - Every AI conclusion mapped to supporting incidents.
  - Immutable audit entries for uploads, RCA edits, status changes, email sends.
- **Reliability**
  - Background jobs retried with dead-letter handling.
  - Daily backup and restore procedure.
- **Accessibility**
  - WCAG-friendly contrast, keyboard navigation, form labels, and accessible charts fallback tables.

### Out of Scope (MVP)
- Full bidirectional ServiceNow synchronization.
- Auto-remediation orchestration.
- Multilingual NLP support.
- Advanced forecasting (ARIMA/Prophet) beyond anomaly/spike alerts.
- Native mobile app.

---

## 3.2 Data Model + Schema

### Required Incident Fields
- incident_number (string)
- created_at (datetime)
- resolved_at (datetime, nullable)
- state (string)
- priority (string/int)
- impact (string/int)
- urgency (string/int)
- configuration_item (string)
- assignment_group (string)
- short_description (text)

### Optional Incident Fields
- resolution_code (string)
- cause (string)
- sub_cause (string)
- technical_service (string)
- business_service (string)
- sla_breach (bool)
- close_notes (text)
- reopened_count (int)
- resolver (string)

### Normalized Schema (Relational)

#### `incidents`
- `id` UUID PK
- `incident_number` VARCHAR(32) UNIQUE NOT NULL
- `created_at` TIMESTAMP NOT NULL
- `resolved_at` TIMESTAMP NULL
- `state` VARCHAR(32) NOT NULL
- `priority` VARCHAR(16) NOT NULL
- `impact` VARCHAR(16) NOT NULL
- `urgency` VARCHAR(16) NOT NULL
- `configuration_item` VARCHAR(255) NOT NULL
- `assignment_group` VARCHAR(255) NOT NULL
- `short_description` TEXT NOT NULL
- `resolution_code` VARCHAR(128) NULL
- `cause` VARCHAR(255) NULL
- `sub_cause` VARCHAR(255) NULL
- `technical_service` VARCHAR(255) NULL
- `business_service` VARCHAR(255) NULL
- `sla_breach` BOOLEAN NULL
- `source_file_id` UUID FK -> `uploads.id`
- `created_by_user_id` UUID FK -> `users.id`
- Indexes: `(created_at)`, `(assignment_group)`, `(configuration_item)`, `(cause, sub_cause)`, full-text index on `short_description`

#### `uploads`
- `id` UUID PK
- `filename` VARCHAR(255)
- `uploaded_at` TIMESTAMP
- `uploaded_by_user_id` UUID FK -> `users.id`
- `row_count` INT
- `status` VARCHAR(32)
- `error_report_path` TEXT NULL

#### `clusters`
- `id` UUID PK
- `cluster_key` VARCHAR(128) UNIQUE
- `title` VARCHAR(255)
- `summary` TEXT
- `algorithm_version` VARCHAR(32)
- `confidence_score` DECIMAL(5,2)
- `incident_count` INT
- `time_window_start` TIMESTAMP
- `time_window_end` TIMESTAMP
- `primary_ci` VARCHAR(255)
- `primary_assignment_group` VARCHAR(255)
- `created_at` TIMESTAMP

#### `cluster_incidents`
- `cluster_id` UUID FK -> `clusters.id`
- `incident_id` UUID FK -> `incidents.id`
- `membership_score` DECIMAL(5,2)
- PK (`cluster_id`, `incident_id`)

#### `rca_reports`
- `id` UUID PK
- `cluster_id` UUID FK -> `clusters.id`
- `team_id` UUID FK -> `teams.id`
- `status` VARCHAR(32) (Draft/Reviewed/Approved/Sent)
- `executive_summary` TEXT
- `generated_by` VARCHAR(32) (AI/Human/Hybrid)
- `confidence_overall` DECIMAL(5,2)
- `html_path` TEXT
- `pdf_path` TEXT
- `created_at` TIMESTAMP
- `updated_at` TIMESTAMP

#### `why_chains`
- `id` UUID PK
- `rca_report_id` UUID FK -> `rca_reports.id`
- `step_number` INT (1..5)
- `question_text` TEXT
- `answer_text` TEXT
- `evidence_strength` VARCHAR(16) (Strong/Moderate/Weak/Insufficient)
- `confidence_score` DECIMAL(5,2)

#### `why_chain_evidence`
- `why_chain_id` UUID FK -> `why_chains.id`
- `incident_id` UUID FK -> `incidents.id`
- PK (`why_chain_id`, `incident_id`)

#### `mitigation_strategies`
- `id` UUID PK
- `cluster_id` UUID FK -> `clusters.id`
- `rca_report_id` UUID FK -> `rca_reports.id`
- `title` VARCHAR(255)
- `description` TEXT
- `justification` TEXT
- `priority_level` VARCHAR(16) (High/Medium/Low)
- `priority_score` DECIMAL(6,2)
- `effort_estimate` VARCHAR(8) (S/M/L)
- `expected_impact` VARCHAR(16) (High/Medium/Low)
- `risk_notes` TEXT
- `dependencies` TEXT
- `owner_team_id` UUID FK -> `teams.id`
- `target_date` DATE
- `status` VARCHAR(32)

#### `problem_records`
- `id` UUID PK
- `prb_number` VARCHAR(32) UNIQUE
- `cluster_id` UUID FK -> `clusters.id`
- `rca_report_id` UUID FK -> `rca_reports.id`
- `title` VARCHAR(255)
- `description` TEXT
- `status` VARCHAR(32)
- `priority` VARCHAR(16)
- `owner_user_id` UUID FK -> `users.id`
- `opened_at` TIMESTAMP
- `closed_at` TIMESTAMP NULL

#### `problem_tasks`
- `id` UUID PK
- `problem_record_id` UUID FK -> `problem_records.id`
- `title` VARCHAR(255)
- `description` TEXT
- `assigned_to_user_id` UUID FK -> `users.id`
- `due_date` DATE
- `status` VARCHAR(32)
- `progress_percent` INT CHECK 0-100
- `notes` TEXT
- `evidence_link` TEXT

#### `users`
- `id` UUID PK
- `name` VARCHAR(255)
- `email` VARCHAR(255) UNIQUE
- `role` VARCHAR(64)
- `team_id` UUID FK -> `teams.id`
- `is_active` BOOLEAN

#### `teams`
- `id` UUID PK
- `name` VARCHAR(255)
- `manager_user_id` UUID FK -> `users.id`

#### `email_logs`
- `id` UUID PK
- `rca_report_id` UUID FK -> `rca_reports.id`
- `sent_by_user_id` UUID FK -> `users.id`
- `recipient_email` VARCHAR(255)
- `subject` VARCHAR(255)
- `status` VARCHAR(32) (Queued/Sent/Failed/Bounced)
- `provider_message_id` VARCHAR(255)
- `error_message` TEXT
- `sent_at` TIMESTAMP

#### `audit_logs`
- `id` UUID PK
- `entity_type` VARCHAR(64)
- `entity_id` UUID
- `action` VARCHAR(64)
- `before_json` JSON
- `after_json` JSON
- `actor_user_id` UUID FK -> `users.id`
- `acted_at` TIMESTAMP

### Relationships (high level)
- One upload → many incidents.
- Many incidents ↔ many clusters (via cluster_incidents).
- One cluster → many RCA reports (versions) and mitigation strategies.
- One RCA report → many 5-Why steps and evidence mappings.
- One cluster/report → one or more PRBs.
- One PRB → many tasks.
- One report → many email log entries.

---

## 3.3 Analytics & Trend Detection Design

### Pivot-like Aggregation Programmatically
Use pandas/SQL materialized queries:
1. **Group dimensions**: CI, assignment_group, technical_service, business_service, cause, sub_cause, resolution_code.
2. **Metrics**:
   - incident_count
   - distinct CI/service count
   - avg/median resolution time
   - SLA breach count/rate
   - reopened rate (if available)
3. **Time Bucketing**:
   - Daily: DATE(created_at)
   - Weekly: ISO week
   - Monthly: YYYY-MM
4. **Pareto**:
   - Sort categories by incident_count desc.
   - Compute cumulative percentage.
   - Flag categories contributing first 80%.

### Clustering for Repeat Issues
**Input features**
- Text: short_description (cleaned/tokenized).
- Categorical: cause, sub_cause, CI, assignment group.
- Optional: resolution_code, close_notes.

**Approach (MVP)**
1. Sentence embeddings for short_description (+ cause/sub-cause concatenated if present).
2. kNN graph with cosine similarity.
3. Agglomerative clustering (distance threshold) or HDBSCAN for variable cluster size.
4. Post-process rules:
   - Merge clusters sharing same CI + cause + high text similarity.
   - Split clusters with low internal cohesion.

**Duplicate / Near-duplicate rules**
- Duplicate: cosine >= 0.93 AND same CI or same cause/sub-cause.
- Near-duplicate: cosine 0.85–0.93 with at least one matching attribute (CI/group/service/cause).
- Non-duplicate: below thresholds unless rule-based exact keyword signatures match known patterns.

**Confidence scoring**
`confidence = 0.5 * avg_pairwise_similarity + 0.2 * categorical_consistency + 0.2 * temporal_recurrence + 0.1 * sample_size_factor`
- Normalize to 0-100.
- Low confidence (<55) triggers “review required.”

### Anomaly Detection (Spikes)
- Build baseline by CI/group/cause using rolling windows (e.g., last 8 weeks).
- Use robust z-score: `(current - median)/MAD`.
- Alert if z-score > 3 and count >= minimum threshold (e.g., 10 incidents).
- Add seasonality guard (compare same weekday/week-of-month).

---

## 3.4 Dashboard Spec

### Pages
1. **Overview**
2. **Trend Analysis**
3. **Clusters & Repeat Issues**
4. **RCA & Mitigation Workspace**
5. **PRB Tracking Board**
6. **Report & Email Center**

### Global Filters
- Date range
- Service (business/technical)
- CI
- Assignment group
- Priority
- Impact/Urgency
- Cause/sub-cause
- State

### Chart Specifications
1. **Incident Volume Over Time (line chart)**
   - Source: incidents
   - Logic: count by day/week/month with filters
   - Interaction: click point → incident list for interval

2. **Top CIs (bar chart)**
   - Source: incidents
   - Logic: top 10 CIs by count
   - Interaction: click CI → filtered dashboard + incident grid

3. **Top Assignment Groups (bar chart)**
   - Source: incidents
   - Logic: top 10 groups by count and avg resolution duration overlay
   - Interaction: click group → group drilldown

4. **Cause/Sub-cause Heatmap**
   - Source: incidents
   - Logic: matrix count(cause, sub_cause)
   - Interaction: click cell → list incidents + clusters

5. **Recurrence Themes (bubble/scatter)**
   - Source: clusters + cluster_incidents
   - Logic: x=frequency, y=avg severity, bubble=confidence
   - Interaction: click bubble → cluster detail page

6. **Aging Distribution (histogram)**
   - Source: incidents
   - Logic: resolved_at - created_at buckets
   - Interaction: click bucket → incident list

7. **SLA Breach Trend (line + stacked bar)**
   - Source: incidents
   - Logic: breach count/rate by period, split by service/group
   - Interaction: click segment → breach incident table

8. **Pareto Chart by Cause/CI/Group**
   - Source: incidents
   - Logic: bars + cumulative % line sorted desc
   - Interaction: select top contributors to create candidate PRB list

---

## 3.5 AI Components (Prompts + Guardrails)

### 1) Theme/Cluster Summarization Prompt
**System prompt**: “You are an ITSM problem analyst. Use only provided incident evidence. Never infer facts not present.”

**User payload template**
- Cluster metadata: CI, group, services, time window, count, confidence.
- Top incident examples: incident_number, short_description, cause, sub-cause, resolution_code, duration.
- Required output:
  - theme_title
  - plain_summary
  - likely_operational_pattern
  - evidence_incident_numbers[]
  - confidence (0-100)

### 2) 5-Why RCA Generation Prompt
- Input: cluster summary + evidence incidents + timeline + known causes.
- Output JSON with Why1..Why5:
  - `question`
  - `answer`
  - `evidence_incidents`
  - `confidence`
  - `evidence_strength`
- Rule: if fewer than N supporting incidents (e.g., <3) for a why-step, output “Insufficient evidence” and request missing fields (e.g., close notes, CI event logs).

### 3) Mitigation Strategy Generation Prompt
- Input: 5-Why chain + top contributing factors + constraints.
- Output per strategy:
  - action
  - type (preventive/detective/process)
  - justification with cited incident IDs
  - effort (S/M/L)
  - expected_impact
  - dependencies
  - risks
  - confidence

### 4) Executive Summary Prompt
- Audience: managers/service owners.
- Output: concise summary of trend, business effect, top 3 actions, ETA, owner, risk.
- Mandatory evidence references in each paragraph (incident numbers or aggregate counts linked to filters).

### 5) Next Best Actions Prompt
- Audience: problem manager.
- Output prioritized checklist for next 7/30 days:
  - immediate containment
  - data needed
  - stakeholder meetings
  - PRB/task updates
  - monitoring KPI updates

### Guardrails (enforced in code)
1. **Evidence binding**: every conclusion must include incident IDs list.
2. **Insufficient evidence fallback**: return explicit gap message.
3. **No fabrication policy**: if field absent, model cannot assert.
4. **Confidence reporting**: per statement and overall report.
5. **Deterministic schema validation**: LLM output must pass JSON schema.
6. **Human-in-the-loop**: report remains Draft until reviewer approval.
7. **Prompt injection defense**: strip user-provided malicious text from incident fields before prompt construction.

---

## 3.6 Priority Model for Mitigation Strategies

### Scoring Rubric (0–100)
- Frequency score (F): repeat count percentile (0–25)
- Business impact score (B): weighted from impact/urgency/priority + critical service tag (0–30)
- Risk score (R): security/compliance/availability risk (0–20)
- Effort score (E): inverse effort where S=15, M=10, L=5 (0–15)
- Time-to-implement score (T): <2 weeks=10, 2–6 weeks=6, >6 weeks=2 (0–10)

**Formula**
`PriorityScore = F + B + R + E + T`

**Priority mapping**
- High: >= 70
- Medium: 45–69
- Low: <45

### Example
Mitigation: “Standardize DB connection pool settings + alerting”
- F=20 (high recurrence)
- B=24 (high impact + medium urgency)
- R=12 (availability risk moderate)
- E=10 (medium effort)
- T=6 (3 weeks)
- Total = 72 → **High Priority**

---

## 3.7 UX / UI Spec

### 1) Upload Incidents Screen
- Components:
  - Drag/drop area, file picker (CSV/XLSX), mapping preview table.
  - Required fields checklist with green/red indicators.
  - “Validate” then “Ingest” buttons.
- Validations:
  - Missing required columns, invalid dates, duplicate incident_number, oversized files.
- Error states:
  - Inline row-level errors + downloadable error CSV.
- RBAC:
  - Upload allowed for Problem Manager / Service Desk Lead.

### 2) Dashboard Screen
- Components:
  - Global filters left panel.
  - Cards: total incidents, recurrence rate, SLA breach rate.
  - Interactive charts and incident table.
- Interactions:
  - Click-through from chart segment to filtered incident list.

### 3) Cluster Detail View
- Sections:
  - Cluster summary + confidence.
  - Incident evidence table.
  - Similarity explanation (“why grouped”).
  - RCA draft panel + editable 5-Why chain.
  - Mitigation list with scoring.
- Errors:
  - “Insufficient evidence” banner with missing data suggestions.

### 4) RCA Report Preview
- Sections:
  - Executive summary
  - Trend evidence visuals
  - 5-Why
  - Mitigation roadmap
  - Appendix incident evidence
- Actions:
  - Edit, approve, export PDF/HTML.

### 5) Email Sending Screen
- Fields:
  - To/CC/BCC (validated email chips)
  - Subject template
  - Rich-text message
  - Attachments toggles (PDF, HTML, CSV)
- Validation:
  - At least one recipient, valid format, report must be Approved.
- Status:
  - Queue/sent/failure log display.

### 6) PRB Tracking Board
- Kanban columns by PRB status.
- PRB card: owner, due date, risk, linked mitigations, task progress.
- Task side panel: notes, evidence links, updates timeline.
- RBAC:
  - Problem Manager can create/close PRB.
  - Team managers can update tasks/status within permitted transitions.

---

## 3.8 Technical Architecture

### MVP Architecture
- **Frontend**: React (simple, manager-friendly UI with reusable components)
- **Backend**: Python FastAPI
- **Database**: SQLite
- **Analytics**: pandas + scikit-learn + plotly
- **PDF**: HTML-to-PDF (WeasyPrint or wkhtmltopdf wrapper)
- **Email**: SMTP
- **Queue**: lightweight background jobs (RQ/Celery-lite alternative)

### Scalable Production Architecture
- Frontend React + role-aware routing.
- Backend FastAPI microservices (ingest, analytics, AI, reporting, notification).
- Postgres + Redis cache + object storage for exports/reports.
- Worker pool for clustering and report generation.
- Observability: OpenTelemetry + centralized logs.
- Secrets management + SSO (SAML/OAuth2).

### Optional ServiceNow REST Integration
1. Import incidents via table API.
2. Create Problem record (PRB).
3. Create/update Problem tasks.
4. Push PRB status changes back to ServiceNow.

### Textual Component Diagram
1. User uploads file in React UI.
2. FastAPI ingest service validates, normalizes, stores incidents.
3. Analytics engine computes aggregates + clusters.
4. AI service receives evidence-bound payloads, returns structured drafts.
5. Report service renders HTML/PDF and stores artifacts.
6. Notification service emails managers and records logs.
7. PRB module manages lifecycle and tasks; optional sync adaptor updates ServiceNow.

### Data Flow Steps
1. Upload → validation → incidents persisted.
2. Scheduled/on-demand analytics refresh.
3. User opens dashboard → query APIs return filtered aggregates.
4. User selects cluster → RCA/mitigation generated with guardrails.
5. Reviewer approves report → export/send email.
6. PRB created from report → tasks tracked until closure.

---

## 3.9 Implementation Plan

### Phase 1: Upload + Pivot Analytics + Dashboard
- Scope: ingest pipeline, schema, filterable dashboard, basic exports.
- Complexity: **M**
- Risks: messy source data, field mapping variability.

### Phase 2: Clustering + RCA Drafts + Mitigation Suggestions
- Scope: NLP clustering, confidence scoring, AI prompt workflows + guardrails.
- Complexity: **L**
- Risks: false clustering, weak evidence quality, prompt consistency.

### Phase 3: Emailing + Report Generation
- Scope: report templates, PDF/HTML, email workflows + delivery logs.
- Complexity: **M**
- Risks: formatting fidelity, SMTP restrictions, attachment limits.

### Phase 4: PRB Tracking + ServiceNow Integration
- Scope: PRB/task board, status workflows, optional ServiceNow sync.
- Complexity: **L**
- Risks: API auth/governance, process alignment, state mismatch handling.

---

## 3.10 Example Outputs (Dummy Data)

### A) Sample Cluster Summary
- **Cluster**: “Repeated database connection saturation on CI: DB-PROD-03”
- **Window**: 2026-01-01 to 2026-02-15
- **Incidents**: INC0012451, INC0012519, INC0012603, INC0012744, INC0012790
- **Pattern**: 72% occurred during batch window 01:00–03:00 UTC; same assignment group and resolution code “service restart.”
- **Confidence**: 81/100

### B) Sample 5-Why Chain
1. Why did incidents recur?  
   Because application requests failed when DB connection pool was exhausted.  
   Evidence: INC0012451, INC0012519, INC0012744 (confidence 86)
2. Why was pool exhausted?  
   Because batch jobs opened parallel sessions beyond configured pool limits.  
   Evidence: INC0012603, INC0012790 (confidence 78)
3. Why were limits too low for peak load?  
   Because capacity settings were not revised after onboarding new business service traffic.  
   Evidence: INC0012519, INC0012603 (confidence 70)
4. Why was onboarding change not reflected in capacity plan?  
   Insufficient evidence (need change records/capacity review logs).
5. Why is governance gap persisting?  
   Insufficient evidence (need RACI/process audit details).

### C) Three Sample Mitigation Strategies
1. **Implement adaptive DB connection pool + alert threshold at 75% utilization**
   - Priority: High (Score 74)
   - Effort: M
   - Expected Impact: High
   - Justification: Directly addresses repeated saturation pattern seen in 5 linked incidents.
   - Risks/Dependencies: Requires app + DBA change window.

2. **Reschedule non-critical batch jobs to off-peak staggered schedule**
   - Priority: Medium (Score 63)
   - Effort: S
   - Expected Impact: Medium
   - Justification: Reduces concurrent load during known failure window.
   - Risks/Dependencies: Business approval for schedule shifts.

3. **Introduce weekly capacity review checkpoint in CAB handoff**
   - Priority: Medium (Score 58)
   - Effort: S
   - Expected Impact: Medium
   - Justification: Prevents recurrence from unmanaged growth in service demand.
   - Risks/Dependencies: Process adoption and ownership clarity.

### D) Sample Manager-Ready RCA Report Outline
1. Executive Summary
2. Incident Trend Snapshot (volume, services, CI impact)
3. Cluster Theme Details
4. Evidence Table (incident numbers, timestamps, cause/sub-cause)
5. 5-Why Analysis with confidence by step
6. Mitigation Strategy Roadmap (priority, owner, due date)
7. PRB and Task Plan
8. Risks, Dependencies, and Monitoring KPIs
9. Appendix (raw incident CSV extract links)

---

## MVP Success Metrics
- 50% reduction in analyst time to produce monthly proactive problem review.
- 30% faster identification of top recurring issue themes.
- 100% of AI-generated conclusions include evidence citations or “Insufficient evidence.”
- 90%+ email delivery success for approved manager reports.
