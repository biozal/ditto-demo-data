# Pluto Demo Queries

This document provides example DQL queries to demonstrate Ditto's capabilities with the Pluto dataset.

## Data Overview

- **5 Bases**: Nellis AFB, Edwards AFB, Luke AFB, Langley AFB, Dyess AFB
- **65 Long-lived documents**: 50 crew members + 15 aircraft
- **207 Short-lived documents**: Missions, logs, briefs, updates, assignments
- **Total**: 272 documents across 8 collections

---

## 1. Location-Based Subscriptions

### Subscribe to Active Crew at Nellis AFB
```sql
SELECT * FROM crew_members
WHERE homeBase = 'Nellis AFB'
AND status = 'Active'
```
**Use Case**: Pilot device at Nellis AFB only syncs active crew from their base.

### Subscribe to Mission-Ready Aircraft at Luke AFB
```sql
SELECT * FROM aircraft
WHERE homeBase = 'Luke AFB'
AND status = 'Mission Ready'
```
**Use Case**: Mission planning system shows available aircraft.

### Subscribe to Active Missions from Dyess AFB
```sql
SELECT * FROM missions
WHERE departureBase = 'Dyess AFB'
AND status IN ('Planned', 'Briefed', 'In Progress')
```
**Use Case**: Operations center only syncs active missions, not completed ones.

---

## 2. Role-Based Subscription Patterns

### Flight Crew: Active Missions Only
```sql
-- Missions
SELECT * FROM missions
WHERE departureBase = 'Nellis AFB'
AND status IN ('Briefed', 'In Progress')

-- My crew assignments
SELECT * FROM crew_assignments
WHERE missionDepartureBase = 'Nellis AFB'
AND missionStatus IN ('Briefed', 'In Progress')

-- Recent status updates
SELECT * FROM status_updates
WHERE missionDepartureBase = 'Nellis AFB'
AND createdAt > '2026-02-02T00:00:00Z'
```

### Maintenance: Aircraft and Maintenance Data
```sql
-- Aircraft at my base
SELECT * FROM aircraft
WHERE homeBase = 'Edwards AFB'

-- Open maintenance issues
SELECT * FROM maintenance_status
WHERE aircraftHomeBase = 'Edwards AFB'
AND status IN ('Open', 'In Progress', 'Awaiting Parts')
```

### Operations Center: Everything
```sql
-- All missions (including completed for tracking)
SELECT * FROM missions
WHERE departureBase = 'Langley AFB'

-- All status updates (last 24 hours)
SELECT * FROM status_updates
WHERE missionDepartureBase = 'Langley AFB'
AND createdAt > '2026-02-02T21:55:00Z'

-- All crew assignments
SELECT * FROM crew_assignments
WHERE missionDepartureBase = 'Langley AFB'
```

---

## 3. Time-Based Filtering

### Recent Missions (Last 7 Days)
```sql
SELECT * FROM missions
WHERE departureBase = 'Luke AFB'
AND createdAt > '2026-01-27T00:00:00Z'
```

### Recent Flight Logs (Last 30 Days)
```sql
SELECT * FROM flight_logs
WHERE missionDepartureBase = 'Nellis AFB'
AND flightDate > '2026-01-03'
```

### Active Status Updates Only (Last 24 Hours)
```sql
SELECT * FROM status_updates
WHERE missionDepartureBase = 'Edwards AFB'
AND createdAt > '2026-02-02T00:00:00Z'
ORDER BY createdAt DESC
```

---

## 4. Status-Based Filtering

### Available Pilots
```sql
SELECT * FROM crew_members
WHERE homeBase = 'Luke AFB'
AND crewPosition = 'Pilot'
AND status = 'Active'
```

### Grounded Aircraft
```sql
SELECT * FROM maintenance_status
WHERE aircraftHomeBase = 'Dyess AFB'
AND groundsAircraft = true
AND status != 'Resolved'
```

### Critical Open Maintenance
```sql
SELECT * FROM maintenance_status
WHERE aircraftHomeBase = 'Nellis AFB'
AND discrepancyType = 'Critical'
AND status IN ('Open', 'In Progress')
```

---

## 5. Eviction Queries

### Evict Completed Missions (30+ Days Old)
```javascript
// First, find old missions to evict
const oldMissions = await ditto.store.execute(`
  SELECT _id FROM missions
  WHERE status = 'Completed'
  AND completedAt < '2026-01-03T00:00:00Z'
`);

// Evict missions
await ditto.store.execute(`
  EVICT FROM missions
  WHERE status = 'Completed'
  AND completedAt < '2026-01-03T00:00:00Z'
`);
```

### Evict Related Data by missionId

After evicting missions, evict all related data by missionId:

```javascript
// Evict crew assignments by missionId
for (const mission of oldMissions) {
  await ditto.store.execute(`
    EVICT FROM crew_assignments WHERE missionId = '${mission._id}'
  `);
}

// Evict flight logs by missionId (90+ days)
for (const mission of oldMissions) {
  await ditto.store.execute(`
    EVICT FROM flight_logs WHERE missionId = '${mission._id}'
  `);
}

// Evict mission briefs by missionId
for (const mission of oldMissions) {
  await ditto.store.execute(`
    EVICT FROM mission_briefs WHERE missionId = '${mission._id}'
  `);
}

// Evict status updates by missionId
for (const mission of oldMissions) {
  await ditto.store.execute(`
    EVICT FROM status_updates WHERE missionId = '${mission._id}'
  `);
}
```

