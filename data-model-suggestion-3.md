# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A pragmatic hybrid that keeps core entities (employees, clock events, attendance summaries) in fully normalized relational tables while using PostgreSQL JSONB columns for data that is inherently polymorphic, jurisdiction-specific, or evolves faster than the rest of the schema. This gives the best of both worlds: relational integrity where it matters most, and schema flexibility where rigidity would cause constant migrations.

## Why This Suits the Domain

The biometric attendance domain has a stable structural core (organisations have employees who produce clock events on shifts) but significant variability at the edges:

- **Biometric templates** vary by modality -- facial templates carry different metadata than fingerprint or iris templates.
- **Consent requirements** differ by jurisdiction (GDPR needs explicit retention periods and legal basis; BIPA requires specific written notice language; PDPA has different categories).
- **Device integration** data varies wildly between ZKTeco, Suprema, eSSL, and mobile sensors.
- **Anomaly detection** results contain varying attributes depending on the detection algorithm.
- **Payroll connectors** each require different export field mappings.

JSONB columns absorb this variability without schema migrations. GIN indexes on JSONB make the flexible fields queryable at near-relational speed.

The trade-off is that JSONB columns lack the compile-time safety of typed columns, so application-level validation becomes critical. Complex JSONB queries can also be less intuitive than standard SQL.

---

## Schema Definition

