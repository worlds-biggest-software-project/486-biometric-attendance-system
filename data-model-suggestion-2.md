# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourcing architecture where every state change is captured as an immutable domain event in an append-only event store. The write side processes commands and emits events; the read side builds projections (materialized views) optimized for specific query patterns. This model uses PostgreSQL as the event store and projection database, though the event store could be swapped for Kafka or EventStoreDB in high-throughput deployments.

## Why This Suits the Domain

Biometric attendance is a natural fit for event sourcing because:

1. **Immutable audit trail is a core requirement.** The README mandates "immutable, timestamped audit log of every clock event, consent action, and administrative change." Event sourcing provides this by construction -- the event log IS the system of record.
2. **Compliance demands provable history.** GDPR, BIPA, and POPIA require demonstrable data lineage. Event sourcing can prove exactly when consent was granted, what template was enrolled, and when deletion occurred.
3. **Offline-first sync maps to event replay.** Devices that queue clock events offline are inherently producing an event stream. Sync is just appending those events to the central store and replaying projections.
4. **Anomaly detection benefits from temporal replay.** ML models can replay event streams to detect patterns across arbitrary time windows without denormalized summary tables.

The trade-off is increased complexity: developers must think in terms of events and projections rather than direct CRUD, eventual consistency between write and read sides requires careful UX design, and replaying large event streams can be slow without snapshotting.

---

## Event Store Schema

```sql
-- =============================================================
-- CORE EVENT STORE
-- =============================================================

CREATE TABLE event_store (
    event_id        BIGSERIAL PRIMARY KEY,
    stream_id       UUID NOT NULL,             -- aggregate root identifier
    stream_type     VARCHAR(64) NOT NULL,       -- 'Employee', 'ClockSession', 'BiometricEnrolment', etc.
    event_type      VARCHAR(128) NOT NULL,      -- 'ClockInRecorded', 'ConsentGranted', etc.
    event_version   INT NOT NULL,               -- position within the stream
    payload         JSONB NOT NULL,             -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',-- correlation_id, causation_id, actor_id, ip
    occurred_at     TIMESTAMPTZ NOT NULL,       -- when the real-world event happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when it was persisted
    UNIQUE (stream_id, event_version)
);

CREATE INDEX idx_events_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_events_type ON event_store(event_type);
CREATE INDEX idx_events_occurred ON event_store(occurred_at DESC);
CREATE INDEX idx_events_recorded ON event_store(recorded_at DESC);

-- Append-only enforcement
CREATE RULE events_no_update AS ON UPDATE TO event_store DO INSTEAD NOTHING;
CREATE RULE events_no_delete AS ON DELETE TO event_store DO INSTEAD NOTHING;

-- =============================================================
-- SNAPSHOTS (for large aggregates)
-- =============================================================

CREATE TABLE event_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(64) NOT NULL,
    snapshot_version INT NOT NULL,             -- event_version at snapshot time
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

---

## Domain Events

### Organisation & Structure Events
```
OrganisationCreated     { name, legal_entity, timezone, data_residency }
SiteAdded               { site_id, name, address, lat, lng, geo_fence_radius }
DepartmentCreated       { dept_id, name, parent_id, site_id }
```

### Employee Lifecycle Events
```
EmployeeRegistered      { employee_id, org_id, dept_id, employee_number, first_name, last_name, hire_date }
EmployeeTransferred     { employee_id, from_dept_id, to_dept_id, effective_date }
EmployeeSuspended       { employee_id, reason, effective_date }
EmployeeTerminated      { employee_id, termination_date }
EmployeeReactivated     { employee_id, effective_date }
```

### Biometric Consent & Enrolment Events
```
ConsentGranted          { consent_id, employee_id, consent_type, jurisdiction, retention_days, consent_text_hash }
ConsentRevoked          { consent_id, employee_id, reason }
TemplateEnrolled        { template_id, employee_id, consent_id, modality, quality_score, device_id }
TemplateRevoked         { template_id, employee_id, reason }
TemplateReEnrolled      { template_id, old_template_id, employee_id, new_transformation_salt }
BiometricDataDeleted    { employee_id, template_ids[], deletion_reason, jurisdiction }
```

### Clock Events (the core business stream)
```
ClockInRecorded         { event_id, employee_id, device_id, timestamp, verification_method, match_score, liveness_passed, lat, lng, geo_fence_ok, source }
ClockOutRecorded        { event_id, employee_id, device_id, timestamp, verification_method, match_score, liveness_passed, lat, lng, geo_fence_ok, source }
BreakStarted            { event_id, employee_id, device_id, timestamp }
BreakEnded              { event_id, employee_id, device_id, timestamp }
ClockEventCorrected     { original_event_id, corrected_by, original_time, corrected_time, reason }
ClockEventInvalidated   { event_id, reason, invalidated_by }
OfflineEventsSynced     { device_id, event_ids[], sync_timestamp }
```

### Scheduling Events
```
ShiftPatternCreated     { pattern_id, org_id, name, rules[] }
ShiftAssigned           { employee_id, pattern_id, effective_from, effective_to }
ShiftOverrideApplied    { employee_id, date, original_pattern_id, override_start, override_end, reason }
```

### Leave Events
```
LeaveRequested          { request_id, employee_id, leave_type, start_date, end_date, reason }
LeaveApproved           { request_id, approved_by }
LeaveRejected           { request_id, rejected_by, reason }
LeaveCancelled          { request_id, cancelled_by, reason }
LeaveBalanceAdjusted    { employee_id, leave_type, year, adjustment, reason }
```

### Anomaly Detection Events
```
AnomalyDetected         { anomaly_id, employee_id, detection_type, severity, confidence, description, implicated_event_ids[] }
AnomalyDismissed        { anomaly_id, dismissed_by, reason }
AnomalyConfirmed        { anomaly_id, confirmed_by, action_taken }
```

### Device Events
```
DeviceRegistered        { device_id, site_id, device_type, serial_number, name }
DeviceHeartbeatReceived { device_id, timestamp, firmware_version }
DeviceWentOffline       { device_id, last_heartbeat }
DeviceDecommissioned    { device_id, reason }
```

### Payroll Events
```
PayrollExportRequested  { export_id, org_id, period_start, period_end, connector_type, requested_by }
PayrollExportCompleted  { export_id, file_url, record_count }
PayrollExportFailed     { export_id, error_message }
```

---

## Command Handlers

```
RecordClockIn(employee_id, device_id, biometric_data, location?)
  -> validates: employee active, template match, liveness check, geo-fence
  -> emits: ClockInRecorded or rejects with reason

