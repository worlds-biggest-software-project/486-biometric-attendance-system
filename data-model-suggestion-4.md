# Data Model Suggestion 4: Time-Series + Relational Hybrid (TimescaleDB on PostgreSQL)

## Approach

A dual-layer architecture using TimescaleDB (a time-series extension for PostgreSQL) for all time-stamped event data and standard PostgreSQL relational tables for reference/configuration data. Clock events, device heartbeats, anomaly scores, and audit entries live in TimescaleDB hypertables with automatic partitioning, compression, and continuous aggregates. Employee records, shift patterns, and organisational structure remain in ordinary relational tables. Because TimescaleDB runs as a PostgreSQL extension, both layers share a single database instance with full SQL compatibility and foreign-key relationships.

## Why This Approach Suits the Domain

Biometric attendance is fundamentally a time-series problem. The system's primary data flow is a continuous stream of timestamped events: clock-ins, clock-outs, break starts, break ends, device heartbeats, liveness checks, and GPS pings. The core analytical queries are temporal: "What are this employee's hours this week?", "Show me late arrivals for the past month", "Detect overtime spikes across Q1". These are precisely the queries that time-series databases optimize for.

Specific advantages for this domain:

1. **Automatic time-based partitioning.** TimescaleDB hypertables automatically partition by time interval. No manual partition management for clock events or audit logs.
2. **Continuous aggregates.** Daily attendance summaries, weekly hour totals, and monthly overtime calculations can be maintained as real-time materialized views that update automatically as new events arrive -- eliminating batch jobs.
3. **Native compression.** Historical attendance data compresses 10-20x with TimescaleDB's columnar compression, dramatically reducing storage costs for multi-year retention requirements.
4. **Time-bucket queries.** Built-in `time_bucket()` function makes it trivial to aggregate events into 15-minute, hourly, daily, or weekly buckets for dashboards and reports.
5. **Retention policies.** Automated data lifecycle management via `drop_chunks()` aligns with GDPR/BIPA retention schedules.
6. **Full PostgreSQL compatibility.** Foreign keys, JOINs, JSONB, GIN indexes, CTEs, window functions -- everything from Suggestions 1 and 3 works unchanged alongside hypertables.

---

## Schema Definition

### Reference Tables (Standard PostgreSQL)

```sql
-- =============================================================
-- ORGANISATION STRUCTURE (standard relational)
-- =============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_entity    VARCHAR(255),
    timezone        VARCHAR(64) NOT NULL DEFAULT 'UTC',
    data_residency  VARCHAR(16) NOT NULL DEFAULT 'global',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    timezone        VARCHAR(64) NOT NULL DEFAULT 'UTC',
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    geo_fence_radius_m INT NOT NULL DEFAULT 100,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sites_org ON sites(organisation_id);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    site_id         UUID REFERENCES sites(id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES departments(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, employee_number)
);

CREATE INDEX idx_emp_org ON employees(organisation_id);
CREATE INDEX idx_emp_dept ON employees(department_id);
CREATE INDEX idx_emp_status ON employees(status);

-- =============================================================
-- BIOMETRIC CONSENT AND TEMPLATES (standard relational)
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
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_emp ON biometric_consents(employee_id);
CREATE INDEX idx_consent_deletion ON biometric_consents(deletion_due_at)
    WHERE revoked_at IS NULL;

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
    device_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tmpl_emp ON biometric_templates(employee_id);
CREATE INDEX idx_tmpl_active ON biometric_templates(employee_id, modality) WHERE is_active = true;

-- =============================================================
-- DEVICES (standard relational)
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
    device_key_ref  VARCHAR(128),
    capabilities    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dev_site ON devices(site_id);

ALTER TABLE biometric_templates
    ADD CONSTRAINT fk_tmpl_device
    FOREIGN KEY (device_id) REFERENCES devices(id) ON DELETE SET NULL;

-- =============================================================
-- SHIFT AND LEAVE (standard relational)
-- =============================================================

CREATE TABLE shift_patterns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    rules           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE employee_shift_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    pattern_id      UUID NOT NULL REFERENCES shift_patterns(id) ON DELETE CASCADE,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_emp ON employee_shift_assignments(employee_id, effective_from);

CREATE TABLE leave_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(64) NOT NULL,
    is_paid         BOOLEAN NOT NULL DEFAULT true,
    policy          JSONB NOT NULL DEFAULT '{}',
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_leave_emp ON leave_requests(employee_id, start_date);

CREATE TABLE leave_balances (
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    leave_type_id   UUID NOT NULL REFERENCES leave_types(id) ON DELETE CASCADE,
    year            INT NOT NULL,
    entitled        DECIMAL(5, 2) NOT NULL DEFAULT 0,
    used            DECIMAL(5, 2) NOT NULL DEFAULT 0,
    carried_forward DECIMAL(5, 2) NOT NULL DEFAULT 0,
    PRIMARY KEY (employee_id, leave_type_id, year)
);

CREATE TABLE work_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(128) NOT NULL,
    rules           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE holidays (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    site_id         UUID REFERENCES sites(id),
    name            VARCHAR(128) NOT NULL,
    holiday_date    DATE NOT NULL,
    is_optional     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payroll_exports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    connector_type  VARCHAR(32) NOT NULL,
    status          VARCHAR(16) NOT NULL DEFAULT 'pending',
    export_config   JSONB NOT NULL DEFAULT '{}',
    exported_by     UUID REFERENCES employees(id),
    exported_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### TimescaleDB Hypertables (Time-Series Data)

```sql
-- =============================================================
-- ENABLE TIMESCALEDB
-- =============================================================