```sql
-- =============================================================
-- ORGANISATION AND STRUCTURE
-- =============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_entity    VARCHAR(255),
    timezone        VARCHAR(64) NOT NULL DEFAULT 'UTC',
    data_residency  VARCHAR(16) NOT NULL DEFAULT 'global',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "branding": { "logo_url": "...", "primary_color": "#2563eb" },
    --   "default_work_rules": { "grace_minutes": 5, "rounding": "nearest_15" },
    --   "payroll_connectors": ["adp", "xero"],
    --   "compliance": { "primary_jurisdiction": "gdpr", "retention_years": 3 }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_settings ON organisations USING GIN (settings);

CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    timezone        VARCHAR(64) NOT NULL DEFAULT 'UTC',
    location        JSONB NOT NULL DEFAULT '{}',
    -- location example: {
    --   "address": "123 Main St, London",
    --   "latitude": 51.5074,
    --   "longitude": -0.1278,
    --   "geo_fence_radius_m": 150,
    --   "geo_fence_polygon": [[51.507, -0.128], [51.508, -0.127], ...]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sites_org ON sites(organisation_id);
CREATE INDEX idx_sites_location ON sites USING GIN (location);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    site_id         UUID REFERENCES sites(id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES departments(id) ON DELETE SET NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dept_org ON departments(organisation_id);

-- =============================================================
-- EMPLOYEES
-- =============================================================

CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    department_id   UUID REFERENCES departments(id) ON DELETE SET NULL,
    employee_number VARCHAR(64) NOT NULL,
    first_name      VARCHAR(128) NOT NULL,
    last_name       VARCHAR(128) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(32),
    hire_date       DATE NOT NULL,
    termination_date DATE,
    status          VARCHAR(16) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'suspended', 'terminated')),
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example: {
    --   "job_title": "Software Engineer",
    --   "manager_id": "uuid",
    --   "contract_type": "full_time",
    --   "weekly_hours": 40,
    --   "work_rule_id": "uuid",
    --   "payroll_id": "EMP-12345",
    --   "custom_fields": { "badge_number": "B-789", "floor": "3" }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, employee_number)
);

CREATE INDEX idx_emp_org ON employees(organisation_id);
CREATE INDEX idx_emp_dept ON employees(department_id);
CREATE INDEX idx_emp_status ON employees(status);
CREATE INDEX idx_emp_profile ON employees USING GIN (profile);

-- =============================================================
-- BIOMETRIC CONSENT (jurisdiction-flexible via JSONB)
-- =============================================================

CREATE TABLE biometric_consents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    consent_type    VARCHAR(32) NOT NULL
                    CHECK (consent_type IN ('facial', 'fingerprint', 'palm_vein', 'iris')),
    jurisdiction    VARCHAR(16) NOT NULL,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    deletion_due_at TIMESTAMPTZ,
    consent_details JSONB NOT NULL,
    -- GDPR example: {
    --   "legal_basis": "explicit_consent",
    --   "consent_text_hash": "sha256:abc...",
    --   "data_controller": "Acme Corp",
    --   "dpo_contact": "dpo@acme.com",
    --   "retention_days": 365,
    --   "cross_border_transfer": false,
    --   "purpose_limitation": ["attendance_tracking", "payroll"],
    --   "automated_decision_making": true
    -- }
    -- BIPA example: {
    --   "consent_text_hash": "sha256:def...",
    --   "written_release_obtained": true,
    --   "purpose_disclosure": "Time and attendance tracking",
    --   "retention_schedule": "Until employment ends + 3 years",
    --   "destruction_schedule": "Within 30 days of retention expiry"
    -- }
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_employee ON biometric_consents(employee_id);
CREATE INDEX idx_consent_deletion ON biometric_consents(deletion_due_at)
    WHERE revoked_at IS NULL;
CREATE INDEX idx_consent_details ON biometric_consents USING GIN (consent_details);

-- =============================================================
-- BIOMETRIC TEMPLATES (modality-flexible via JSONB)
-- =============================================================

CREATE TABLE biometric_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    consent_id      UUID NOT NULL REFERENCES biometric_consents(id) ON DELETE CASCADE,
    modality        VARCHAR(32) NOT NULL
                    CHECK (modality IN ('facial_2d', 'facial_3d', 'fingerprint', 'palm_vein', 'iris')),
    template_version INT NOT NULL DEFAULT 1,
    template_blob   BYTEA NOT NULL,
    encryption_key_ref VARCHAR(128) NOT NULL,
    transformation_salt BYTEA NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    revoked_at      TIMESTAMPTZ,
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    template_metadata JSONB NOT NULL DEFAULT '{}',
    -- facial example: {
    --   "quality_score": 0.95,
    --   "liveness_method": "ir_depth",
    --   "face_angle": { "yaw": 2.1, "pitch": -1.3 },
    --   "illumination_score": 0.88,
    --   "algorithm_version": "arcface-v3.2"
    -- }
    -- fingerprint example: {
    --   "quality_score": 0.91,
    --   "finger_position": "right_index",
    --   "sensor_type": "capacitive",
    --   "minutiae_count": 47,
    --   "algorithm_version": "sourceafis-v4.1"
    -- }
    device_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tmpl_employee ON biometric_templates(employee_id);
CREATE INDEX idx_tmpl_active ON biometric_templates(employee_id, modality) WHERE is_active = true;
CREATE INDEX idx_tmpl_metadata ON biometric_templates USING GIN (template_metadata);

-- =============================================================
-- DEVICES (vendor-flexible via JSONB)
-- =============================================================

CREATE TABLE devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    device_type     VARCHAR(32) NOT NULL
                    CHECK (device_type IN ('zkteco', 'suprema', 'essl', 'mobile', 'webcam', 'custom')),
    serial_number   VARCHAR(128),
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(16) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'maintenance', 'offline')),
    last_heartbeat  TIMESTAMPTZ,
    device_key_ref  VARCHAR(128),
    capabilities    JSONB NOT NULL DEFAULT '{}',
    -- zkteco example: {
    --   "firmware_version": "ZK-8.0.3",
    --   "model": "SpeedFace-V5L",
    --   "supported_modalities": ["facial_3d", "palm_vein"],
    --   "max_templates": 50000,
    --   "has_temperature_sensor": true,
    --   "communication_protocol": "push_sdk",
    --   "offline_capacity_events": 100000
    -- }
    -- mobile example: {
    --   "os": "ios",
    --   "os_version": "17.4",
    --   "app_version": "1.2.0",
    --   "has_face_id": true,
    --   "has_fingerprint": true,
    --   "gps_available": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dev_site ON devices(site_id);
CREATE INDEX idx_dev_status ON devices(status);
CREATE INDEX idx_dev_capabilities ON devices USING GIN (capabilities);

ALTER TABLE biometric_templates
    ADD CONSTRAINT fk_tmpl_device
    FOREIGN KEY (device_id) REFERENCES devices(id) ON DELETE SET NULL;

-- =============================================================
-- CLOCK EVENTS
-- =============================================================

CREATE TABLE clock_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    device_id       UUID REFERENCES devices(id) ON DELETE SET NULL,
    event_type      VARCHAR(16) NOT NULL
                    CHECK (event_type IN ('clock_in', 'clock_out', 'break_start', 'break_end')),
    event_time      TIMESTAMPTZ NOT NULL,
    verification_method VARCHAR(32) NOT NULL
                    CHECK (verification_method IN ('facial', 'fingerprint', 'palm_vein', 'iris', 'pin', 'manual', 'mobile_gps')),
    source          VARCHAR(16) NOT NULL DEFAULT 'device'
                    CHECK (source IN ('device', 'mobile', 'web', 'manual', 'import')),
    synced_at       TIMESTAMPTZ,
    verification    JSONB NOT NULL DEFAULT '{}',
    -- example: {
    --   "match_score": 0.97,
    --   "liveness_passed": true,
    --   "liveness_method": "ir_depth",
    --   "anti_spoof_score": 0.99,
    --   "template_id": "uuid",
    --   "location": {
    --     "latitude": 51.5074,
    --     "longitude": -0.1278,
    --     "accuracy_m": 5.2,
    --     "geo_fence_ok": true,
    --     "geo_fence_distance_m": 23.4
    --   },
    --   "device_context": {
    --     "battery_level": 0.85,
    --     "network_type": "wifi"
    --   }
    -- }
    anomaly         JSONB,
    -- example when flagged: {
    --   "is_anomaly": true,
    --   "flags": ["buddy_punch_suspect", "unusual_time"],
    --   "confidence": 0.82,
    --   "related_event_ids": ["uuid1", "uuid2"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_time);

-- Monthly partitions
CREATE TABLE clock_events_2025_01 PARTITION OF clock_events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE clock_events_2025_02 PARTITION OF clock_events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
-- ... additional partitions created by maintenance job

CREATE INDEX idx_clock_emp_time ON clock_events(employee_id, event_time DESC);
CREATE INDEX idx_clock_time ON clock_events(event_time DESC);
CREATE INDEX idx_clock_verification ON clock_events USING GIN (verification);
CREATE INDEX idx_clock_anomaly ON clock_events USING GIN (anomaly)
    WHERE anomaly IS NOT NULL;
CREATE INDEX idx_clock_unsynced ON clock_events(device_id)
    WHERE synced_at IS NULL;

-- =============================================================
-- SHIFT SCHEDULING
-- =============================================================

CREATE TABLE shift_patterns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    description     TEXT,
    rules           JSONB NOT NULL,
    -- example: {
    --   "type": "weekly_rotating",
    --   "rotation_weeks": 2,
    --   "schedules": [
    --     {
    --       "week": 1,
    --       "days": {
    --         "mon": { "start": "09:00", "end": "17:00", "break_mins": 60 },
    --         "tue": { "start": "09:00", "end": "17:00", "break_mins": 60 },
    --         "wed": { "start": "09:00", "end": "17:00", "break_mins": 60 },
    --         "thu": { "start": "09:00", "end": "17:00", "break_mins": 60 },
    --         "fri": { "start": "09:00", "end": "15:00", "break_mins": 30 }
    --       }
    --     },
    --     {
    --       "week": 2,
    --       "days": {
    --         "tue": { "start": "14:00", "end": "22:00", "break_mins": 45 },
    --         "wed": { "start": "14:00", "end": "22:00", "break_mins": 45 },
    --         "thu": { "start": "14:00", "end": "22:00", "break_mins": 45 },
    --         "fri": { "start": "14:00", "end": "22:00", "break_mins": 45 },
    --         "sat": { "start": "10:00", "end": "18:00", "break_mins": 60 }
    --       }
    --     }
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_org ON shift_patterns(organisation_id);
CREATE INDEX idx_shift_rules ON shift_patterns USING GIN (rules);

CREATE TABLE employee_shift_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    pattern_id      UUID NOT NULL REFERENCES shift_patterns(id) ON DELETE CASCADE,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    overrides       JSONB NOT NULL DEFAULT '[]',
    -- example: [
    --   { "date": "2025-03-17", "start": "10:00", "end": "18:00", "reason": "client meeting" },
    --   { "date": "2025-03-21", "off": true, "reason": "training day" }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_asgn_emp ON employee_shift_assignments(employee_id, effective_from);

-- =============================================================
-- WORK RULES
-- =============================================================

CREATE TABLE work_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    rules           JSONB NOT NULL,
    -- example: {
    --   "daily_hours_threshold": 8.0,
    --   "weekly_hours_threshold": 40.0,
    --   "overtime": {
    --     "multiplier": 1.5,
    --     "double_time_after_hours": 12,
    --     "double_time_multiplier": 2.0,
    --     "weekend_multiplier": 2.0
    --   },
    --   "grace_period_minutes": 5,
    --   "rounding_rule": "nearest_15",
    --   "auto_clock_out_after_hours": 16,
    --   "minimum_break_minutes": 30,
    --   "break_after_hours": 6
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_work_rules_org ON work_rules(organisation_id);

-- =============================================================
-- LEAVE MANAGEMENT
-- =============================================================

CREATE TABLE leave_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(64) NOT NULL,
    is_paid         BOOLEAN NOT NULL DEFAULT true,
    policy          JSONB NOT NULL DEFAULT '{}',
    -- example: {
    --   "max_days_per_year": 25,
    --   "accrual_rate_per_month": 2.083,
    --   "carry_forward_max": 5,
    --   "carry_forward_expiry_months": 3,
    --   "min_notice_days": 14,
    --   "requires_documentation": false,
    --   "applicable_after_months": 0
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE leave_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    leave_type_id   UUID NOT NULL REFERENCES leave_types(id) ON DELETE CASCADE,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(16) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'cancelled')),
    details         JSONB NOT NULL DEFAULT '{}',
    -- example: {
    --   "half_day_start": false,
    --   "half_day_end": true,
    --   "reason": "Family event",
    --   "documentation_url": "...",
    --   "reviewed_by": "uuid",
    --   "reviewed_at": "2025-03-15T10:30:00Z",
    --   "rejection_reason": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leave_emp ON leave_requests(employee_id, start_date);
CREATE INDEX idx_leave_status ON leave_requests(status) WHERE status = 'pending';

CREATE TABLE leave_balances (
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    leave_type_id   UUID NOT NULL REFERENCES leave_types(id) ON DELETE CASCADE,
    year            INT NOT NULL,
    balance         JSONB NOT NULL,
    -- example: {
    --   "entitled": 25.0,
    --   "used": 12.5,
    --   "pending": 3.0,
    --   "carried_forward": 4.0,
    --   "adjustments": [
    --     { "date": "2025-06-01", "amount": 1.0, "reason": "Birthday bonus", "by": "uuid" }
    --   ]
    -- }
    PRIMARY KEY (employee_id, leave_type_id, year)
);

-- =============================================================
-- DAILY ATTENDANCE SUMMARY
-- =============================================================

CREATE TABLE daily_attendance (
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    work_date       DATE NOT NULL,
    first_clock_in  TIMESTAMPTZ,
    last_clock_out  TIMESTAMPTZ,
    total_work_mins INT NOT NULL DEFAULT 0,
    total_break_mins INT NOT NULL DEFAULT 0,
    overtime_mins   INT NOT NULL DEFAULT 0,
    status          VARCHAR(16) NOT NULL DEFAULT 'present'
                    CHECK (status IN ('present', 'absent', 'late', 'half_day', 'leave', 'holiday', 'weekend')),
    details         JSONB NOT NULL DEFAULT '{}',
    -- example: {
    --   "exceptions": ["late_arrival"],
    --   "late_by_minutes": 12,
    --   "early_departure_minutes": 0,
    --   "missed_punches": false,
    --   "clock_events": ["uuid1", "uuid2", "uuid3", "uuid4"],
    --   "approved_by": "uuid",
    --   "correction_history": [
    --     { "field": "last_clock_out", "old": "...", "new": "...", "by": "uuid", "at": "..." }
    --   ]
    -- }
    PRIMARY KEY (employee_id, work_date)
);

CREATE INDEX idx_daily_date ON daily_attendance(work_date);

-- =============================================================
-- HOLIDAYS
-- =============================================================

CREATE TABLE holidays (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    site_id         UUID REFERENCES sites(id),
    name            VARCHAR(128) NOT NULL,
    holiday_date    DATE NOT NULL,
    is_optional     BOOLEAN NOT NULL DEFAULT false,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_holidays_org ON holidays(organisation_id, holiday_date);

-- =============================================================
-- PAYROLL EXPORTS
-- =============================================================

CREATE TABLE payroll_exports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    connector_type  VARCHAR(32) NOT NULL,
    status          VARCHAR(16) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    export_config   JSONB NOT NULL DEFAULT '{}',
    -- example: {
    --   "field_mappings": {
    --     "employee_id_field": "payroll_id",
    --     "hours_field": "regular_hours",
    --     "overtime_field": "ot_hours"
    --   },
    --   "filters": { "departments": ["uuid1", "uuid2"] },
    --   "format": "csv",
    --   "file_url": "s3://...",
    --   "record_count": 342,
    --   "error_message": null
    -- }
    exported_by     UUID REFERENCES employees(id),
    exported_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- AUDIT LOG (immutable, with flexible payload)
-- =============================================================

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id        UUID,
    actor_type      VARCHAR(16) NOT NULL DEFAULT 'user',
    action          VARCHAR(64) NOT NULL,
    entity_type     VARCHAR(64) NOT NULL,
    entity_id       UUID,
    changes         JSONB NOT NULL,
    -- example: {
    --   "old": { "status": "active" },
    --   "new": { "status": "suspended" },
    --   "reason": "Policy violation investigation"
    -- }
    context         JSONB NOT NULL DEFAULT '{}',
    -- example: { "ip": "192.168.1.1", "user_agent": "...", "session_id": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (timestamp);

CREATE TABLE audit_log_2025_q1 PARTITION OF audit_log
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
-- ... quarterly partitions

CREATE RULE audit_no_update AS ON UPDATE TO audit_log DO INSTEAD NOTHING;
CREATE RULE audit_no_delete AS ON DELETE TO audit_log DO INSTEAD NOTHING;

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(timestamp DESC);
CREATE INDEX idx_audit_changes ON audit_log USING GIN (changes);

-- =============================================================
-- ANOMALY REPORTS
-- =============================================================

CREATE TABLE anomaly_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    detection_type  VARCHAR(64) NOT NULL,
    severity        VARCHAR(16) NOT NULL
                    CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    status          VARCHAR(16) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'investigating', 'confirmed', 'dismissed')),
    analysis        JSONB NOT NULL,
    -- example: {
    --   "confidence": 0.87,
    --   "description": "Employee clocked in within 30 seconds of another employee at same device, 5 times this week",
    --   "implicated_event_ids": ["uuid1", "uuid2"],
    --   "pattern_details": {
    --     "co_occurring_employee_id": "uuid",
    --     "occurrence_count": 5,
    --     "avg_interval_seconds": 22.4,
    --     "dates": ["2025-03-10", "2025-03-11", "2025-03-12", "2025-03-13", "2025-03-14"]
    --   },
    --   "model_version": "anomaly-v2.3",
    --   "reviewed_by": null,
    --   "reviewed_at": null,
    --   "review_notes": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomaly_emp ON anomaly_reports(employee_id);
CREATE INDEX idx_anomaly_open ON anomaly_reports(severity DESC) WHERE status IN ('open', 'investigating');
CREATE INDEX idx_anomaly_analysis ON anomaly_reports USING GIN (analysis);
```

