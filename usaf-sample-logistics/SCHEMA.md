# Pluto Schema Design

This document describes the data model for Pluto, an aircrew digital coordination system designed for low-connectivity environments.

## Design Principles

### 1. Subscription-Driven Architecture
**All collections MUST be filterable by location (base)** because Ditto subscriptions work at the collection level with WHERE clauses. There are no JOINs in DQL, so data must be denormalized to enable efficient subscriptions.

**Example Subscription Pattern:**
```javascript
// Subscribe only to missions from my base
ditto.store.execute("SELECT * FROM missions WHERE departureBase = 'Nellis AFB'")

// Subscribe only to active crew at my base
ditto.store.execute("SELECT * FROM crew_members WHERE homeBase = 'Nellis AFB' AND status = 'Active'")
```

### 2. Heavy Denormalization
Since Ditto doesn't support JOINs, related data is denormalized into each document. For example:
- `flight_logs` includes both `missionId` AND `missionDepartureBase` (denormalized from mission)
- `status_updates` includes `missionId` AND `missionDepartureBase` (denormalized from mission)
- This allows subscription filtering without needing to fetch the parent mission first

### 3. Data Lifecycle: Long-Lived vs Short-Lived

**CRITICAL DISTINCTION:**

#### Long-Lived Reference Data (Months/Years on Device)
**Examples:** Crew members, aircraft inventory, qualifications

**Characteristics:**
- **Low churn rate** - Data doesn't change frequently
- **Persistent across missions** - Same crew flies multiple missions
- **Status-based filtering** - Use `status = 'Active'` instead of time-based eviction
- **Larger dataset** - May have 100s of crew members, dozens of aircraft

**Subscription Pattern:**
```javascript
// Subscribe to active crew (stays on device indefinitely)
SELECT * FROM crew_members
WHERE homeBase = 'Nellis AFB'
AND status = 'Active'
```

**Eviction Pattern:**
```javascript
// Rarely evict - only when crew member leaves permanently
// Usually just change status to 'Inactive' or 'Transferred'
EVICT FROM crew_members
WHERE status IN ('Retired', 'Transferred')
AND updatedAt < :oneYearAgo
```

#### Short-Lived Transactional Data (Days/Weeks on Device)
**Examples:** Missions, flight logs, status updates, briefs

**Characteristics:**
- **High churn rate** - New data created frequently
- **Mission-specific** - Tied to specific mission lifecycle
- **Time-based eviction** - Evict after mission completion + retention period
- **Smaller working set** - Only need recent/active data

**Subscription Pattern:**
```javascript
// Subscribe to active missions only (evict when completed)
SELECT * FROM missions
WHERE departureBase = 'Nellis AFB'
AND status IN ('Planned', 'Briefed', 'In Progress')
```

**Eviction Pattern:**
```javascript
// Aggressively evict completed missions
EVICT FROM missions
WHERE status = 'Completed'
AND completedAt < :thirtyDaysAgo
```

### 4. Local vs System-Wide Operations
- **DELETE**: Removes data from entire Ditto mesh (creates tombstones)
- **EVICT**: Removes data only from local device storage
- Use EVICT sparingly (triggers full resync with peers)

### 5. Status vs Time-Based Filtering
- **Long-lived data**: Use `status` field for filtering (Active, Inactive, Retired)
- **Short-lived data**: Use `status` + timestamp for eviction (Completed + completedAt)
- **Subscriptions**: Filter by status to control what syncs
- **Eviction**: Time + status to clean up old transactional data

---

## Collection Schemas

### Collection Categories

**Long-Lived Reference Data (Rarely Evicted):**
1. `crew_members` - Aircrew personnel roster
2. `aircraft` - Aircraft inventory

**Short-Lived Transactional Data (Actively Evicted):**
3. `missions` - Mission records
4. `crew_assignments` - Mission crew linkage
5. `flight_logs` - Individual flight records
6. `maintenance_status` - Aircraft maintenance issues
7. `mission_briefs` - Pre-flight briefings
8. `status_updates` - Real-time mission updates

---

## Long-Lived Reference Data

### 1. `crew_members` Collection
**Lifecycle: LONG-LIVED** - Stays on device for months/years, controlled by status.

Aircrew personnel roster.