CREATE EXTENSION IF NOT EXISTS timescaledb;

-- =============================================================
-- CLOCK EVENTS HYPERTABLE
-- =============================================================

CREATE TABLE clock_events (
    event_time          TIMESTAMPTZ NOT NULL,
    id                  UUID NOT NULL DEFAULT gen_random_uuid(),
    employee_id         UUID NOT NULL,  -- logical FK to employees
    device_id           UUID,           -- logical FK to devices
    event_type          VARCHAR(16) NOT NULL
                        CHECK (event_type IN ('clock_in', 'clock_out', 'break_start', 'break_end')),
    verification_method VARCHAR(32) NOT NULL,
    source              VARCHAR(16) NOT NULL DEFAULT 'device',
    match_score         DECIMAL(5, 2),
    liveness_passed     BOOLEAN NOT NULL DEFAULT true,
    latitude            DECIMAL(10, 7),
    longitude           DECIMAL(10, 7),
    geo_fence_ok        BOOLEAN,
    synced_at           TIMESTAMPTZ,
    is_anomaly          BOOLEAN NOT NULL DEFAULT false,
    anomaly_flags       TEXT[],
    verification_meta   JSONB,
    PRIMARY KEY (event_time, id)
);

-- Convert to hypertable with weekly chunks
SELECT create_hypertable('clock_events', 'event_time',
    chunk_time_interval => INTERVAL '7 days');

-- Indexes on hypertable
CREATE INDEX idx_clock_emp ON clock_events(employee_id, event_time DESC);
CREATE INDEX idx_clock_device ON clock_events(device_id, event_time DESC);
CREATE INDEX idx_clock_anomaly ON clock_events(employee_id, event_time DESC)
    WHERE is_anomaly = true;
CREATE INDEX idx_clock_unsynced ON clock_events(device_id, event_time DESC)
    WHERE synced_at IS NULL;

-- Enable compression on chunks older than 30 days
ALTER TABLE clock_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'employee_id',
    timescaledb.compress_orderby = 'event_time DESC'
);

SELECT add_compression_policy('clock_events', INTERVAL '30 days');

-- =============================================================
-- DEVICE HEARTBEATS HYPERTABLE
-- =============================================================

