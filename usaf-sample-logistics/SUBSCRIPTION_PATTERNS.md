# Ditto Subscription Patterns - Critical Concept

## The Core Problem

**You cannot efficiently subscribe to data by IDs or lists.**

### ❌ What DOESN'T Work

```javascript
// Can't subscribe by single ID - would need separate subscription per mission
ditto.store.execute("SELECT * FROM crew_assignments WHERE missionId = 'mission-123'")

// Can't subscribe by list - IDs change constantly, not scalable
ditto.store.execute("SELECT * FROM crew_assignments WHERE missionId IN ('m1', 'm2', 'm3', ...)")

// Can't use time alone - you don't know mission duration
ditto.store.execute("SELECT * FROM crew_assignments WHERE createdAt > '2026-01-01'")
```

**Why this fails:**
1. Mission IDs are UUIDs - you don't know them upfront
2. New missions created constantly - can't maintain ID list
3. Would need to update subscriptions every time a mission is created/completed
4. Not performant - too many subscription changes

---

## ✅ The Solution: Aligned Subscription Fields

**Related data MUST be subscribable using the SAME fields as parent data.**

### Pattern: Subscribe by Location, Evict by Parent ID

```javascript
// Subscribe to missions at my base (active only)
const missionSubscription = ditto.store.execute(`
  SELECT * FROM missions
  WHERE departureBase = 'Nellis AFB'
  AND status IN ('Planned', 'Briefed', 'In Progress')
`);

// Subscribe to ALL crew assignments at my base (simpler!)
const crewAssignmentSubscription = ditto.store.execute(`
  SELECT * FROM crew_assignments
  WHERE missionDepartureBase = 'Nellis AFB'
`);
```

**How this works:**
1. When mission `MSN-2026-050` is created at Nellis AFB with status 'Planned':
   - Mission document syncs (matches subscription filter)
   - ALL crew assignment documents for that mission ALSO sync automatically
   - They have `missionDepartureBase = 'Nellis AFB'`

2. In your app code:
   - Display crew assignments for missions you have locally
   - Orphaned assignments (mission evicted) can be filtered out in app

3. When mission is evicted:
   - Evict mission by status + completedAt
   - Evict crew assignments by missionId (targeted eviction)

---

## Why Denormalization is Required

**crew_assignments Schema:**
```json
{
  "_id": "UUID",
  "missionId": "mission-123",              // For eviction by parent ID
  "missionDepartureBase": "Nellis AFB",    // ⭐ DENORMALIZED for subscription
  "crewMemberId": "crew-456",
  "crewMemberName": "Capt John Smith",
  ...
}
```

**Why this is minimal:**
- `missionDepartureBase` - Required for location-based subscription
- `missionId` - Required for targeted eviction when parent mission is evicted
- **NO missionStatus** - Not needed; filter by parent missions in app code
- **NO missionCompletedAt** - Not needed; evict by missionId when mission is evicted

Without `missionDepartureBase`, you'd have to:
1. Subscribe to ALL crew assignments from all bases (massive over-sync)
2. Fetch each mission separately to check if it's at your base
3. Filter in memory (defeats purpose of subscriptions)

---

## Complete Subscription Set for Mission Planning

All subscriptions use **the same base location and status filters**:

```javascript
const myBase = 'Nellis AFB';
const activeStatuses = ['Planned', 'Briefed', 'In Progress'];

// 1. Missions
ditto.store.execute(`
  SELECT * FROM missions
  WHERE departureBase = '${myBase}'
  AND status IN ('${activeStatuses.join("','")}')
`);

// 2. Crew Assignments - Same location, no status filter needed
ditto.store.execute(`
  SELECT * FROM crew_assignments
  WHERE missionDepartureBase = '${myBase}'
`);

// 3. Flight Logs - SAME BASE (but different time filter for historical data)
ditto.store.execute(`
  SELECT * FROM flight_logs
  WHERE missionDepartureBase = '${myBase}'
  AND flightDate > '2026-01-03'
`);

// 4. Mission Briefs - SAME FILTERS
ditto.store.execute(`
  SELECT * FROM mission_briefs
  WHERE missionDepartureBase = '${myBase}'
  AND missionStatus IN ('${activeStatuses.join("','")}')
`);

// 5. Status Updates - SAME BASE (recent only)
ditto.store.execute(`
  SELECT * FROM status_updates
  WHERE missionDepartureBase = '${myBase}'
  AND createdAt > '${yesterday}'
`);

// 6. Maintenance - Uses aircraft base
ditto.store.execute(`
  SELECT * FROM maintenance_status
  WHERE aircraftHomeBase = '${myBase}'
  AND status IN ('Open', 'In Progress')
`);
```

**Key Point**: All mission-related collections use `missionDepartureBase` for location filtering. Child collections (like crew_assignments) don't need status filters - just sync everything at the location and filter by parent in app code.

---

## Real-World Scenario

### Scenario: New Mission Created

1. **Operations creates mission at Nellis AFB**
   ```javascript
   // Creates mission MSN-2026-100
   {
     "_id": "abc-123",
     "missionNumber": "MSN-2026-100",
     "departureBase": "Nellis AFB",
     "status": "Planned",
     ...
   }
   ```