```json
{
  "_id": "UUID",
  "firstName": "string",
  "middleName": "string",
  "lastName": "string",
  "rank": "string",
  "rankAbbreviation": "string",

  "crewPosition": "string - 'Pilot', 'Co-Pilot', 'Navigator', 'Flight Engineer', 'Loadmaster', 'Boom Operator'",
  "primaryAircraftType": "string - Primary aircraft qualification",

  "qualifications": "object - Map of aircraft types to qualification level",

  "status": "string - 'Active', 'Inactive', 'On Leave', 'Medical Hold', 'Training', 'Transferred', 'Retired'",
  "statusReason": "string - Brief explanation (nullable)",
  "statusEffectiveDate": "ISO 8601 datetime",

  "homeBase": "string - REQUIRED FOR SUBSCRIPTIONS",

  "email": "string - firstname.lastname@fake.us.af.mil",
  "dutyPhone": "string - XXX-555-01XX",

  "totalFlightHours": "number",
  "lastFlightDate": "ISO 8601 datetime (nullable)",

  "createdAt": "ISO 8601 datetime",
  "updatedAt": "ISO 8601 datetime"
}
```

**Qualifications Object Example:**
```json
{
  "qualifications": {
    "F-16C": "Instructor Pilot",
    "F-35A": "Mission Qualified",
    "T-38": "Basic Qualified"
  }
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to active crew at my base (RECOMMENDED - stays on device)
SELECT * FROM crew_members
WHERE homeBase = 'Nellis AFB'
AND status = 'Active'

-- Subscribe to all operational crew (active + on leave + training)
SELECT * FROM crew_members
WHERE homeBase = 'Nellis AFB'
AND status IN ('Active', 'On Leave', 'Training')

-- Subscribe to available pilots specifically
SELECT * FROM crew_members
WHERE homeBase = 'Nellis AFB'
AND status = 'Active'
AND crewPosition = 'Pilot'
```

**Eviction Query Examples:**
```sql
-- RARELY evict - only for permanently departed crew (1+ year old)
EVICT FROM crew_members
WHERE status IN ('Retired', 'Transferred')
AND statusEffectiveDate < '2025-01-01T00:00:00Z'

-- Usually you just change status to 'Inactive' or 'Transferred' instead of evicting
-- This keeps historical crew data available for past mission lookups
```

**Why This Design:**
- **Status-based filtering** controls subscriptions (not time-based)
- Crew members stay on device across many missions
- Status changes (Active → On Leave → Active) don't require re-sync
- Only evict when crew permanently leaves the base
- Denormalized qualifications as map (not array) follows Ditto best practices

---

### 2. `aircraft` Collection
**Lifecycle: LONG-LIVED** - Stays on device for months/years, controlled by status.

Aircraft inventory and operational status.

```json
{
  "_id": "UUID",
  "tailNumber": "string - Unique e.g., '91-0409'",
  "aircraftType": "string - 'F-16C', 'C-130J', 'KC-135R', 'B-52H', 'F-35A'",
  "squadron": "string - e.g., '56th Fighter Wing'",

  "homeBase": "string - REQUIRED FOR SUBSCRIPTIONS",
  "currentLocation": "string - Current base or 'In Flight'",

  "status": "string - 'Mission Ready', 'Maintenance', 'Inspection', 'Down for Repairs', 'Retired', 'Transferred'",
  "statusReason": "string - Brief explanation (nullable)",
  "statusEffectiveDate": "ISO 8601 datetime",

  "flightHours": "number - Total flight hours",
  "flightsSinceInspection": "number",

  "lastInspectionDate": "ISO 8601 datetime",
  "nextInspectionDue": "ISO 8601 datetime",

  "fuelCapacity": "number - Gallons",
  "maxRange": "number - Nautical miles",
  "maxAltitude": "number - Feet",

  "createdAt": "ISO 8601 datetime",
  "updatedAt": "ISO 8601 datetime"
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to operational aircraft at my base (RECOMMENDED)
SELECT * FROM aircraft
WHERE homeBase = 'Nellis AFB'
AND status IN ('Mission Ready', 'Maintenance', 'Inspection')

-- Subscribe to mission-ready aircraft only
SELECT * FROM aircraft
WHERE homeBase = 'Nellis AFB'
AND status = 'Mission Ready'

-- Subscribe to all aircraft (including retired for historical data)
SELECT * FROM aircraft WHERE homeBase = 'Nellis AFB'
```

**Eviction Query Examples:**
```sql
-- RARELY evict - only for permanently retired/transferred aircraft
EVICT FROM aircraft
WHERE status IN ('Retired', 'Transferred')
AND statusEffectiveDate < '2025-01-01T00:00:00Z'

-- Usually keep aircraft data even after retirement for historical mission lookups
```

