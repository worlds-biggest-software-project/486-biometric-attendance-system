# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity gets its own table with strict foreign keys, check constraints, and appropriate indexes. This is the most widely understood pattern for enterprise workforce-management systems and maps directly onto the domain's inherent structure: organisations contain departments, departments contain employees, employees produce clock events governed by shift schedules.

## Why This Suits the Domain

Biometric attendance is fundamentally transactional. Clock events must be atomic, consistent, and durable. Payroll calculations, overtime rules, and compliance audits all demand referential integrity that relational databases enforce at the engine level. PostgreSQL's row-level security can enforce data-residency rules (EU employees' data visible only to EU-region queries), and its mature ecosystem of backup, replication, and monitoring tools aligns with self-hosted enterprise deployment requirements.

The main trade-off is rigidity: schema changes require migrations, and deeply nested or polymorphic data (e.g., varying biometric modalities, jurisdiction-specific consent fields) can lead to wide tables or excessive joins.

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
    data_residency  VARCHAR(16) NOT NULL DEFAULT 'global',  -- 'eu', 'us', 'global'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    address         TEXT,
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    geo_fence_radius_m INT NOT NULL DEFAULT 100,
    timezone        VARCHAR(64) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sites_organisation ON sites(organisation_id);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    site_id         UUID REFERENCES sites(id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES departments(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_departments_org ON departments(organisation_id);
CREATE INDEX idx_departments_parent ON departments(parent_id);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, employee_number)
);

CREATE INDEX idx_employees_org ON employees(organisation_id);
CREATE INDEX idx_employees_dept ON employees(department_id);
CREATE INDEX idx_employees_status ON employees(status);

-- =============================================================
-- BIOMETRIC ENROLMENT AND TEMPLATES
-- =============================================================

CREATE TABLE biometric_consents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    consent_type    VARCHAR(32) NOT NULL
                    CHECK (consent_type IN ('facial', 'fingerprint', 'palm_vein', 'iris')),
    jurisdiction    VARCHAR(16) NOT NULL DEFAULT 'default',  -- 'gdpr', 'bipa', 'pdpa', 'popia'
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ,
    retention_days  INT NOT NULL DEFAULT 365,
    deletion_due_at TIMESTAMPTZ,
    consent_text_hash VARCHAR(64) NOT NULL,  -- SHA-256 of the consent text shown
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consents_employee ON biometric_consents(employee_id);
CREATE INDEX idx_consents_deletion ON biometric_consents(deletion_due_at)
    WHERE revoked_at IS NULL;

CREATE TABLE biometric_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    consent_id      UUID NOT NULL REFERENCES biometric_consents(id) ON DELETE CASCADE,
    modality        VARCHAR(32) NOT NULL
                    CHECK (modality IN ('facial_2d', 'facial_3d', 'fingerprint', 'palm_vein', 'iris')),
    template_version INT NOT NULL DEFAULT 1,
    template_blob   BYTEA NOT NULL,       -- encrypted, irreversible transformed template
    encryption_key_ref VARCHAR(128) NOT NULL,  -- reference to per-device/per-org key in vault
    transformation_salt BYTEA NOT NULL,    -- for cancellable biometrics re-enrolment
    quality_score   DECIMAL(5, 2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    revoked_at      TIMESTAMPTZ,
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    device_id       UUID,                  -- device where enrolment occurred
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_templates_employee ON biometric_templates(employee_id);
CREATE INDEX idx_templates_active ON biometric_templates(employee_id, modality)
    WHERE is_active = true;

-- =============================================================
-- DEVICES
-- =============================================================

CREATE TABLE devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    site_id         UUID NOT NULL REFERENCES sites(id) ON DELETE CASCADE,
    device_type     VARCHAR(32) NOT NULL
                    CHECK (device_type IN ('zkteco', 'suprema', 'essl', 'mobile', 'webcam', 'custom')),
    serial_number   VARCHAR(128),
    firmware_version VARCHAR(64),
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(16) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'maintenance', 'offline')),
    last_heartbeat  TIMESTAMPTZ,
    device_key_ref  VARCHAR(128),         -- reference to per-device encryption key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_devices_site ON devices(site_id);
CREATE INDEX idx_devices_status ON devices(status);