2. **Assigns 3 crew members**
   ```javascript
   // Creates 3 crew assignment documents
   [
     {
       "_id": "ca-1",
       "missionId": "abc-123",
       "missionDepartureBase": "Nellis AFB",  // ⭐ For subscription
       "crewMemberId": "pilot-1",
       ...
     },
     {
       "_id": "ca-2",
       "missionId": "abc-123",
       "missionDepartureBase": "Nellis AFB",  // ⭐ For subscription
       "crewMemberId": "copilot-1",
       ...
     },
     {
       "_id": "ca-3",
       "missionId": "abc-123",
       "missionDepartureBase": "Nellis AFB",  // ⭐ For subscription
       "crewMemberId": "engineer-1",
       ...
     }
   ]
   ```

3. **All devices at Nellis AFB with active subscriptions receive:**
   - 1 mission document (matches `departureBase = 'Nellis AFB' AND status = 'Planned'`)
   - 3 crew assignment documents (match `missionDepartureBase = 'Nellis AFB'`)
   - **No subscription updates needed!** Data automatically syncs because it matches existing filters.

4. **Devices at other bases (Luke AFB, Edwards AFB) receive:**
   - **Nothing** - mission and assignments don't match their base filters

---

## What Happens When Mission Completes

1. **Mission status changes to 'Completed'**
   ```javascript
   // Mission document updates
   {
     "_id": "abc-123",
     "status": "Completed",  // Changed
     "completedAt": "2026-02-03T...",
     ...
   }
   ```

2. **Crew assignments DON'T need to update** (simpler!)
   - No denormalized status to maintain
   - Crew assignments stay unchanged
   - Remain synced to devices (subscription still matches by location)

3. **Devices with active-only mission subscriptions**
   ```javascript
   // Mission subscription: status IN ('Planned', 'Briefed', 'In Progress')
   // Mission no longer matches - stops syncing updates

   // Crew assignment subscription: missionDepartureBase = 'Nellis AFB'
   // Crew assignments STILL match - continue syncing
   // App code filters out orphaned assignments (no local mission)
   ```

4. **Later: Eviction**
   ```javascript
   // 30 days later, evict completed missions
   const oldMissions = await ditto.store.execute(`
     SELECT _id FROM missions
     WHERE status = 'Completed'
     AND completedAt < '2026-01-03T00:00:00Z'
   `);

   EVICT FROM missions
   WHERE status = 'Completed'
   AND completedAt < '2026-01-03T00:00:00Z'

   // Evict crew assignments by missionId
   for (const mission of oldMissions) {
     EVICT FROM crew_assignments WHERE missionId = '${mission._id}'
   }
   ```

---

## Anti-Pattern: Wrong Denormalization

### ❌ Bad Schema - Can't Subscribe Efficiently

```json
{
  "_id": "ca-1",
  "missionId": "abc-123",
  "crewMemberId": "pilot-1"
  // ❌ Missing missionDepartureBase - can't filter by base
  // ❌ Have to sync ALL crew assignments, filter in memory
}
```

**Problem**: To get crew assignments for Nellis AFB missions, you'd have to:
1. Sync ALL crew assignments from ALL bases (massive over-sync)
2. Fetch each mission separately to check `departureBase`
3. Filter in memory (defeats purpose of subscriptions)

### ✅ Good Schema - Subscription-Ready

```json
{
  "_id": "ca-1",
  "missionId": "abc-123",                   // For eviction by parent
  "missionDepartureBase": "Nellis AFB",     // ⭐ For subscription filtering
  "crewMemberId": "pilot-1",
  "crewMemberName": "Capt John Smith"       // Denormalized for display
}
```

**Simpler**: Only denormalize what's needed for subscriptions. Evict by missionId when parent is evicted.

---

## Key Takeaways

1. **Design subscriptions FIRST, then design schema**
   - "How will devices subscribe to this data?"
   - "What fields enable efficient filtering?"

2. **Related data must use subscription-friendly fields**
   - Missions: `WHERE departureBase = X AND status = Y`
   - Crew assignments: `WHERE missionDepartureBase = X` (location only)
   - Don't over-denormalize - only copy what's needed for subscriptions

3. **Denormalize minimally**
   - Copy `departureBase` → `missionDepartureBase` (for subscription)
   - Keep `missionId` (for targeted eviction)
   - DON'T copy status/completedAt unless truly needed

4. **Subscriptions should be stable**
   - Don't change subscriptions constantly
   - Filter by location (very stable)
   - Optionally filter by status for parent data only
   - Not by IDs or rapidly changing fields

5. **Evict by parent ID, not by denormalized fields**
   - Find missions to evict (by status + completedAt)
   - Evict those missions
   - Evict related data by missionId (targeted, efficient)

---

## Design Checklist

For any new collection related to an existing parent:

- [ ] What subscription filter is used for the parent?
- [ ] Copy ALL subscription fields into child documents
- [ ] Copy timestamp fields needed for eviction
- [ ] Verify child documents can be subscribed using same filter as parent
- [ ] Test: If parent syncs, does child sync automatically?
- [ ] Test: If parent stops syncing (status change), does child stop syncing?

**If the answer to any test is "no", you need more denormalization.**
