# VitalsDrive - Data Storage Layer PRD

**Document Version:** 1.0  
**Last Updated:** 2026-03-29  
**Layer:** L2 - Data Storage (Supabase PostgreSQL)  
**Status:** Draft for Review

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Database Schema](#2-database-schema)
3. [Additional Tables](#3-additional-tables)
4. [Supabase Realtime Configuration](#4-supabase-realtime-configuration)
5. [Row Level Security (RLS) Policies](#5-row-level-security-rls-policies)
6. [API Design](#6-api-design)
7. [Database Functions and Triggers](#7-database-functions-and-triggers)
8. [Indexing Strategy](#8-indexing-strategy)
9. [Backup and Recovery](#9-backup-and-recovery)
10. [Migration Strategy](#10-migration-strategy)
11. [Monitoring](#11-monitoring)
12. [Scaling Considerations](#12-scaling-considerations)
13. [Cost Estimation](#13-cost-estimation)
14. [Testing Strategy](#14-testing-strategy)

---

## 1. Executive Summary

### Purpose
The Data Storage Layer provides the persistence foundation for VitalsDrive's fleet health monitoring platform, leveraging Supabase PostgreSQL with realtime capabilities for real-time telemetry ingestion and WebSocket distribution to Angular clients.

### Scope
- Primary data store for vehicle telemetry logs
- User and fleet management
- Alert/notification persistence
- Realtime pub/sub for live telemetry updates

### Goals
- **Reliability:** 99.9% uptime with automated backups
- **Performance:** Sub-100ms read latency for dashboard queries
- **Security:** Tenant isolation via Row Level Security
- **Cost Efficiency:** Stay within Supabase free tier for MVP

### Key Decisions
| Decision | Choice | Rationale |
|----------|--------|-----------|
| Database | Supabase PostgreSQL | Built-in auth, realtime, RLS |
| Realtime | Postgres Changes | Native Supabase feature |
| Multi-tenancy | RLS policies | Row-level isolation without separate DBs |
| Backup | Supabase managed | Daily automated, point-in-time recovery |

---

## 2. Database Schema

### 2.1 Core Tables

#### telemetry_logs
```sql
CREATE TABLE telemetry_logs (
  id          BIGSERIAL PRIMARY KEY,
  vehicle_id  UUID NOT NULL,
  lat         DECIMAL(10, 7) CHECK (lat >= -90 AND lat <= 90),
  lng         DECIMAL(10, 7) CHECK (lng >= -180 AND lng <= 180),
  temp        FLOAT,           -- Coolant temperature (°C)
  voltage     FLOAT,           -- Battery voltage (V)
  rpm         INTEGER CHECK (rpm >= 0),
  dtc_codes   TEXT[],          -- Diagnostic trouble codes
  timestamp   TIMESTAMPTZ DEFAULT NOW() NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE telemetry_logs IS 'Vehicle telemetry data points ingested from IoT sensors';
COMMENT ON COLUMN telemetry_logs.dtc_codes IS 'Array of active diagnostic trouble codes, empty array means no faults';
```

#### vehicles
```sql
CREATE TABLE vehicles (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  fleet_id      UUID NOT NULL,
  vin           VARCHAR(17) UNIQUE NOT NULL,  -- Vehicle Identification Number
  make          VARCHAR(50) NOT NULL,
  model         VARCHAR(50) NOT NULL,
  year          INTEGER,
  license_plate VARCHAR(20),
  status        VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'maintenance')),
  metadata      JSONB DEFAULT '{}',
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE vehicles IS 'Registered vehicles belonging to fleets';
```

#### fleets
```sql
CREATE TABLE fleets (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        VARCHAR(100) NOT NULL,
  owner_id    UUID NOT NULL,  -- References auth.users
  settings    JSONB DEFAULT '{"timezone": "UTC", "data_retention_days": 90}',
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE fleets IS 'Fleet groupings for organizing vehicles';
```

#### users
```sql
CREATE TABLE users (
  id            UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email         VARCHAR(255) NOT NULL,
  display_name  VARCHAR(100),
  role          VARCHAR(20) DEFAULT 'viewer' CHECK (role IN ('admin', 'editor', 'viewer')),
  preferences   JSONB DEFAULT '{"theme": "dark", "notifications": true}',
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  last_login    TIMESTAMPTZ
);

COMMENT ON TABLE users IS 'Application users with role-based access';
```

#### fleet_members
```sql
CREATE TABLE fleet_members (
  fleet_id    UUID REFERENCES fleets(id) ON DELETE CASCADE,
  user_id     UUID REFERENCES users(id) ON DELETE CASCADE,
  role        VARCHAR(20) DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
  joined_at   TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (fleet_id, user_id)
);

COMMENT ON TABLE fleet_members IS 'Junction table for user-fleet many-to-many relationship';
```

#### alerts
```sql
CREATE TABLE alerts (
  id            BIGSERIAL PRIMARY KEY,
  vehicle_id    UUID NOT NULL REFERENCES vehicles(id) ON DELETE CASCADE,
  fleet_id      UUID NOT NULL REFERENCES fleets(id) ON DELETE CASCADE,
  severity      VARCHAR(10) NOT NULL CHECK (severity IN ('critical', 'warning', 'info')),
  code          VARCHAR(50) NOT NULL,  -- e.g., 'ENGINE_OVERTEMP', 'LOW_VOLTAGE'
  message       TEXT NOT NULL,
  dtc_codes     TEXT[],               -- Related diagnostic codes
  lat           DECIMAL(10, 7),
  lng           DECIMAL(10, 7),
  acknowledged  BOOLEAN DEFAULT FALSE,
  acknowledged_by UUID REFERENCES users(id),
  acknowledged_at TIMESTAMPTZ,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  resolved_at   TIMESTAMPTZ
);

COMMENT ON TABLE alerts IS 'Generated alerts from telemetry threshold violations';
```

#### telemetry_rules
```sql
CREATE TABLE telemetry_rules (
  id          BIGSERIAL PRIMARY KEY,
  fleet_id    UUID NOT NULL REFERENCES fleets(id) ON DELETE CASCADE,
  name        VARCHAR(100) NOT NULL,
  metric      VARCHAR(20) NOT NULL CHECK (metric IN ('temp', 'voltage', 'rpm')),
  operator    VARCHAR(5) NOT NULL CHECK (operator IN ('gt', 'lt', 'gte', 'lte', 'eq')),
  threshold   FLOAT NOT NULL,
  severity    VARCHAR(10) NOT NULL CHECK (severity IN ('critical', 'warning', 'info')),
  enabled     BOOLEAN DEFAULT TRUE,
  cooldown_seconds INTEGER DEFAULT 300,  -- Minimum time between repeated alerts
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE telemetry_rules IS 'User-defined threshold rules for generating alerts';
```

---

## 3. Additional Tables

### 3.1 audit_logs (Optional - for compliance)
```sql
CREATE TABLE audit_logs (
  id          BIGSERIAL PRIMARY KEY,
  user_id     UUID,
  action      VARCHAR(50) NOT NULL,
  table_name  VARCHAR(50),
  record_id   UUID,
  old_data    JSONB,
  new_data    JSONB,
  ip_address  INET,
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE audit_logs IS 'Audit trail for compliance and debugging';
```

### 3.2 scheduled_maintenance
```sql
CREATE TABLE scheduled_maintenance (
  id            BIGSERIAL PRIMARY KEY,
  vehicle_id    UUID NOT NULL REFERENCES vehicles(id) ON DELETE CASCADE,
  type          VARCHAR(50) NOT NULL,  -- e.g., 'oil_change', 'tire_rotation', 'inspection'
  description   TEXT,
  due_date      DATE NOT NULL,
  due_mileage   INTEGER,
  completed     BOOLEAN DEFAULT FALSE,
  completed_at  TIMESTAMPTZ,
  cost          DECIMAL(10, 2),
  notes         TEXT,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

COMMENT ON TABLE scheduled_maintenance IS 'Vehicle maintenance schedule tracking';
```

---

## 4. Supabase Realtime Configuration

### 4.1 Enable Realtime on telemetry_logs

```sql
-- Enable realtime for telemetry_logs table via Supabase dashboard or migration:
ALTER PUBLICATION supabase_realtime ADD TABLE telemetry_logs;
ALTER PUBLICATION supabase_realtime ADD TABLE alerts;
```

### 4.2 Realtime Filter Configuration

```javascript
// Angular client subscription pattern
const channel = supabase
  .channel('telemetry-live')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'telemetry_logs',
      filter: 'vehicle_id=eq.' + vehicleId  // Filter by specific vehicle
    },
    (payload) => {
      console.log('New telemetry:', payload.new);
      this.onTelemetry.next(payload.new);
    }
  )
  .subscribe();

// Multiple vehicle subscription
const channel = supabase
  .channel('fleet-telemetry')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'telemetry_logs'
      // No filter - receive all, filter client-side for user's fleet
    },
    (payload) => {
      this.handleTelemetry(payload.new);
    }
  )
  .subscribe();
```

### 4.3 Realtime Best Practices

| Practice | Implementation |
|----------|----------------|
| Channel cleanup | `supabase.removeChannel(channel)` on component destroy |
| Reconnection | Built-in Supabase handle with exponential backoff |
| Subscription limits | Max 10 concurrent subscriptions per client |
| Payload size | Keep payloads < 1KB for optimal performance |

---

## 5. Row Level Security (RLS) Policies

### 5.1 Enable RLS on All Tables

```sql
-- Enable RLS
ALTER TABLE telemetry_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE vehicles ENABLE ROW LEVEL SECURITY;
ALTER TABLE fleets ENABLE ROW LEVEL SECURITY;
ALTER TABLE fleet_members ENABLE ROW LEVEL SECURITY;
ALTER TABLE alerts ENABLE ROW LEVEL SECURITY;
ALTER TABLE telemetry_rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
```

### 5.2 Helper Function for Fleet Access

```sql
CREATE OR REPLACE FUNCTION get_user_fleet_ids()
RETURNS SETOF UUID AS $$
  SELECT fleet_id FROM fleet_members WHERE user_id = auth.uid();
$$ LANGUAGE SQL SECURITY DEFINER STABLE;

COMMENT ON FUNCTION get_user_fleet_ids IS 'Returns all fleet IDs the current user has access to';
```

### 5.3 RLS Policies

#### telemetry_logs
```sql
-- Users can only read telemetry for vehicles in their fleets
CREATE POLICY "Users can read own fleet telemetry"
  ON telemetry_logs FOR SELECT
  USING (
    vehicle_id IN (
      SELECT v.id FROM vehicles v
      JOIN fleets f ON v.fleet_id = f.id
      WHERE f.id IN (SELECT get_user_fleet_ids())
    )
  );

-- Only service accounts can insert telemetry (managed via service role key)
CREATE POLICY "Service accounts can insert telemetry"
  ON telemetry_logs FOR INSERT
  WITH CHECK (true);  -- Service role bypasses RLS anyway
```

#### vehicles
```sql
-- Users can read vehicles in their fleets
CREATE POLICY "Users can read own fleet vehicles"
  ON vehicles FOR SELECT
  USING (fleet_id IN (SELECT get_user_fleet_ids()));

-- Only admins can modify vehicles
CREATE POLICY "Admins can manage vehicles"
  ON vehicles FOR ALL
  USING (
    fleet_id IN (
      SELECT fleet_id FROM fleet_members
      WHERE user_id = auth.uid() AND role IN ('owner', 'admin')
    )
  );
```

#### fleets
```sql
-- Users can read fleets they are members of
CREATE POLICY "Users can read own fleets"
  ON fleets FOR SELECT
  USING (id IN (SELECT get_user_fleet_ids()));

-- Only owners can delete fleets
CREATE POLICY "Owners can delete fleets"
  ON fleets FOR DELETE
  USING (owner_id = auth.uid());

-- Fleet owners and admins can update
CREATE POLICY "Owners and admins can update fleets"
  ON fleets FOR UPDATE
  USING (
    id IN (
      SELECT fleet_id FROM fleet_members
      WHERE user_id = auth.uid() AND role IN ('owner', 'admin')
    )
  );
```

#### alerts
```sql
-- Users can read alerts for their fleets
CREATE POLICY "Users can read own fleet alerts"
  ON alerts FOR SELECT
  USING (fleet_id IN (SELECT get_user_fleet_ids()));

-- Fleet admins and owners can acknowledge alerts
CREATE POLICY "Admins can acknowledge alerts"
  ON alerts FOR UPDATE
  USING (
    fleet_id IN (
      SELECT fleet_id FROM fleet_members
      WHERE user_id = auth.uid() AND role IN ('owner', 'admin')
    )
  );

-- System can create alerts (bypassed by service role)
CREATE POLICY "System can create alerts"
  ON alerts FOR INSERT
  WITH CHECK (true);
```

#### users
```sql
-- Users can read their own profile
CREATE POLICY "Users can read own profile"
  ON users FOR SELECT
  USING (id = auth.uid());

-- Users can update their own profile (except role)
CREATE POLICY "Users can update own profile"
  ON users FOR UPDATE
  USING (id = auth.uid())
  WITH CHECK (id = auth.uid() AND role = (SELECT role FROM users WHERE id = auth.uid()));
```

### 5.4 RLS Performance Note

```sql
-- Create index to speed up fleet-based RLS lookups
CREATE INDEX idx_fleet_members_user_id ON fleet_members(user_id);
CREATE INDEX idx_vehicles_fleet_id ON vehicles(fleet_id);
```

---

## 6. API Design

### 6.1 Supabase Client Initialization (Angular)

```typescript
// environments/environment.ts
export const environment = {
  supabaseUrl: 'https://your-project.supabase.co',
  supabaseAnonKey: 'your-anon-key'
};

// services/supabase.service.ts
import { createClient } from '@supabase/supabase-js';
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class SupabaseService {
  private supabase = createClient(
    environment.supabaseUrl,
    environment.supabaseAnonKey,
    {
      auth: {
        persistSession: true,
        autoRefreshToken: true
      },
      realtime: {
        params: { eventsPerSecond: 10 }
      }
    }
  );

  get client() {
    return this.supabase;
  }
}
```

### 6.2 Telemetry CRUD Operations

```typescript
// TelemetryService
@Injectable({ providedIn: 'root' })
export class TelemetryService {
  constructor(private supabase: SupabaseService) {}

  // Insert telemetry (from IoT gateway)
  async ingestTelemetry(data: TelemetryPayload): Promise<void> {
    const { error } = await this.supabase.client
      .from('telemetry_logs')
      .insert({
        vehicle_id: data.vehicleId,
        lat: data.latitude,
        lng: data.longitude,
        temp: data.coolantTemp,
        voltage: data.batteryVoltage,
        rpm: data.rpm,
        dtc_codes: data.dtcCodes || []
      });
    
    if (error) throw new Error(`Telemetry ingestion failed: ${error.message}`);
  }

  // Get latest telemetry for a vehicle
  async getLatestTelemetry(vehicleId: string): Promise<TelemetryLog | null> {
    const { data, error } = await this.supabase.client
      .from('telemetry_logs')
      .select('*')
      .eq('vehicle_id', vehicleId)
      .order('timestamp', { ascending: false })
      .limit(1)
      .single();
    
    if (error && error.code !== 'PGRST116') throw error;
    return data;
  }

  // Get telemetry history for a vehicle
  async getTelemetryHistory(
    vehicleId: string,
    startTime: Date,
    endTime: Date
  ): Promise<TelemetryLog[]> {
    const { data, error } = await this.supabase.client
      .from('telemetry_logs')
      .select('*')
      .eq('vehicle_id', vehicleId)
      .gte('timestamp', startTime.toISOString())
      .lte('timestamp', endTime.toISOString())
      .order('timestamp', { ascending: true })
      .limit(10000);  // Prevent accidental large queries
    
    if (error) throw error;
    return data || [];
  }

  // Subscribe to realtime updates
  subscribeToVehicleTelemetry(
    vehicleId: string,
    callback: (telemetry: TelemetryLog) => void
  ): RealtimeChannel {
    return this.supabase.client
      .channel(`telemetry:${vehicleId}`)
      .on(
        'postgres_changes',
        {
          event: 'INSERT',
          schema: 'public',
          table: 'telemetry_logs',
          filter: `vehicle_id=eq.${vehicleId}`
        },
        (payload) => callback(payload.new as TelemetryLog)
      )
      .subscribe();
  }
}
```

### 6.3 Fleet and Vehicle Operations

```typescript
// FleetService
@Injectable({ providedIn: 'root' })
export class FleetService {
  constructor(private supabase: SupabaseService) {}

  async getFleets(): Promise<Fleet[]> {
    const { data, error } = await this.supabase.client
      .from('fleets')
      .select('*')
      .order('name');
    
    if (error) throw error;
    return data || [];
  }

  async getFleetVehicles(fleetId: string): Promise<Vehicle[]> {
    const { data, error } = await this.supabase.client
      .from('vehicles')
      .select('*')
      .eq('fleet_id', fleetId)
      .order('make');
    
    if (error) throw error;
    return data || [];
  }

  async addVehicle(vehicle: Partial<Vehicle>): Promise<Vehicle> {
    const { data, error } = await this.supabase.client
      .from('vehicles')
      .insert(vehicle)
      .select()
      .single();
    
    if (error) throw error;
    return data;
  }
}
```

### 6.4 Alert Operations

```typescript
// AlertService
@Injectable({ providedIn: 'root' })
export class AlertService {
  constructor(private supabase: SupabaseService) {}

  async getActiveAlerts(fleetId?: string): Promise<Alert[]> {
    let query = this.supabase.client
      .from('alerts')
      .select('*, vehicles(vin, make, model)')
      .eq('acknowledged', false)
      .order('created_at', { ascending: false });

    if (fleetId) {
      query = query.eq('fleet_id', fleetId);
    }

    const { data, error } = await query;
    if (error) throw error;
    return data || [];
  }

  async acknowledgeAlert(alertId: string): Promise<void> {
    const { error } = await this.supabase.client
      .from('alerts')
      .update({
        acknowledged: true,
        acknowledged_by: this.supabase.client.auth.user()?.id,
        acknowledged_at: new Date().toISOString()
      })
      .eq('id', alertId);
    
    if (error) throw error;
  }
}
```

### 6.5 RPC Functions

```typescript
// Dashboard statistics
async getFleetStats(fleetId: string): Promise<FleetStats> {
  const { data, error } = await this.supabase.client
    .rpc('get_fleet_statistics', { p_fleet_id: fleetId });
  
  if (error) throw error;
  return data;
}

// Bulk vehicle assignment
async assignVehiclesToFleet(vehicleIds: string[], fleetId: string): Promise<void> {
  const { error } = await this.supabase.client
    .rpc('assign_vehicles_to_fleet', {
      p_vehicle_ids: vehicleIds,
      p_fleet_id: fleetId
    });
  
  if (error) throw error;
}
```

---

## 7. Database Functions and Triggers

### 7.1 Alert Generation Trigger

```sql
CREATE OR REPLACE FUNCTION generate_alert_if_needed()
RETURNS TRIGGER AS $$
DECLARE
  v_fleet_id UUID;
  v_rule RECORD;
  v_metric_value FLOAT;
  v_condition_met BOOLEAN;
BEGIN
  -- Get fleet_id for the vehicle
  SELECT fleet_id INTO v_fleet_id FROM vehicles WHERE id = NEW.vehicle_id;
  
  -- Check each enabled rule for this fleet
  FOR v_rule IN 
    SELECT * FROM telemetry_rules 
    WHERE fleet_id = v_fleet_id AND enabled = TRUE
  LOOP
    -- Get the metric value based on rule
    v_metric_value := CASE v_rule.metric
      WHEN 'temp' THEN NEW.temp
      WHEN 'voltage' THEN NEW.voltage
      WHEN 'rpm' THEN NEW.rpm::FLOAT
      ELSE NULL
    END;
    
    -- Check if condition is met
    v_condition_met := CASE v_rule.operator
      WHEN 'gt' THEN v_metric_value > v_rule.threshold
      WHEN 'lt' THEN v_metric_value < v_rule.threshold
      WHEN 'gte' THEN v_metric_value >= v_rule.threshold
      WHEN 'lte' THEN v_metric_value <= v_rule.threshold
      WHEN 'eq' THEN v_metric_value = v_rule.threshold
      ELSE FALSE
    END;
    
    -- If condition met and no recent alert (cooldown), create alert
    IF v_condition_met AND (
      SELECT COUNT(*) FROM alerts 
      WHERE vehicle_id = NEW.vehicle_id 
        AND code = v_rule.name
        AND created_at > NOW() - (v_rule.cooldown_seconds || ' seconds')::INTERVAL
    ) = 0 THEN
      
      INSERT INTO alerts (
        vehicle_id, fleet_id, severity, code, message,
        dtc_codes, lat, lng
      ) VALUES (
        NEW.vehicle_id,
        v_fleet_id,
        v_rule.severity,
        v_rule.code,
        v_rule.name || ' threshold exceeded: ' || v_metric_value || ' (threshold: ' || v_rule.threshold || ')',
        NEW.dtc_codes,
        NEW.lat,
        NEW.lng
      );
    END IF;
  END LOOP;
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trigger_generate_alert
  AFTER INSERT ON telemetry_logs
  FOR EACH ROW
  EXECUTE FUNCTION generate_alert_if_needed();
```

### 7.2 Fleet Statistics Function

```sql
CREATE OR REPLACE FUNCTION get_fleet_statistics(p_fleet_id UUID)
RETURNS TABLE (
  total_vehicles BIGINT,
  active_vehicles BIGINT,
  active_alerts BIGINT,
  critical_alerts BIGINT,
  last_telemetry TIMESTAMPTZ
) AS $$
BEGIN
  RETURN QUERY
  SELECT 
    COUNT(DISTINCT v.id)::BIGINT as total_vehicles,
    COUNT(DISTINCT CASE WHEN v.status = 'active' THEN v.id END)::BIGINT as active_vehicles,
    COUNT(DISTINCT CASE WHEN a.acknowledged = FALSE THEN a.id END)::BIGINT as active_alerts,
    COUNT(DISTINCT CASE WHEN a.acknowledged = FALSE AND a.severity = 'critical' THEN a.id END)::BIGINT as critical_alerts,
    MAX(t.timestamp) as last_telemetry
  FROM vehicles v
  LEFT JOIN alerts a ON v.id = a.vehicle_id
  LEFT JOIN telemetry_logs t ON v.id = t.vehicle_id
  WHERE v.fleet_id = p_fleet_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;
```

### 7.3 Vehicle Assignment Function

```sql
CREATE OR REPLACE FUNCTION assign_vehicles_to_fleet(
  p_vehicle_ids UUID[],
  p_fleet_id UUID
)
RETURNS VOID AS $$
BEGIN
  UPDATE vehicles SET fleet_id = p_fleet_id, updated_at = NOW()
  WHERE id = ANY(p_vehicle_ids);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 7.4 Updated At Trigger

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_vehicles_updated_at
  BEFORE UPDATE ON vehicles
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_fleets_updated_at
  BEFORE UPDATE ON fleets
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### 7.5 Data Retention Trigger (Partitioning Helper)

```sql
CREATE OR REPLACE FUNCTION cleanup_old_telemetry(p_retention_days INTEGER DEFAULT 90)
RETURNS INTEGER AS $$
DECLARE
  v_deleted_count INTEGER;
BEGIN
  WITH deleted AS (
    DELETE FROM telemetry_logs 
    WHERE timestamp < NOW() - (p_retention_days || ' days')::INTERVAL
    RETURNING id
  )
  SELECT COUNT(*) INTO v_deleted_count FROM deleted;
  
  RETURN v_deleted_count;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## 8. Indexing Strategy

### 8.1 Primary Indexes

```sql
-- Already indexed via PRIMARY KEY on id columns

-- Cluster telemetry_logs by vehicle_id and timestamp for optimal read performance
CLUSTER telemetry_logs USING idx_telemetry_logs_vehicle_timestamp;
```

### 8.2 Performance Indexes

```sql
-- telemetry_logs: Most common query patterns
CREATE INDEX idx_telemetry_logs_vehicle_timestamp 
  ON telemetry_logs(vehicle_id, timestamp DESC);

CREATE INDEX idx_telemetry_logs_timestamp 
  ON telemetry_logs(timestamp DESC);

CREATE INDEX idx_telemetry_logs_dtc_codes 
  ON telemetry_logs USING GIN(dtc_codes);

-- alerts: Alert lookup patterns
CREATE INDEX idx_alerts_fleet_acknowledged 
  ON alerts(fleet_id, acknowledged) WHERE acknowledged = FALSE;

CREATE INDEX idx_alerts_vehicle_created 
  ON alerts(vehicle_id, created_at DESC);

-- vehicles: Fleet lookup
CREATE INDEX idx_vehicles_fleet_id 
  ON vehicles(fleet_id);

CREATE INDEX idx_vehicles_vin 
  ON vehicles(vin);

-- fleet_members: User access lookup (critical for RLS performance)
CREATE INDEX idx_fleet_members_user_id 
  ON fleet_members(user_id);

-- telemetry_rules: Rule matching
CREATE INDEX idx_telemetry_rules_fleet_enabled 
  ON telemetry_rules(fleet_id, enabled) WHERE enabled = TRUE;
```

### 8.3 Partial Indexes

```sql
-- Only index unacknowledged alerts for faster dashboard queries
CREATE INDEX idx_alerts_unacknowledged 
  ON alerts(fleet_id, created_at DESC)
  WHERE acknowledged = FALSE;

-- Only index active vehicles
CREATE INDEX idx_vehicles_active 
  ON vehicles(fleet_id, vin)
  WHERE status = 'active';
```

### 8.4 Index Maintenance

```sql
-- Schedule: Run weekly during low-traffic period
REINDEX INDEX CONCURRENTLY idx_telemetry_logs_vehicle_timestamp;
ANALYZE telemetry_logs;
```

---

## 9. Backup and Recovery

### 9.1 Supabase Managed Backups

| Backup Type | Frequency | Retention | Recovery Point Objective |
|-------------|-----------|-----------|--------------------------|
| Automated daily | Daily | 7 days | 24 hours |
| Point-in-time | Continuous | 30 days | 1 hour |
| Manual | On-demand | Until deleted | Depends on manual action |

### 9.2 Recovery Procedures

```bash
# Point-in-time recovery via Supabase Dashboard:
# 1. Navigate to Database > Backups
# 2. Select Point-in-time recovery
# 3. Choose recovery timestamp
# 4. Create new project or restore to existing

# For programmatic backup (pg_dump):
pg_dump -h db.your-project.supabase.co -U postgres -d postgres \
  --format=c \
  --compress=9 \
  --file=vitalsdrive_backup_$(date +%Y%m%d).dump
```

### 9.3 Recovery Time Objective (RTO)

| Scenario | RTO | Procedure |
|----------|-----|-----------|
| Accidental delete | < 1 hour | Point-in-time restore |
| Table drop | < 1 hour | Point-in-time restore specific table |
| Project failure | < 2 hours | Restore from backup to new project |
| Data corruption | < 2 hours | Point-in-time restore |

### 9.4 Backup Verification

```sql
-- Verify backup integrity (run on restored copy)
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)),
  n_live_tup,
  n_dead_tup,
  last_vacuum,
  last_autovacuum
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## 10. Migration Strategy

### 10.1 Migration Tool

Use **Supabase CLI** with **db Push** for schema migrations:

```bash
# Install Supabase CLI
npm install -g supabase

# Login
supabase login

# Initialize (if not already done)
supabase init

# Link to project
supabase link --project-ref your-project-ref

# Create migration
supabase migration new add_vehicle_table

# Push migrations
supabase db push
```

### 10.2 Migration File Structure

```
supabase/
├── migrations/
│   ├── 20260329_001_initial_schema.sql
│   ├── 20260329_002_add_telemetry_rules.sql
│   └── 20260329_003_add_realtime.sql
└── seed/
    └── 001_seed_data.sql
```

### 10.3 Initial Schema Migration (20260329_001)

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create tables (from Section 2)
CREATE TABLE telemetry_logs (...);
CREATE TABLE vehicles (...);
CREATE TABLE fleets (...);
CREATE TABLE users (...);
CREATE TABLE fleet_members (...);
CREATE TABLE alerts (...);
CREATE TABLE telemetry_rules ...;

-- Add indexes
CREATE INDEX idx_telemetry_logs_vehicle_timestamp ON telemetry_logs(vehicle_id, timestamp DESC);
...

-- Enable RLS
ALTER TABLE telemetry_logs ENABLE ROW LEVEL SECURITY;
...

-- Create policies (from Section 5)
CREATE POLICY "Users can read own fleet telemetry" ON telemetry_logs ...;
...

-- Create functions and triggers
CREATE OR REPLACE FUNCTION generate_alert_if_needed() ...;
CREATE TRIGGER trigger_generate_alert ...;
```

### 10.4 Zero-Downtime Migration Strategy

```sql
-- Phase 1: Add new column (non-breaking)
ALTER TABLE telemetry_logs ADD COLUMN speed FLOAT;

-- Phase 2: Deploy application code that writes to both old and new
-- (Application handles nulls gracefully)

-- Phase 3: Backfill data (if needed, during low-traffic window)
UPDATE telemetry_logs SET speed = 0 WHERE speed IS NULL;

-- Phase 4: Add NOT NULL constraint after backfill
ALTER TABLE telemetry_logs ALTER COLUMN speed SET NOT NULL;
ALTER TABLE telemetry_logs ALTER COLUMN speed SET DEFAULT 0;
```

### 10.5 Rollback Strategy

```bash
# Supabase CLI handles rollback via migration version
supabase migration revert 20260329_001_add_telemetry_rules

# For critical migrations, maintain manual rollback script:
-- migration_rollback_001.sql
ALTER TABLE telemetry_logs DROP COLUMN IF EXISTS new_column;
```

---

## 11. Monitoring

### 11.1 Supabase Dashboard Metrics

Access at: https://app.supabase.com/project/{project}/monitoring

| Metric | Warning Threshold | Critical Threshold | Action |
|--------|-------------------|-------------------|--------|
| Database size | 400MB (80%) | 480MB (96%) | Review data retention |
| Connection count | 40 | 50 | Check for connection leaks |
| Queries/sec | 50 | 80 | Optimize slow queries |
| Replication lag | 2s | 5s | Check network latency |

### 11.2 Custom Monitoring Queries

```sql
-- Connection usage
SELECT 
  count(*) as current_connections,
  (SELECT setting FROM pg_settings WHERE name = 'max_connections') as max_connections
FROM pg_stat_activity WHERE state = 'active';

-- Table sizes
SELECT 
  relname as table_name,
  pg_size_pretty(pg_total_relation_size(relid)) as total_size,
  pg_size_pretty(pg_relation_size(relid)) as table_size,
  pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) as indexes_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Slow queries (queries taking > 1 second)
SELECT 
  pid,
  now() - query_start as duration,
  usename,
  query
FROM pg_stat_activity
WHERE state = 'active' 
  AND query_start < now() - interval '1 second'
ORDER BY query_start;

-- Index hit ratio
SELECT 
  relname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch,
  100.0 * idx_scan / NULLIF(idx_scan + seq_scan, 0) as index_hit_ratio
FROM pg_stat_user_tables
WHERE (idx_scan + seq_scan) > 0
ORDER BY idx_scan;
```

### 11.3 Alerting Configuration

```javascript
// Supabase webhook for monitoring alerts
// Configure in Dashboard > Database > Webhooks

const alertWebhook = {
  event: 'INSERT',
  table: 'pg_stat_activity',
  // Custom filter logic in edge function
};

// Recommended: Use Grafana + Prometheus for advanced monitoring
// Supabase exports metrics at /metrics endpoint
```

### 11.4 Log Analysis

```sql
-- Enable query logging (run once)
ALTER DATABASE postgres SET log_statement = 'all';
ALTER DATABASE postgres SET log_min_duration_statement = 1000;  -- Log > 1s queries

-- View recent logs
SELECT 
  log_time,
  left(query, 100) as query_preview,
  (total_time / 1000)::text || 's' as duration
FROM pg_stat_activity
JOIN pg_ls_dir('/var/log/supabase/postgresql/') AS files(filename)
ORDER BY log_time DESC;
```

---

## 12. Scaling Considerations

### 12.1 Supabase Tier Upgrades

| Tier | Price | DB Size | Bandwidth | Connections |
|------|-------|---------|-----------|-------------|
| Free | $0 | 500MB | 2GB | 60 |
| Pro | $25/month | 8GB | 50GB | 200 |
| Team | $599/month | 100GB | 1TB | 500 |

### 12.2 Scaling Triggers

| Metric | Threshold to Upgrade |
|--------|---------------------|
| Database size | > 400MB sustained |
| Bandwidth | > 1.5GB/month |
| Connection errors | > 5/month |
| Query latency | > 500ms p99 |

### 12.3 Read Replicas

```sql
-- For Pro tier and above, add read replica
-- Configure in Dashboard > Database > Replication
-- Read replica URL provided separately

-- Application read/write splitting
const supabaseRead = createClient(
  'https://your-project-read-replica.supabase.co',
  environment.supabaseAnonKey
);

const supabaseWrite = createClient(
  environment.supabaseUrl,
  environment.supabaseAnonKey
);
```

### 12.4 Connection Pooling

```sql
-- Supabase uses PgBouncer internally
-- Pool mode: Transaction mode (recommended for HTTP)

-- For direct access with pooling:
-- Host: db.your-project.supabase.co
-- Port: 5432
-- Database: postgres
-- User: postgres
-- Password: [your-password]
-- Pool mode: Transaction
-- Max connections: configurable per tier
```

### 12.5 Future Partitioning Strategy

```sql
-- When telemetry_logs exceeds 100M rows, implement partitioning
CREATE TABLE telemetry_logs (
  id          BIGSERIAL,
  vehicle_id  UUID NOT NULL,
  lat         DECIMAL(10, 7),
  lng         DECIMAL(10, 7),
  temp        FLOAT,
  voltage     FLOAT,
  rpm         INTEGER,
  dtc_codes   TEXT[],
  timestamp   TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

-- Monthly partitions
CREATE TABLE telemetry_logs_2026_01 PARTITION OF telemetry_logs
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Index on each partition
CREATE INDEX ON telemetry_logs_2026_01 (vehicle_id, timestamp DESC);
```

---

## 13. Cost Estimation

### 13.1 Free Tier Capacity (MVP)

| Resource | Usage | Limit | Buffer |
|----------|-------|-------|--------|
| Database | ~50MB | 500MB | 450MB |
| Bandwidth | ~0.5GB/month | 2GB | 1.5GB |
| Storage | ~30MB | 1GB | 970MB |
| Connections | ~10 avg | 60 | 50 |

**Estimated Monthly Cost: $0.00**

### 13.2 Growth Projections

| Month | DB Size | Bandwidth | Projected Cost |
|-------|---------|-----------|----------------|
| 1 | 10MB | 0.1GB | $0 |
| 6 | 60MB | 0.6GB | $0 |
| 12 | 120MB | 1.2GB | $0 |
| 18 | 250MB | 2.0GB | $0* |
| 24 | 400MB | 3.0GB | $25 (Pro tier) |

*Bandwidth may exceed 2GB; monitor closely.

### 13.3 Cost Optimization Strategies

```sql
-- 1. Implement data retention policy
-- Run monthly cleanup
SELECT cleanup_old_telemetry(90);  -- Keep 90 days

-- 2. Use compression
-- PostgreSQL TOAST compression is automatic for large values

-- 3. Archive old data to cold storage
-- Export to CSV, store in Supabase Storage (cheap)
COPY (SELECT * FROM telemetry_logs WHERE timestamp < '2025-01-01') 
TO '/tmp/archive_2024.csv' CSV HEADER;
```

### 13.4 Unit Economics

| Cost Center | Calculation | Monthly Cost |
|-------------|-------------|--------------|
| Supabase hosting | Free tier | $0.00 |
| CDN (if needed) | 100GB @ $0.02/GB | $2.00 |
| Backup storage | Included | $0.00 |
| **Total** | | **$0.50** (baseline) |

---

## 14. Testing Strategy

### 14.1 Unit Tests

```typescript
// __tests__/services/telemetry.service.spec.ts
describe('TelemetryService', () => {
  let service: TelemetryService;
  let mockSupabaseService: Partial<SupabaseService>;

  beforeEach(() => {
    mockSupabaseService = {
      client: {
        from: jest.fn().mockReturnValue({
          insert: jest.fn().mockResolvedValue({ error: null }),
          select: jest.fn().mockReturnValue({
            eq: jest.fn().mockReturnValue({
              order: jest.fn().mockReturnValue({
                limit: jest.fn().mockResolvedValue({ data: null, error: null })
              })
            })
          })
        }) as any,
        channel: jest.fn().mockReturnValue({
          on: jest.fn().mockReturnThis(),
          subscribe: jest.fn().mockResolvedValue('channel')
        })
      }
    };
    service = new TelemetryService(mockSupabaseService as SupabaseService);
  });

  describe('ingestTelemetry', () => {
    it('should insert telemetry without error', async () => {
      const payload: TelemetryPayload = {
        vehicleId: 'uuid-123',
        latitude: 37.7749,
        longitude: -122.4194,
        coolantTemp: 90,
        batteryVoltage: 12.6,
        rpm: 2000,
        dtcCodes: []
      };

      await expect(service.ingestTelemetry(payload)).resolves.not.toThrow();
    });

    it('should throw on insert error', async () => {
      (mockSupabaseService.client.from('').insert as jest.Mock)
        .mockResolvedValueOnce({ error: { message: 'Insert failed' } });

      await expect(service.ingestTelemetry({} as any)).rejects.toThrow();
    });
  });
});
```

### 14.2 Integration Tests

```sql
-- __tests__/database/telemetry_rules.test.sql

-- Test alert generation
BEGIN;

-- Setup: Create test fleet and vehicle
INSERT INTO fleets (id, name, owner_id) VALUES ('test-fleet-uuid', 'Test Fleet', 'test-user-uuid');
INSERT INTO vehicles (id, fleet_id, vin, make, model) VALUES ('test-vehicle-uuid', 'test-fleet-uuid', '1HGBH41JXMN109186', 'Honda', 'Civic');

-- Setup: Create rule
INSERT INTO telemetry_rules (fleet_id, name, metric, operator, threshold, severity)
VALUES ('test-fleet-uuid', 'OVERTEMP', 'temp', 'gt', 100, 'critical');

-- Test: Insert telemetry exceeding threshold
INSERT INTO telemetry_logs (vehicle_id, temp) VALUES ('test-vehicle-uuid', 105);

-- Verify: Alert created
ASSERT (SELECT COUNT(*) FROM alerts WHERE code = 'OVERTEMP' AND vehicle_id = 'test-vehicle-uuid') = 1,
  'Alert should be generated for temp > 100';

-- Test: Cooldown prevents duplicate alerts
INSERT INTO telemetry_logs (vehicle_id, temp) VALUES ('test-vehicle-uuid', 106);

ASSERT (SELECT COUNT(*) FROM alerts WHERE code = 'OVERTEMP' AND vehicle_id = 'test-vehicle-uuid') = 1,
  'Second alert should be blocked by cooldown';

ROLLBACK;
```

### 14.3 Load Tests

```typescript
// __tests__/load/telemetry-load.spec.ts
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 10 },   // Ramp up
    { duration: '5m', target: 50 },   // Steady state
    { duration: '1m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% under 500ms
    http_req_failed: ['rate<0.01'],     // <1% failure rate
  },
};

export default () => {
  const payload = JSON.stringify({
    vehicle_id: 'test-vehicle-uuid',
    temp: 85 + Math.random() * 20,
    voltage: 12 + Math.random(),
    rpm: 1500 + Math.random() * 1000,
    dtc_codes: []
  });

  const res = http.post(
    `${__ENV.SUPABASE_URL}/rest/v1/telemetry_logs`,
    payload,
    {
      headers: {
        'Content-Type': 'application/json',
        'apikey': __Env.SUPABASE_ANON_KEY,
        'Authorization': `Bearer ${__ENV.SUPABASE_ANON_KEY}`,
        'Prefer': 'return=minimal'
      }
    }
  );

  check(res, { 'status was 201': (r) => r.status === 201 });
  sleep(1);
};
```

### 14.4 RLS Policy Tests

```sql
-- __tests__/database/rls_policies.test.sql

-- Test: User can only see their fleet's data
BEGIN;

-- Setup
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SET LOCAL app.current_user = 'user-uuid-1';

-- Verify: User A cannot see User B's fleet
ASSERT NOT EXISTS (
  SELECT 1 FROM fleets WHERE id = 'user-b-fleet-uuid'
), 'RLS should block cross-fleet access';

-- Verify: User A can see their own fleet
ASSERT EXISTS (
  SELECT 1 FROM fleets WHERE id = 'user-a-fleet-uuid'
), 'RLS should allow own fleet access';

-- Verify: User A cannot see User B's telemetry
ASSERT NOT EXISTS (
  SELECT 1 FROM telemetry_logs tl
  JOIN vehicles v ON tl.vehicle_id = v.id
  JOIN fleets f ON v.fleet_id = f.id
  WHERE f.id = 'user-b-fleet-uuid'
), 'RLS should block cross-fleet telemetry';

ROLLBACK;
```

### 14.5 Test Environments

| Environment | Purpose | Data Freshness |
|-------------|---------|----------------|
| Local (Supabase CLI) | Unit tests, local dev | Seeded with synthetic data |
| Staging (Supabase staging project) | Integration tests | Anonymized production copy |
| Production | Load tests (limited) | Real data |

---

## Appendix A: Complete SQL Reference

### A.1 Full Schema Creation Script

```sql
-- VitalsDrive Database Schema
-- Version: 1.0.0

BEGIN;

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Tables
CREATE TABLE telemetry_logs (...);  -- See Section 2.1
CREATE TABLE vehicles (...);         -- See Section 2.2
CREATE TABLE fleets (...);           -- See Section 2.3
CREATE TABLE users (...);            -- See Section 2.4
CREATE TABLE fleet_members (...);    -- See Section 2.5
CREATE TABLE alerts (...);           -- See Section 2.6
CREATE TABLE telemetry_rules (...);  -- See Section 2.7

-- Indexes
-- See Section 8

-- Functions
CREATE OR REPLACE FUNCTION generate_alert_if_needed() ...;  -- See Section 7.1
CREATE OR REPLACE FUNCTION get_fleet_statistics() ...;      -- See Section 7.2
CREATE OR REPLACE FUNCTION assign_vehicles_to_fleet() ...;  -- See Section 7.3
CREATE OR REPLACE FUNCTION update_updated_at_column() ...;  -- See Section 7.4
CREATE OR REPLACE FUNCTION cleanup_old_telemetry() ...;      -- See Section 7.5
CREATE OR REPLACE FUNCTION get_user_fleet_ids() ...;        -- See Section 5.2

-- Triggers
CREATE TRIGGER trigger_generate_alert ...;  -- See Section 7.1
CREATE TRIGGER update_vehicles_updated_at ...; -- See Section 7.4
CREATE TRIGGER update_fleets_updated_at ...;   -- See Section 7.4

-- RLS
ALTER TABLE ... ENABLE ROW LEVEL SECURITY;  -- See Section 5.1
CREATE POLICY ...;                            -- See Section 5.3

-- Realtime
ALTER PUBLICATION supabase_realtime ADD TABLE telemetry_logs;
ALTER PUBLICATION supabase_realtime ADD TABLE alerts;

COMMIT;
```

---

## Appendix B: Environment Variables

```bash
# .env.local (Angular)
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key

# .env (Supabase CLI)
SUPABASE_DB_PASSWORD=your-db-password
POSTGRES_PASSWORD=your-db-password
```

---

## Appendix C: Related Documents

| Document | Path |
|----------|------|
| Main PRD | docs/PRD-Master.md |
| API Specification | docs/PRD-Layer3-API.md |
| Frontend Architecture | docs/PRD-Layer4-Frontend.md |
| IoT Integration Guide | docs/PRD-Layer1-IoT.md |

---

**Document Owner:** Backend Team  
**Reviewers:** [TBD]  
**Next Review Date:** 2026-04-29