CREATE TABLE device_heartbeats (
    heartbeat_time      TIMESTAMPTZ NOT NULL,
    device_id           UUID NOT NULL,
    firmware_version    VARCHAR(64),
    cpu_usage           DECIMAL(5, 2),
    memory_usage        DECIMAL(5, 2),
    disk_usage          DECIMAL(5, 2),
    network_status      VARCHAR(16),
    pending_sync_count  INT NOT NULL DEFAULT 0,
    temperature         DECIMAL(5, 2),
    metrics             JSONB,
    PRIMARY KEY (heartbeat_time, device_id)
);

SELECT create_hypertable('device_heartbeats', 'heartbeat_time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_heartbeat_device ON device_heartbeats(device_id, heartbeat_time DESC);

ALTER TABLE device_heartbeats SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby = 'heartbeat_time DESC'
);

SELECT add_compression_policy('device_heartbeats', INTERVAL '7 days');

-- Retain heartbeats for 90 days only
SELECT add_retention_policy('device_heartbeats', INTERVAL '90 days');

-- =============================================================
-- ANOMALY SCORES HYPERTABLE (ML model output over time)
-- =============================================================

CREATE TABLE anomaly_scores (
    scored_at           TIMESTAMPTZ NOT NULL,
    employee_id         UUID NOT NULL,
    detection_type      VARCHAR(64) NOT NULL,
    score               DECIMAL(5, 4) NOT NULL,
    severity            VARCHAR(16) NOT NULL
                        CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    model_version       VARCHAR(32) NOT NULL,
    details             JSONB NOT NULL,
    -- example: {
    --   "description": "Clock-in time drifting later by avg 2.3 min/week over 6 weeks",
    --   "implicated_event_ids": ["uuid1", "uuid2"],
    --   "baseline_avg_clock_in": "08:55",
    --   "current_avg_clock_in": "09:09",
    --   "trend_slope_minutes_per_week": 2.3
    -- }
    status              VARCHAR(16) NOT NULL DEFAULT 'open'
                        CHECK (status IN ('open', 'investigating', 'confirmed', 'dismissed')),
    reviewed_by         UUID,
    reviewed_at         TIMESTAMPTZ,
    PRIMARY KEY (scored_at, employee_id, detection_type)
);

SELECT create_hypertable('anomaly_scores', 'scored_at',
    chunk_time_interval => INTERVAL '30 days');

CREATE INDEX idx_anomaly_emp ON anomaly_scores(employee_id, scored_at DESC);
CREATE INDEX idx_anomaly_open ON anomaly_scores(scored_at DESC)
    WHERE status IN ('open', 'investigating');

ALTER TABLE anomaly_scores SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'employee_id',
    timescaledb.compress_orderby = 'scored_at DESC'
);

SELECT add_compression_policy('anomaly_scores', INTERVAL '90 days');

-- =============================================================
-- AUDIT LOG HYPERTABLE
-- =============================================================

CREATE TABLE audit_log (
    log_time        TIMESTAMPTZ NOT NULL DEFAULT now(),
    id              BIGINT GENERATED ALWAYS AS IDENTITY,
    actor_id        UUID,
    actor_type      VARCHAR(16) NOT NULL DEFAULT 'user',
    action          VARCHAR(64) NOT NULL,
    entity_type     VARCHAR(64) NOT NULL,
    entity_id       UUID,
    changes         JSONB NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (log_time, id)
);

SELECT create_hypertable('audit_log', 'log_time',
    chunk_time_interval => INTERVAL '30 days');

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id, log_time DESC);
CREATE INDEX idx_audit_actor ON audit_log(actor_id, log_time DESC);

ALTER TABLE audit_log SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'entity_type',
    timescaledb.compress_orderby = 'log_time DESC'
);

SELECT add_compression_policy('audit_log', INTERVAL '90 days');
```

### Continuous Aggregates (Auto-Maintained Materialized Views)

```sql
-- =============================================================
-- CONTINUOUS AGGREGATE: Daily Attendance Summary
-- =============================================================