---

## Trade-offs

**Strengths:**
- Core relational integrity (foreign keys, constraints, typed columns) for the structural backbone.
- JSONB absorbs jurisdiction-specific, vendor-specific, and modality-specific variability without migrations.
- GIN indexes on JSONB columns provide performant querying on flexible fields.
- Easier to add new biometric modalities, device types, or jurisdiction rules without DDL changes.
- Partitioned clock_events and audit_log for time-based data lifecycle management.
- Single database technology (PostgreSQL) simplifies deployment and operations.

**Weaknesses:**
- JSONB columns are not self-documenting -- the schema of what goes inside them must be maintained in application code or JSON Schema validation.
- Complex JSONB queries (e.g., filtering on nested fields) can be harder to write and debug than standard SQL.
- Type safety for JSONB content relies on application-level validation, not database constraints.
- JSONB storage is slightly less space-efficient than typed columns for fixed-schema data.

## Scalability Considerations

- Declarative partitioning on clock_events and audit_log manages data growth and enables efficient retention deletion.
- JSONB with GIN indexes scales well for mixed query patterns -- both exact-match and containment queries are indexed.
- For very large deployments, consider extracting high-cardinality JSONB fields into materialized views or summary tables.
- PostgreSQL's TOAST mechanism automatically compresses large JSONB values.

## Migration Path

- This is a natural evolution from Suggestion 1: start with a normalized schema and progressively move variable-structure fields into JSONB columns.
- If JSONB complexity becomes a pain point, high-traffic JSONB fields can be "promoted" to dedicated typed columns via ALTER TABLE ADD COLUMN + backfill.
- If strict schema validation is needed, introduce JSON Schema checks via CHECK constraints or application-level middleware.
- The JSONB audit_log.changes column is a stepping stone toward full event sourcing (Suggestion 2) if complete replay capability is later desired.