GrantBiometricConsent(employee_id, modality, jurisdiction, consent_text)
  -> validates: no active consent for same modality, employee active
  -> emits: ConsentGranted

EnrolBiometricTemplate(employee_id, consent_id, modality, template_data, device_id)
  -> validates: active consent exists, template quality threshold
  -> emits: TemplateEnrolled

RequestLeave(employee_id, leave_type, start_date, end_date)
  -> validates: sufficient balance, no overlapping requests
  -> emits: LeaveRequested

DetectAnomalies(employee_id, time_window)
  -> replays clock events, applies ML model
  -> emits: AnomalyDetected for each finding

SyncOfflineEvents(device_id, events[])
  -> validates: event ordering, no duplicates (idempotency keys)
  -> emits: individual ClockInRecorded/ClockOutRecorded + OfflineEventsSynced
```

---

## Read Projections (CQRS Query Side)

```sql
-- =============================================================
-- PROJECTION: Current Employee State
-- =============================================================

CREATE TABLE proj_employees (
    employee_id     UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    department_id   UUID,
    employee_number VARCHAR(64) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    status          VARCHAR(16) NOT NULL,
    hire_date       DATE NOT NULL,
    termination_date DATE,
    active_modalities TEXT[],             -- ['facial_3d', 'fingerprint']
    consent_status  JSONB,                -- { "facial": "active", "fingerprint": "revoked" }
    last_event_version BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- =============================================================
-- PROJECTION: Daily Attendance Dashboard
-- =============================================================

CREATE TABLE proj_daily_attendance (
    employee_id     UUID NOT NULL,
    work_date       DATE NOT NULL,
    first_in        TIMESTAMPTZ,
    last_out        TIMESTAMPTZ,
    total_work_mins INT NOT NULL DEFAULT 0,
    total_break_mins INT NOT NULL DEFAULT 0,
    overtime_mins   INT NOT NULL DEFAULT 0,
    status          VARCHAR(16) NOT NULL,
    exceptions      TEXT[],
    clock_event_count INT NOT NULL DEFAULT 0,
    last_event_version BIGINT NOT NULL,
    PRIMARY KEY (employee_id, work_date)
);

CREATE INDEX idx_proj_daily_date ON proj_daily_attendance(work_date);

-- =============================================================
-- PROJECTION: Exception Dashboard (for managers)
-- =============================================================

CREATE TABLE proj_exceptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL,
    work_date       DATE NOT NULL,
    exception_type  VARCHAR(32) NOT NULL,
    details         JSONB NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    resolved_by     UUID,
    last_event_version BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_exceptions_unresolved ON proj_exceptions(employee_id)
    WHERE resolved = false;

-- =============================================================
-- PROJECTION: Leave Balances
-- =============================================================

CREATE TABLE proj_leave_balances (
    employee_id     UUID NOT NULL,
    leave_type      VARCHAR(64) NOT NULL,
    year            INT NOT NULL,
    entitled        DECIMAL(5, 2) NOT NULL,
    used            DECIMAL(5, 2) NOT NULL,
    pending         DECIMAL(5, 2) NOT NULL,
    remaining       DECIMAL(5, 2) NOT NULL,
    last_event_version BIGINT NOT NULL,
    PRIMARY KEY (employee_id, leave_type, year)
);

-- =============================================================
-- PROJECTION: Anomaly Feed
-- =============================================================

CREATE TABLE proj_anomalies (
    anomaly_id      UUID PRIMARY KEY,
    employee_id     UUID NOT NULL,
    detection_type  VARCHAR(64) NOT NULL,
    severity        VARCHAR(16) NOT NULL,
    confidence      DECIMAL(5, 4) NOT NULL,
    description     TEXT NOT NULL,
    status          VARCHAR(16) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    last_event_version BIGINT NOT NULL
);

CREATE INDEX idx_proj_anomalies_open ON proj_anomalies(severity DESC)
    WHERE status = 'open';

-- =============================================================
-- PROJECTION: Device Health
-- =============================================================

CREATE TABLE proj_devices (
    device_id       UUID PRIMARY KEY,
    site_id         UUID NOT NULL,
    device_type     VARCHAR(32) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(16) NOT NULL,
    last_heartbeat  TIMESTAMPTZ,
    events_pending_sync INT NOT NULL DEFAULT 0,
    last_event_version BIGINT NOT NULL
);

-- =============================================================
-- PROJECTION: Compliance / Consent Tracker
-- =============================================================

CREATE TABLE proj_consent_status (
    employee_id     UUID NOT NULL,
    modality        VARCHAR(32) NOT NULL,
    jurisdiction    VARCHAR(16) NOT NULL,
    consent_granted_at TIMESTAMPTZ,
    consent_revoked_at TIMESTAMPTZ,
    deletion_due_at TIMESTAMPTZ,
    template_active BOOLEAN NOT NULL DEFAULT false,
    last_event_version BIGINT NOT NULL,
    PRIMARY KEY (employee_id, modality)
);

CREATE INDEX idx_proj_consent_deletion ON proj_consent_status(deletion_due_at)
    WHERE deletion_due_at IS NOT NULL AND template_active = true;
```

---

## Projection Rebuilding

Projections are disposable and can be rebuilt from the event store at any time. A projection rebuilder reads events sequentially and applies them to the projection tables:

```
ProjectionRebuilder:
  1. Truncate the target projection table
  2. Read events from event_store ordered by event_id
  3. For each event, apply the projection handler
  4. Track last_event_version for incremental updates
```

For live operation, a subscription mechanism tails the event_store (via PostgreSQL LISTEN/NOTIFY or a polling cursor) and applies new events to projections in near-real-time.

---

## Trade-offs

**Strengths:**
- Complete, immutable audit trail is built into the architecture. No separate audit mechanism needed.
- Temporal queries ("what was this employee's status on March 15?") are trivially answered by replaying events up to that point.
- Offline sync is natural: devices produce events locally and replay them to the central store.
- Projections can be purpose-built for each UI view, eliminating complex joins.
- GDPR "right to erasure" can be handled via crypto-shredding: encrypt event payloads per-employee, destroy the key to effectively erase data while preserving the event structure.

**Weaknesses:**
- Higher learning curve and development complexity compared to CRUD.
- Eventual consistency between command and query sides means a clock-in may not appear on the dashboard for a few hundred milliseconds.
- Event schema evolution requires versioning and upcasting strategies.
- Rebuilding projections for large event stores can take significant time without snapshots.
- Simple queries (e.g., "is this employee active?") require a projection rather than a single table lookup.

## Scalability Considerations

- Partition event_store by stream_type or by month for large deployments.
- Use snapshots for aggregates with long event histories (e.g., employees with years of clock events).
- Scale read side independently by deploying projection databases on separate infrastructure.
- For very high throughput (10,000+ devices), consider Apache Kafka as the event transport with PostgreSQL projections as consumers.

## Migration Path

- Can be introduced incrementally: start with CRUD for reference data (organisations, sites, departments), event-source only the clock events and consent streams.
- Existing CRUD data can be "bootstrapped" into the event store by generating synthetic creation events.
- If the complexity proves too high, projections can be promoted to the system of record and the event store retained as a read-only audit archive.