**Why This Design:**
- **Status-based filtering** controls subscriptions
- Aircraft data persists across many missions
- Status changes (Mission Ready → Maintenance → Mission Ready) don't require re-sync
- Only evict when aircraft permanently leaves the base
- Critical reference data for mission planning

---

## Short-Lived Transactional Data

### 3. `missions` Collection
**Lifecycle: SHORT-LIVED** - Evicted 30 days after completion.

The primary collection for mission planning and execution.

```json
{
  "_id": "UUID",
  "missionNumber": "string - Unique identifier e.g., 'MSN-2026-001'",
  "missionType": "string - 'Training', 'Combat', 'Transport', 'Reconnaissance', 'Refueling'",
  "status": "string - 'Planned', 'Briefed', 'In Progress', 'Completed', 'Aborted'",

  "aircraftTailNumber": "string - Denormalized from aircraft",
  "aircraftType": "string - Denormalized e.g., 'F-16C'",

  "commandPilotName": "string - Denormalized crew name",
  "commandPilotId": "string - Reference to crew_members._id",

  "departureBase": "string - REQUIRED FOR SUBSCRIPTIONS",
  "destinationBase": "string - Base name or 'N/A'",

  "scheduledDepartureTime": "ISO 8601 datetime",
  "actualDepartureTime": "ISO 8601 datetime (nullable)",
  "scheduledArrivalTime": "ISO 8601 datetime",
  "actualArrivalTime": "ISO 8601 datetime (nullable)",

  "flightPlanRoute": "string - Brief route description",
  "missionObjective": "string - Primary mission goal",
  "weatherConditions": "string - 'VFR', 'IFR', 'MVFR'",

  "fuelRequired": "number - Gallons",
  "fuelUsed": "number - Gallons (nullable)",

  "createdAt": "ISO 8601 datetime",
  "updatedAt": "ISO 8601 datetime",
  "completedAt": "ISO 8601 datetime (nullable) - REQUIRED FOR EVICTION",
  "createdBy": "string - User who created mission"
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to active missions only (RECOMMENDED - minimizes data on device)
SELECT * FROM missions
WHERE departureBase = 'Nellis AFB'
AND status IN ('Planned', 'Briefed', 'In Progress')

-- Subscribe to recent missions (last 7 days, including completed for debrief)
SELECT * FROM missions
WHERE departureBase = 'Nellis AFB'
AND createdAt > '2026-01-27T00:00:00Z'

-- Operations center: Subscribe to all missions (including completed for tracking)
SELECT * FROM missions WHERE departureBase = 'Nellis AFB'
```

**Eviction Query Examples:**
```sql
-- Evict completed missions older than 30 days
EVICT FROM missions WHERE status = 'Completed' AND completedAt < '2025-12-03T00:00:00Z'

-- Evict aborted missions older than 7 days
EVICT FROM missions WHERE status = 'Aborted' AND updatedAt < '2026-01-27T00:00:00Z'
```

**Why This Design:**
- `departureBase` enables location-based subscriptions (key requirement)
- Denormalized `aircraftTailNumber` and `aircraftType` for display without fetching aircraft
- Denormalized `commandPilotName` for display without fetching crew
- `completedAt` enables time-based eviction of old missions
- Flat structure (no deep nesting) for efficient sync

---

### 2. `aircraft` Collection

Aircraft inventory and operational status.

```json
{
  "_id": "UUID",
  "tailNumber": "string - Unique e.g., '91-0409'",
  "aircraftType": "string - 'F-16C', 'C-130J', 'KC-135R', 'B-52H', 'F-35A'",
  "squadron": "string - e.g., '56th Fighter Wing'",

  "homeBase": "string - REQUIRED FOR SUBSCRIPTIONS",
  "currentLocation": "string - Current base or 'In Flight'",

  "status": "string - 'Mission Ready', 'Maintenance', 'Down for Repairs', 'Inspection'",
  "statusReason": "string - Brief explanation of status",

  "flightHours": "number - Total flight hours",
  "flightsSinceInspection": "number",

  "lastInspectionDate": "ISO 8601 datetime",
  "nextInspectionDue": "ISO 8601 datetime",

  "fuelCapacity": "number - Gallons",
  "maxRange": "number - Nautical miles",
  "maxAltitude": "number - Feet",

  "updatedAt": "ISO 8601 datetime"
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to aircraft at my base
SELECT * FROM aircraft WHERE homeBase = 'Nellis AFB'

-- Subscribe to mission-ready aircraft only
SELECT * FROM aircraft WHERE homeBase = 'Nellis AFB' AND status = 'Mission Ready'

-- Subscribe to aircraft needing inspection soon
SELECT * FROM aircraft WHERE homeBase = 'Nellis AFB' AND nextInspectionDue < '2026-02-15T00:00:00Z'
```