CREATE MATERIALIZED VIEW daily_attendance_summary
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', event_time) AS work_date,
    employee_id,
    MIN(event_time) FILTER (WHERE event_type = 'clock_in') AS first_clock_in,
    MAX(event_time) FILTER (WHERE event_type = 'clock_out') AS last_clock_out,
    COUNT(*) FILTER (WHERE event_type = 'clock_in') AS clock_in_count,
    COUNT(*) FILTER (WHERE event_type = 'clock_out') AS clock_out_count,
    COUNT(*) FILTER (WHERE event_type = 'break_start') AS break_count,
    BOOL_OR(is_anomaly) AS has_anomalies,
    array_agg(DISTINCT unnest) FILTER (WHERE anomaly_flags IS NOT NULL) AS all_anomaly_flags
FROM clock_events, LATERAL unnest(COALESCE(anomaly_flags, '{}'))
GROUP BY time_bucket('1 day', event_time), employee_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('daily_attendance_summary',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- =============================================================
-- CONTINUOUS AGGREGATE: Hourly Device Activity
-- =============================================================

CREATE MATERIALIZED VIEW hourly_device_activity
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', event_time) AS hour,
    device_id,
    COUNT(*) AS event_count,
    COUNT(DISTINCT employee_id) AS unique_employees,
    AVG(match_score) AS avg_match_score,
    COUNT(*) FILTER (WHERE NOT liveness_passed) AS liveness_failures,
    COUNT(*) FILTER (WHERE is_anomaly) AS anomaly_count
FROM clock_events
GROUP BY time_bucket('1 hour', event_time), device_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('hourly_device_activity',
    start_offset    => INTERVAL '2 hours',
    end_offset      => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes');

-- =============================================================
-- CONTINUOUS AGGREGATE: Weekly Employee Hours
-- =============================================================

CREATE MATERIALIZED VIEW weekly_employee_hours
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('7 days', event_time) AS week_start,
    employee_id,
    COUNT(*) FILTER (WHERE event_type = 'clock_in') AS days_present,
    MIN(event_time) FILTER (WHERE event_type = 'clock_in') AS earliest_clock_in,
    MAX(event_time) FILTER (WHERE event_type = 'clock_out') AS latest_clock_out,
    COUNT(*) FILTER (WHERE is_anomaly) AS anomaly_count
FROM clock_events
GROUP BY time_bucket('7 days', event_time), employee_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('weekly_employee_hours',
    start_offset    => INTERVAL '14 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');

-- =============================================================
-- CONTINUOUS AGGREGATE: Device Health Summary
-- =============================================================

CREATE MATERIALIZED VIEW device_health_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', heartbeat_time) AS hour,
    device_id,
    AVG(cpu_usage) AS avg_cpu,
    MAX(cpu_usage) AS max_cpu,
    AVG(memory_usage) AS avg_memory,
    MAX(pending_sync_count) AS max_pending_sync,
    COUNT(*) AS heartbeat_count
FROM device_heartbeats
GROUP BY time_bucket('1 hour', heartbeat_time), device_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('device_health_hourly',
    start_offset    => INTERVAL '3 hours',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

### Retention Policies (GDPR/BIPA Compliance)

```sql
-- Automatically drop clock event chunks older than the configured retention
-- (adjust per jurisdiction; this is a default)
SELECT add_retention_policy('clock_events', INTERVAL '7 years');

-- Audit log: retain for regulatory minimum (often 7+ years)
SELECT add_retention_policy('audit_log', INTERVAL '10 years');

-- Anomaly scores: retain for 3 years
SELECT add_retention_policy('anomaly_scores', INTERVAL '3 years');

-- Device heartbeats: already set to 90 days above

-- For GDPR per-employee deletion, use targeted DELETE + recompress:
-- DELETE FROM clock_events WHERE employee_id = $1;
-- (TimescaleDB handles chunk-level operations efficiently)
```

---

## Example Analytical Queries

```sql
-- Employees with rising average clock-in times (schedule drift detection)
SELECT
    employee_id,
    time_bucket('7 days', event_time) AS week,
    AVG(EXTRACT(EPOCH FROM event_time::time)) / 60 AS avg_clock_in_minutes
FROM clock_events
WHERE event_type = 'clock_in'
  AND event_time > now() - INTERVAL '12 weeks'
GROUP BY employee_id, time_bucket('7 days', event_time)
ORDER BY employee_id, week;

-- Department-level absence rates by month
SELECT
    d.name AS department,
    time_bucket('1 month', das.work_date) AS month,
    COUNT(DISTINCT das.employee_id) AS employees_tracked,
    COUNT(*) FILTER (WHERE das.clock_in_count = 0) AS absent_days,
    ROUND(COUNT(*) FILTER (WHERE das.clock_in_count = 0)::decimal /
          NULLIF(COUNT(*), 0) * 100, 1) AS absence_rate_pct
FROM daily_attendance_summary das
JOIN employees e ON e.id = das.employee_id
JOIN departments d ON d.id = e.department_id
GROUP BY d.name, time_bucket('1 month', das.work_date)
ORDER BY month DESC, absence_rate_pct DESC;

-- Devices with declining match scores (hardware degradation)
SELECT
    d.name AS device_name,
    h.hour,
    h.avg_match_score,
    h.liveness_failures
FROM hourly_device_activity h
JOIN devices d ON d.id = h.device_id
WHERE h.hour > now() - INTERVAL '30 days'
  AND h.avg_match_score < 0.85
ORDER BY h.avg_match_score ASC;
```

---

## Trade-offs

**Strengths:**
- Purpose-built for the dominant data pattern: timestamped events arriving continuously.
- Automatic chunking eliminates manual partition management entirely.
- Native compression reduces storage 10-20x for historical data, critical for multi-year retention compliance.
- Continuous aggregates provide real-time dashboards without batch ETL jobs or cron-based summary computations.
- Built-in retention policies directly implement GDPR/BIPA deletion schedules at the infrastructure level.
- Full PostgreSQL compatibility means all relational features (FKs, JOINs, CTEs, JSONB) work unchanged.
- The `time_bucket()` function and window operations are optimized for the exact query patterns HR dashboards need.

**Weaknesses:**
- TimescaleDB is an additional dependency (PostgreSQL extension). Self-hosted deployments must install and maintain it.
- Hypertable primary keys must include the time column, which changes some access patterns (lookups by UUID alone require an index scan across chunks).
- Foreign keys from hypertables to regular tables are supported, but FKs TO hypertables are not. Clock events reference employees via logical FK (application-enforced), not database-enforced FK.
- Continuous aggregates have refresh latency (configurable, typically minutes), so real-time dashboards may show slightly stale summaries.
- Compressed chunks are read-only; updates to historical events require decompression first.

## Scalability Considerations

- TimescaleDB handles billions of rows efficiently via chunk-level operations. A deployment with 10,000 employees and 3 years of data (roughly 30M clock events) would perform well on a single node.
- For larger deployments, TimescaleDB supports multi-node distributed hypertables that shard across multiple PostgreSQL instances.
- Compression ratios of 10-20x mean that 30M clock events (~3 GB uncompressed) compress to ~200-300 MB.
- Continuous aggregates offload dashboard queries from the raw event data, keeping read performance consistent as data grows.

## Migration Path

- Start with Suggestion 1 or 3 on plain PostgreSQL. When time-series query performance or storage becomes a concern, install TimescaleDB and convert the clock_events table to a hypertable with a single command: `SELECT create_hypertable('clock_events', 'event_time', migrate_data => true);`.
- Continuous aggregates can be added incrementally for specific dashboard views without changing the application's write path.
- If TimescaleDB proves unnecessary (small deployment, low data volume), the schema works on plain PostgreSQL with manual partitioning -- the only TimescaleDB-specific features are the hypertable conversion, compression policies, continuous aggregates, and retention policies.
- For organisations that want a fully managed time-series backend, the same schema works on Timescale Cloud with no application changes.