-- Add FK now that devices table exists
ALTER TABLE biometric_templates
    ADD CONSTRAINT fk_templates_device
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
    biometric_match_score DECIMAL(5, 2),
    verification_method VARCHAR(32) NOT NULL
                    CHECK (verification_method IN ('facial', 'fingerprint', 'palm_vein', 'iris', 'pin', 'manual', 'mobile_gps')),
    liveness_passed BOOLEAN NOT NULL DEFAULT true,
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    geo_fence_ok    BOOLEAN,
    source          VARCHAR(16) NOT NULL DEFAULT 'device'
                    CHECK (source IN ('device', 'mobile', 'web', 'manual', 'import')),
    synced_at       TIMESTAMPTZ,          -- NULL until offline device syncs
    is_anomaly      BOOLEAN NOT NULL DEFAULT false,
    anomaly_flags   TEXT[],               -- e.g. {'buddy_punch_suspect', 'geo_mismatch'}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_clock_employee_time ON clock_events(employee_id, event_time DESC);
CREATE INDEX idx_clock_event_time ON clock_events(event_time DESC);
CREATE INDEX idx_clock_anomalies ON clock_events(employee_id)
    WHERE is_anomaly = true;
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE shift_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pattern_id      UUID NOT NULL REFERENCES shift_patterns(id) ON DELETE CASCADE,
    day_of_week     SMALLINT NOT NULL CHECK (day_of_week BETWEEN 0 AND 6),
    start_time      TIME NOT NULL,
    end_time        TIME NOT NULL,
    break_minutes   INT NOT NULL DEFAULT 0,
    is_overnight    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_rules_pattern ON shift_rules(pattern_id);

CREATE TABLE employee_shift_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    pattern_id      UUID NOT NULL REFERENCES shift_patterns(id) ON DELETE CASCADE,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_assign_employee ON employee_shift_assignments(employee_id, effective_from);

-- =============================================================
-- WORK RULES AND OVERTIME
-- =============================================================

CREATE TABLE work_rules (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id         UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name                    VARCHAR(128) NOT NULL,
    daily_hours_threshold   DECIMAL(4, 2) NOT NULL DEFAULT 8.0,
    weekly_hours_threshold  DECIMAL(5, 2) NOT NULL DEFAULT 40.0,
    overtime_multiplier     DECIMAL(3, 2) NOT NULL DEFAULT 1.5,
    double_time_multiplier  DECIMAL(3, 2) NOT NULL DEFAULT 2.0,
    grace_period_minutes    INT NOT NULL DEFAULT 5,
    rounding_rule           VARCHAR(16) NOT NULL DEFAULT 'nearest_15'
                            CHECK (rounding_rule IN ('none', 'nearest_5', 'nearest_15', 'nearest_30')),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_work_rules_org ON work_rules(organisation_id);

-- =============================================================
-- LEAVE AND PTO
-- =============================================================

CREATE TABLE leave_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(64) NOT NULL,      -- 'annual', 'sick', 'parental', 'unpaid'
    is_paid         BOOLEAN NOT NULL DEFAULT true,
    max_days        INT,
    accrual_rate    DECIMAL(5, 2),             -- days accrued per month
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE leave_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    leave_type_id   UUID NOT NULL REFERENCES leave_types(id) ON DELETE CASCADE,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    half_day_start  BOOLEAN NOT NULL DEFAULT false,
    half_day_end    BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(16) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'cancelled')),
    reason          TEXT,
    reviewed_by     UUID REFERENCES employees(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leave_employee ON leave_requests(employee_id, start_date);
CREATE INDEX idx_leave_status ON leave_requests(status) WHERE status = 'pending';

CREATE TABLE leave_balances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    leave_type_id   UUID NOT NULL REFERENCES leave_types(id) ON DELETE CASCADE,
    year            INT NOT NULL,
    entitled_days   DECIMAL(5, 2) NOT NULL DEFAULT 0,
    used_days       DECIMAL(5, 2) NOT NULL DEFAULT 0,
    carried_forward DECIMAL(5, 2) NOT NULL DEFAULT 0,
    UNIQUE (employee_id, leave_type_id, year)
);

-- =============================================================
-- ATTENDANCE SUMMARY (computed daily)
-- =============================================================