**Why This Design:**
- `homeBase` enables location-based subscriptions
- No eviction needed (persistent aircraft inventory)
- Status fields enable filtering for mission planning
- Denormalized capacity specs for quick reference

---


### 4. `crew_assignments` Collection
**Lifecycle: SHORT-LIVED** - Evicted 30 days after mission completion.

Links crew members to specific missions.

```json
{
  "_id": "UUID",
  "missionId": "string - Reference to missions._id - FOR EVICTION BY PARENT",
  "missionDepartureBase": "string - DENORMALIZED FOR SUBSCRIPTIONS ONLY",

  "crewMemberId": "string - Reference to crew_members._id",
  "crewMemberName": "string - Denormalized for display",
  "crewMemberRank": "string - Denormalized for display",

  "crewPosition": "string - Role for this mission",
  "seatPosition": "string - Physical seat e.g., 'Left Seat', 'Right Seat', 'Jump Seat'",

  "assignedAt": "ISO 8601 datetime",
  "status": "string - 'Assigned', 'Briefed', 'Flying', 'Completed'",

  "briefingAttended": "boolean",
  "briefingAttendedAt": "ISO 8601 datetime (nullable)"
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to ALL crew assignments from my base (simpler approach)
SELECT * FROM crew_assignments WHERE missionDepartureBase = 'Nellis AFB'

-- Get assignments for specific crew member (filter locally after sync)
SELECT * FROM crew_assignments
WHERE missionDepartureBase = 'Nellis AFB'
AND crewMemberId = 'crew-uuid-123'
```

**Eviction Strategy:**
```javascript
// 1. Find old missions to evict
const oldMissions = await ditto.store.execute(`
  SELECT _id FROM missions
  WHERE status = 'Completed'
  AND completedAt < :thirtyDaysAgo
`);

// 2. Evict missions
await ditto.store.execute(`
  EVICT FROM missions
  WHERE status = 'Completed'
  AND completedAt < :thirtyDaysAgo
`);

// 3. Evict crew assignments by missionId
for (const mission of oldMissions) {
  await ditto.store.execute(`
    EVICT FROM crew_assignments
    WHERE missionId = '${mission._id}'
  `);
}
```

**Why This Design:**
- `missionDepartureBase` enables location-based subscriptions (required)
- `missionId` enables targeted eviction when parent mission is evicted
- **No need for missionStatus** - filter by parent missions in app code
- **No need for missionCompletedAt** - evict by missionId when mission is evicted
- Simpler schema, less denormalization to maintain

---

### 5. `flight_logs` Collection
**Lifecycle: SHORT-LIVED** - Evicted 90 days after mission completion (kept longer for records).

Individual flight records (digital AFTO Form 781).

```json
{
  "_id": "UUID",
  "missionId": "string - Reference to missions._id - FOR EVICTION BY PARENT",
  "missionDepartureBase": "string - DENORMALIZED FOR SUBSCRIPTIONS ONLY",

  "crewMemberId": "string - Reference to crew_members._id",
  "crewMemberName": "string - Denormalized for display",

  "aircraftTailNumber": "string - Denormalized",
  "aircraftType": "string - Denormalized",

  "flightDate": "ISO 8601 date",
  "departureTime": "ISO 8601 datetime",
  "arrivalTime": "ISO 8601 datetime",

  "flightDuration": "number - Minutes",
  "instrumentTime": "number - Minutes (IFR time)",
  "nightTime": "number - Minutes",

  "landingsDay": "number - Count",
  "landingsNight": "number - Count",
  "touchAndGo": "number - Count",

  "remarks": "string - Flight notes, issues encountered",

  "loggedAt": "ISO 8601 datetime",
  "loggedBy": "string - Crew member who created log"
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to flight logs for missions from my base
SELECT * FROM flight_logs WHERE missionDepartureBase = 'Nellis AFB'

-- Subscribe to recent flight logs (last 30 days)
SELECT * FROM flight_logs
WHERE missionDepartureBase = 'Nellis AFB'
AND flightDate > '2026-01-03T00:00:00Z'
```

**Eviction Strategy:**
Evict flight logs by missionId when parent mission is evicted:
```javascript
// First, find missions to evict
const oldMissions = await ditto.store.execute(`
  SELECT _id FROM missions
  WHERE status = 'Completed'
  AND completedAt < '2025-11-03T00:00:00Z'