### Evict Resolved Maintenance (60+ Days Old)
```sql
EVICT FROM maintenance_status
WHERE status = 'Resolved'
AND resolvedAt < '2025-12-03T00:00:00Z'
```

**Note**: Run eviction queries sparingly (daily, off-peak) as they trigger full resync with peers.

---

## 6. Complex Filtering Examples

### Missions Needing Crew Assignments
```sql
SELECT * FROM missions
WHERE departureBase = 'Nellis AFB'
AND status = 'Planned'
```
Then in application code, filter out missions that already have assignments.

### Aircraft Needing Inspection Soon (Next 30 Days)
```sql
SELECT * FROM aircraft
WHERE homeBase = 'Luke AFB'
AND nextInspectionDue < '2026-03-03T00:00:00Z'
AND status IN ('Mission Ready', 'Maintenance')
```

### High-Priority Urgent Updates
```sql
SELECT * FROM status_updates
WHERE missionDepartureBase = 'Edwards AFB'
AND updatePriority IN ('Urgent', 'Emergency')
ORDER BY createdAt DESC
LIMIT 10
```

---

## 7. Demonstration Scenarios

### Scenario 1: Offline Flight Logging
1. Pilot takes off from Nellis AFB (aircraft goes offline)
2. During flight, crew creates status updates locally
3. Updates queue in local Ditto store (offline)
4. Aircraft lands, reconnects to wifi
5. **Status updates automatically sync** to operations center

**Query at Ops Center**:
```sql
SELECT * FROM status_updates
WHERE missionDepartureBase = 'Nellis AFB'
AND missionNumber = 'MSN-2026-001'
ORDER BY createdAt ASC
```

### Scenario 2: Mission Planning with Stale Data
1. Operations at Luke AFB planning mission
2. Subscribe to active crew and mission-ready aircraft
3. **Only current, relevant data syncs** (not all historical data)
4. Create mission, assign crew, generate briefs
5. All devices at Luke AFB see updates in real-time

**Subscription**:
```sql
-- Crew available for assignment
SELECT * FROM crew_members
WHERE homeBase = 'Luke AFB'
AND status = 'Active'

-- Aircraft available for mission
SELECT * FROM aircraft
WHERE homeBase = 'Luke AFB'
AND status = 'Mission Ready'
```

### Scenario 3: Maintenance Updates
1. Maintenance crew discovers hydraulic leak
2. Create maintenance_status document offline
3. Mark aircraft status as 'Down for Repairs'
4. **Update syncs to all devices** subscribed to aircraft at that base
5. Mission planning immediately sees aircraft unavailable

**Query**:
```sql
-- See all maintenance for my aircraft
SELECT * FROM maintenance_status
WHERE aircraftTailNumber = '90-0001'
AND status != 'Resolved'
ORDER BY priority ASC
```

### Scenario 4: Cross-Base Coordination
1. Mission departs from Nellis AFB
2. Lands at Edwards AFB for refueling
3. Edwards ops center **doesn't see mission** (departureBase != Edwards)
4. Status updates still reference Nellis (missionDepartureBase)
5. Edwards only sees their own missions

**This demonstrates location-based isolation** - each base only syncs relevant data.

---

## 8. Teaching Points

### Why Denormalization?
```sql
-- BAD: Can't filter by mission's base
SELECT * FROM flight_logs WHERE missionId = 'xyz'

-- GOOD: Can subscribe by base
SELECT * FROM flight_logs WHERE missionDepartureBase = 'Nellis AFB'
```

### Status vs Time-Based Filtering
```sql
-- Long-lived: Status-based
SELECT * FROM crew_members WHERE status = 'Active'

-- Short-lived: Time + status
SELECT * FROM missions WHERE status = 'In Progress'
OR (status = 'Completed' AND completedAt > '2026-01-27T00:00:00Z')
```

### No JOINs - Use DAOs
```javascript
// 1. Fetch mission (already synced via subscription)
const mission = await ditto.store.execute(
  "SELECT * FROM missions WHERE _id = :id",
  { id: missionId }
);

// 2. Fetch crew assignments (already synced via subscription)
const crew = await ditto.store.execute(
  "SELECT * FROM crew_assignments WHERE missionId = :id",
  { id: missionId }
);

// 3. Combine in memory
const missionWithCrew = { ...mission, crew };
```

---

## Data Generation Details

**Generated with realistic distributions:**
- Missions: 25% Planned, 20% Briefed, 15% In Progress, 40% Completed
- Crew Status: 80% Active, 15% operational (leave/training), 5% inactive
- Aircraft Status: 60% Mission Ready, 30% Maintenance, 10% other
- Maintenance: 60% open, 40% resolved

**Denormalization applied:**
- `missionDepartureBase` in all mission-related collections
- `aircraftHomeBase` in maintenance_status
- `missionStatus` and `missionCompletedAt` for eviction filtering
- Names, ranks, aircraft types for display without additional fetches

**Realistic data:**
- Dates span last 60 days to present + 7 days future
- Status updates include position data (lat/long/altitude)
- Flight logs include duration, landings, instrument time
- Maintenance includes work orders, priorities, parts required