CREATE TABLE daily_attendance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    work_date       DATE NOT NULL,
    first_clock_in  TIMESTAMPTZ,
    last_clock_out  TIMESTAMPTZ,
    total_work_mins INT NOT NULL DEFAULT 0,
    total_break_mins INT NOT NULL DEFAULT 0,
    overtime_mins   INT NOT NULL DEFAULT 0,
    status          VARCHAR(16) NOT NULL DEFAULT 'present'
                    CHECK (status IN ('present', 'absent', 'late', 'half_day', 'leave', 'holiday', 'weekend')),
    is_exception    BOOLEAN NOT NULL DEFAULT false,
    exception_type  VARCHAR(32),           -- 'late_arrival', 'early_departure', 'missed_punch'
    approved_by     UUID REFERENCES employees(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, work_date)
);

CREATE INDEX idx_daily_att_date ON daily_attendance(work_date);
CREATE INDEX idx_daily_att_exceptions ON daily_attendance(employee_id)
    WHERE is_exception = true;

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_holidays_org_date ON holidays(organisation_id, holiday_date);

-- =============================================================
-- PAYROLL EXPORT
-- =============================================================

CREATE TABLE payroll_exports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    connector_type  VARCHAR(32) NOT NULL
                    CHECK (connector_type IN ('adp', 'paychex', 'sap_sf', 'xero', 'quickbooks', 'csv')),
    status          VARCHAR(16) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    file_url        TEXT,
    exported_by     UUID REFERENCES employees(id),
    exported_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =============================================================
-- AUDIT LOG (immutable)
-- =============================================================

CREATE TABLE audit_log (
    id              BIGSERIAL PRIMARY KEY,
    timestamp       TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id        UUID,                  -- NULL for system actions
    actor_type      VARCHAR(16) NOT NULL DEFAULT 'user'
                    CHECK (actor_type IN ('user', 'system', 'device', 'api')),
    action          VARCHAR(64) NOT NULL,
    entity_type     VARCHAR(64) NOT NULL,
    entity_id       UUID,
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    user_agent      TEXT
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_timestamp ON audit_log(timestamp DESC);
CREATE INDEX idx_audit_actor ON audit_log(actor_id);

-- Make audit log append-only via a rule
CREATE RULE audit_no_update AS ON UPDATE TO audit_log DO INSTEAD NOTHING;
CREATE RULE audit_no_delete AS ON DELETE TO audit_log DO INSTEAD NOTHING;

-- =============================================================
-- ANOMALY DETECTION RESULTS
-- =============================================================

CREATE TABLE anomaly_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    detection_type  VARCHAR(64) NOT NULL,  -- 'buddy_punch_cluster', 'overtime_spike', 'schedule_drift'
    severity        VARCHAR(16) NOT NULL
                    CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    confidence      DECIMAL(5, 4) NOT NULL,
    description     TEXT NOT NULL,
    clock_event_ids UUID[] NOT NULL,       -- references to implicated clock events
    status          VARCHAR(16) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'investigating', 'confirmed', 'dismissed')),
    reviewed_by     UUID REFERENCES employees(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomaly_employee ON anomaly_reports(employee_id);
CREATE INDEX idx_anomaly_status ON anomaly_reports(status) WHERE status IN ('open', 'investigating');
```

---

## Trade-offs

**Strengths:**
- Full referential integrity prevents orphaned records and enforces business rules at the database level.
- Well-understood by most development teams; vast tooling ecosystem (ORMs, migration frameworks, monitoring).
- PostgreSQL's row-level security and partitioning capabilities directly address data-residency and retention requirements.
- Straightforward to build payroll-export queries with standard SQL joins.

**Weaknesses:**
- Schema evolution for new biometric modalities or jurisdiction-specific consent fields requires DDL migrations.
- High-volume clock events (thousands of devices, sub-second intervals) may strain write throughput without partitioning.
- The anomaly_flags TEXT[] column on clock_events is a mild denormalization to avoid a join table for a small, bounded set of flags.

## Scalability Considerations

- Partition `clock_events` by month using PostgreSQL declarative partitioning to keep indexes small and enable efficient retention-based deletion.
- Partition `audit_log` similarly, since it is append-only and grows without bound.
- `daily_attendance` acts as a materialized summary, reducing the need to scan raw clock events for dashboards and payroll.
- Read replicas can serve reporting and analytics workloads without impacting transactional writes.

## Migration Path

- Start with this schema for MVP.
- If write volume exceeds single-node capacity, partition clock_events and audit_log.
- If polymorphic biometric metadata becomes unwieldy, consider migrating template metadata to JSONB (see Suggestion 3).
- If full event replay is needed for compliance audits, layer event sourcing on top of the audit_log table (see Suggestion 2).