`);

// Evict missions
await ditto.store.execute(`
  EVICT FROM missions
  WHERE status = 'Completed'
  AND completedAt < '2025-11-03T00:00:00Z'
`);

// Evict flight logs by missionId
for (const mission of oldMissions) {
  await ditto.store.execute(`
    EVICT FROM flight_logs WHERE missionId = '${mission._id}'
  `);
}
```

**Why This Design:**
- `missionDepartureBase` denormalized enables location-based subscriptions (critical!)
- `missionId` enables targeted eviction when parent mission is evicted
- **No missionStatus** - Not needed; subscribe by location, filter by parent in app
- **No missionCompletedAt** - Not needed; evict by missionId when mission is evicted
- Without missionDepartureBase, you'd need to sync ALL flight logs then filter in memory
- Denormalized mission/aircraft/crew info for display
- Can be logged offline during/after flight, syncs when connected

---

### 6. `maintenance_status` Collection
**Lifecycle: SHORT-LIVED** - Evicted 60 days after resolution.

Aircraft maintenance issues and work orders.

```json
{
  "_id": "UUID",
  "aircraftId": "string - Reference to aircraft._id",
  "aircraftTailNumber": "string - Denormalized for display",
  "aircraftType": "string - Denormalized",
  "aircraftHomeBase": "string - DENORMALIZED FOR SUBSCRIPTIONS",

  "discrepancyType": "string - 'Critical', 'Major', 'Minor', 'Inspection'",
  "description": "string - Issue description",
  "workOrderNumber": "string - Maintenance tracking number (nullable)",

  "reportedBy": "string - Name of reporter",
  "reportedAt": "ISO 8601 datetime",

  "status": "string - 'Open', 'In Progress', 'Awaiting Parts', 'Resolved'",
  "priority": "number - 1 (highest) to 5 (lowest)",

  "assignedTo": "string - Maintenance crew/unit (nullable)",
  "estimatedCompletionTime": "ISO 8601 datetime (nullable)",

  "resolvedAt": "ISO 8601 datetime (nullable) - REQUIRED FOR EVICTION",
  "resolutionNotes": "string (nullable)",

  "groundsAircraft": "boolean - Prevents flight if true",
  "partsRequired": "object - Map of part numbers to quantities"
}
```

**Parts Required Example:**
```json
{
  "partsRequired": {
    "NSN-5840-01-234-5678": 2,
    "NSN-1560-00-987-6543": 1
  }
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to maintenance for aircraft at my base
SELECT * FROM maintenance_status WHERE aircraftHomeBase = 'Nellis AFB'

-- Subscribe to open/critical maintenance only
SELECT * FROM maintenance_status
WHERE aircraftHomeBase = 'Nellis AFB'
AND status IN ('Open', 'In Progress')
AND discrepancyType = 'Critical'

-- Subscribe to maintenance that grounds aircraft
SELECT * FROM maintenance_status
WHERE aircraftHomeBase = 'Nellis AFB'
AND groundsAircraft = true
AND status != 'Resolved'
```

**Eviction Query Examples:**
```sql
-- Evict resolved maintenance older than 60 days
EVICT FROM maintenance_status
WHERE status = 'Resolved'
AND resolvedAt < '2025-12-03T00:00:00Z'
```

**Why This Design:**
- `aircraftHomeBase` denormalized enables location-based subscriptions
- Maintenance crew at a base only sync maintenance for their aircraft
- `resolvedAt` enables eviction of old resolved issues
- Map for parts (not array) follows Ditto best practices

---

### 7. `mission_briefs` Collection
**Lifecycle: SHORT-LIVED** - Evicted 30 days after mission completion.

Pre-flight briefing information.

```json
{
  "_id": "UUID",
  "missionId": "string - Reference to missions._id - FOR EVICTION BY PARENT",
  "missionDepartureBase": "string - DENORMALIZED FOR SUBSCRIPTIONS ONLY",

  "briefingType": "string - 'Pre-Flight', 'Weather', 'Intelligence', 'Emergency Procedures', 'Debrief'",
  "briefingTitle": "string - Brief descriptive title",
  "briefingContent": "string - Main briefing text",

  "briefedBy": "string - Officer conducting brief",
  "briefedByRank": "string - Rank of briefer",

  "briefedAt": "ISO 8601 datetime",
  "briefingLocation": "string - Building/room",

  "attendees": "object - Map of crew member IDs to attendance status",
  "attachmentCount": "number - Count of attachments (not the attachments themselves)"
}
```

**Attendees Example:**
```json
{
  "attendees": {
    "crew-uuid-1": "Present",
    "crew-uuid-2": "Present",
    "crew-uuid-3": "Absent - Medical"
  }
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to briefs for missions from my base
SELECT * FROM mission_briefs WHERE missionDepartureBase = 'Nellis AFB'

-- Subscribe to recent briefs (last 7 days)
SELECT * FROM mission_briefs
WHERE missionDepartureBase = 'Nellis AFB'
AND briefedAt > '2026-01-27T00:00:00Z'
```

**Eviction Strategy:**
Evict mission briefs by missionId when parent mission is evicted:
```javascript
// First, find missions to evict
const oldMissions = await ditto.store.execute(`
  SELECT _id FROM missions
  WHERE status = 'Completed'
  AND completedAt < '2025-12-03T00:00:00Z'
`);

// Evict missions
await ditto.store.execute(`
  EVICT FROM missions
  WHERE status = 'Completed'
  AND completedAt < '2025-12-03T00:00:00Z'
`);

// Evict mission briefs by missionId
for (const mission of oldMissions) {
  await ditto.store.execute(`
    EVICT FROM mission_briefs WHERE missionId = '${mission._id}'
  `);
}
```

**Why This Design:**
- `missionDepartureBase` enables location-based subscriptions
- `missionId` enables targeted eviction when parent mission is evicted
- **No missionStatus** - Not needed; subscribe by location, filter by parent in app
- **No missionCompletedAt** - Not needed; evict by missionId when mission is evicted
- Map for attendees (not array) follows Ditto best practices
- Can create briefs offline, sync when connected
- Attachments tracked by count; actual attachments handled separately via Ditto attachments API

---

### 8. `status_updates` Collection
**Lifecycle: SHORT-LIVED** - Evicted 7-14 days after mission completion.

Real-time mission status updates (perfect for demonstrating Ditto sync!).

```json
{
  "_id": "UUID",
  "missionId": "string - Reference to missions._id - FOR EVICTION BY PARENT",
  "missionDepartureBase": "string - DENORMALIZED FOR SUBSCRIPTIONS ONLY",

  "updateType": "string - 'Position', 'Status Change', 'Weather', 'Emergency', 'Crew Note', 'Fuel Status'",
  "updatePriority": "string - 'Routine', 'Important', 'Urgent', 'Emergency'",
  "message": "string - Update text",

  "latitude": "number (nullable) - Decimal degrees",
  "longitude": "number (nullable) - Decimal degrees",
  "altitude": "number (nullable) - Feet MSL",
  "heading": "number (nullable) - Degrees",
  "speed": "number (nullable) - Knots",

  "createdBy": "string - Crew member name or 'System'",
  "createdByPosition": "string - Crew position",
  "createdAt": "ISO 8601 datetime",
  "syncedAt": "ISO 8601 datetime (nullable) - When update reached server"
}
```

**Subscription Query Examples:**
```sql
-- Subscribe to status updates for missions from my base
SELECT * FROM status_updates WHERE missionDepartureBase = 'Nellis AFB'

-- Subscribe to recent updates (last 24 hours)
SELECT * FROM status_updates
WHERE missionDepartureBase = 'Nellis AFB'
AND createdAt > '2026-02-02T00:00:00Z'

-- Subscribe to high-priority updates only
SELECT * FROM status_updates
WHERE missionDepartureBase = 'Nellis AFB'
AND updatePriority IN ('Urgent', 'Emergency')
```

**Eviction Strategy:**
Evict status updates by missionId when parent mission is evicted:
```javascript
// First, find missions to evict
const oldMissions = await ditto.store.execute(`
  SELECT _id FROM missions
  WHERE status = 'Completed'
  AND completedAt < '2026-01-20T00:00:00Z'
`);

// Evict missions
await ditto.store.execute(`
  EVICT FROM missions
  WHERE status = 'Completed'
  AND completedAt < '2026-01-20T00:00:00Z'
`);

// Evict status updates by missionId
for (const mission of oldMissions) {
  await ditto.store.execute(`
    EVICT FROM status_updates WHERE missionId = '${mission._id}'
  `);
}
```

**Why This Design:**
- `missionDepartureBase` enables location-based subscriptions (key!)
- `missionId` enables targeted eviction when parent mission is evicted
- **No missionStatus** - Not needed; subscribe by location, filter by parent in app
- **No missionCompletedAt** - Not needed; evict by missionId when mission is evicted
- Small documents for fast creation and sync (perfect for offline→online scenarios)
- Created offline in aircraft, syncs automatically when connectivity restored
- Operations center can watch real-time updates as aircraft reconnect
- Perfect for demonstrating Ditto's real-time sync and offline capabilities

---

## Subscription Patterns Summary

### Location-Based Subscriptions with Status Filtering

**CRITICAL: Long-lived vs Short-lived Data**

```javascript
const myBase = 'Nellis AFB';

// === LONG-LIVED REFERENCE DATA (Status-based filtering) ===

// Subscribe to ACTIVE crew at my base (persistent data)
ditto.store.execute(`
  SELECT * FROM crew_members
  WHERE homeBase = '${myBase}'
  AND status = 'Active'
`);

// Subscribe to OPERATIONAL aircraft at my base (persistent data)
ditto.store.execute(`
  SELECT * FROM aircraft
  WHERE homeBase = '${myBase}'
  AND status IN ('Mission Ready', 'Maintenance', 'Inspection')
`);

// === SHORT-LIVED TRANSACTIONAL DATA (Time/status-based filtering) ===

// Subscribe to ACTIVE missions only (evict when completed)
ditto.store.execute(`
  SELECT * FROM missions
  WHERE departureBase = '${myBase}'
  AND status IN ('Planned', 'Briefed', 'In Progress')
`);

// Subscribe to crew assignments for ACTIVE missions only
ditto.store.execute(`
  SELECT * FROM crew_assignments
  WHERE missionDepartureBase = '${myBase}'
  AND missionStatus IN ('Planned', 'Briefed', 'In Progress')
`);

// Subscribe to recent flight logs (last 30 days)
const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString();
ditto.store.execute(`
  SELECT * FROM flight_logs
  WHERE missionDepartureBase = '${myBase}'
  AND flightDate > '${thirtyDaysAgo}'
`);

// Subscribe to OPEN maintenance only
ditto.store.execute(`
  SELECT * FROM maintenance_status
  WHERE aircraftHomeBase = '${myBase}'
  AND status IN ('Open', 'In Progress', 'Awaiting Parts')
`);

// Subscribe to briefs for ACTIVE missions only
ditto.store.execute(`
  SELECT * FROM mission_briefs
  WHERE missionDepartureBase = '${myBase}'
  AND missionStatus IN ('Planned', 'Briefed', 'In Progress')
`);

// Subscribe to recent status updates (last 24 hours)
const yesterday = new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString();
ditto.store.execute(`
  SELECT * FROM status_updates
  WHERE missionDepartureBase = '${myBase}'
  AND createdAt > '${yesterday}'
`);
```

### Time-Based Subscriptions (Secondary Pattern)

Limit sync to recent data:

```javascript
const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString();

// Only sync recent missions
ditto.store.execute(`
  SELECT * FROM missions
  WHERE departureBase = 'Nellis AFB'
  AND createdAt > '${sevenDaysAgo}'
`);

// Only sync active missions
ditto.store.execute(`
  SELECT * FROM missions
  WHERE departureBase = 'Nellis AFB'
  AND status IN ('Planned', 'Briefed', 'In Progress')
`);
```

### Role-Based Subscriptions (Tertiary Pattern)

Different roles subscribe to different data:

```javascript
// Flight crew: Only active mission data
ditto.store.execute(`
  SELECT * FROM missions
  WHERE departureBase = 'Nellis AFB'
  AND status IN ('Briefed', 'In Progress')
`);

// Maintenance: Only maintenance and aircraft data
ditto.store.execute(`SELECT * FROM aircraft WHERE homeBase = 'Nellis AFB'`);
ditto.store.execute(`SELECT * FROM maintenance_status WHERE aircraftHomeBase = 'Nellis AFB'`);

// Operations: Everything for coordination
ditto.store.execute(`SELECT * FROM missions WHERE departureBase = 'Nellis AFB'`);
ditto.store.execute(`SELECT * FROM status_updates WHERE missionDepartureBase = 'Nellis AFB'`);
```

---

## Eviction Strategy

### Recommended Eviction Schedule

Run evictions **once daily during off-peak hours** (e.g., 3 AM local time) to minimize network impact.

### Eviction Queries

```javascript
const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString();
const sixtyDaysAgo = new Date(Date.now() - 60 * 24 * 60 * 60 * 1000).toISOString();
const ninetyDaysAgo = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000).toISOString();

// Evict old completed missions (30 days)
ditto.store.execute(`
  EVICT FROM missions
  WHERE status = 'Completed'
  AND completedAt < '${thirtyDaysAgo}'
`);

// Evict crew assignments by missionId (when parent mission is evicted)
// This would be done after evicting missions - loop through old missions and evict their crew assignments
for (const mission of oldMissions) {
  ditto.store.execute(`
    EVICT FROM crew_assignments WHERE missionId = '${mission._id}'
  `);
}

// Evict flight logs by missionId (when parent mission is evicted)
// This would be done after evicting missions - loop through old missions and evict their flight logs
for (const mission of oldMissions) {
  ditto.store.execute(`
    EVICT FROM flight_logs WHERE missionId = '${mission._id}'
  `);
}

// Evict old resolved maintenance (60 days)
ditto.store.execute(`
  EVICT FROM maintenance_status
  WHERE status = 'Resolved'
  AND resolvedAt < '${sixtyDaysAgo}'
`);

// Evict mission briefs by missionId (when parent mission is evicted)
for (const mission of oldMissions) {
  ditto.store.execute(`
    EVICT FROM mission_briefs WHERE missionId = '${mission._id}'
  `);
}

// Evict status updates by missionId (when parent mission is evicted)
for (const mission of oldMissions) {
  ditto.store.execute(`
    EVICT FROM status_updates WHERE missionId = '${mission._id}'
  `);
}
```

### Eviction Best Practices

1. **Cancel subscriptions before eviction** to prevent immediate re-sync
2. **Evict related data together** (mission + assignments + logs + briefs + updates)
3. **Use Big Peer for centralized eviction** when possible
4. **Monitor storage usage** to determine eviction frequency
5. **Keep eviction windows conservative** (don't evict too aggressively)

---

## Data Relationships (Handled in Application Layer)

Since Ditto doesn't support JOINs, relationships are managed in application code using Data Access Objects (DAOs).

### Example: Getting Complete Mission Information

```javascript
// 1. Subscribe to and fetch mission
const mission = await ditto.store.execute(
  "SELECT * FROM missions WHERE _id = :missionId",
  { missionId: "mission-uuid-123" }
);

// 2. Fetch related crew assignments (already synced via subscription)
const crewAssignments = await ditto.store.execute(
  "SELECT * FROM crew_assignments WHERE missionId = :missionId",
  { missionId: "mission-uuid-123" }
);

// 3. Fetch status updates (already synced)
const statusUpdates = await ditto.store.execute(
  "SELECT * FROM status_updates WHERE missionId = :missionId ORDER BY createdAt DESC LIMIT 10",
  { missionId: "mission-uuid-123" }
);

// 4. Combine in memory using DAO
const completeMission = {
  ...mission,
  crew: crewAssignments,
  recentUpdates: statusUpdates
};
```

### Example: Getting Aircraft with Open Maintenance

```javascript
// 1. Fetch aircraft
const aircraft = await ditto.store.execute(
  "SELECT * FROM aircraft WHERE homeBase = 'Nellis AFB' AND status = 'Maintenance'"
);

// 2. Fetch related open maintenance (already synced)
const openMaintenance = await ditto.store.execute(
  "SELECT * FROM maintenance_status WHERE aircraftHomeBase = 'Nellis AFB' AND status = 'Open'"
);

// 3. Group maintenance by aircraft in memory
const aircraftWithMaintenance = aircraft.map(ac => ({
  ...ac,
  openIssues: openMaintenance.filter(m => m.aircraftTailNumber === ac.tailNumber)
}));
```

---

## Key Advantages of This Design

1. **Efficient Subscriptions**: Every collection filterable by location - no need to sync entire mesh
2. **Offline-First**: All collections support offline creation, sync automatically when connected
3. **Storage Management**: Eviction fields enable automatic cleanup of old data
4. **No JOINs Needed**: Denormalization provides display data without fetches
5. **Demonstration-Friendly**: Clear patterns for showing Ditto capabilities:
   - Real-time sync (status_updates)
   - Offline operation (flight_logs)
   - Collaboration (mission planning)
   - Storage management (eviction)
   - Role-based access (subscription patterns)

---

## Sample Data Requirements

To effectively demonstrate this system, generate:

- **5 bases** (representing different locations)
- **15 aircraft** (3 per base)
- **50 crew members** (10 per base)
- **20 missions** (mix of planned, in progress, completed across bases)
- **60 crew assignments** (~3 crew per mission)
- **40 flight logs** (for completed missions)
- **25 maintenance status entries** (mix of open/resolved)
- **30 mission briefs** (multiple per mission)
- **100 status updates** (showing real-time coordination)

**Total**: ~340 documents across 8 collections

This provides enough data to demonstrate subscriptions, relationships, and eviction without overwhelming the demo.
