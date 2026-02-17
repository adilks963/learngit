# Route Care Tracking Cron Job Documentation

## Group 1: Cron Scheduling & Entry Point

### Overview
The tracking system runs every 10 seconds via a shell loop that executes the cron job to update vehicle positions and stop statuses.

### Components

#### 1. Shell Loop (`run.sh`)
**Location:** [run.sh:165-171](../run.sh#L165-L171)

```bash
while true; do
    python cron.py >> ../../logs/cron.log 2>&1
    sleep 1  # Configurable interval (1s in local dev)
done
```

**What it does:**
- Runs in background as part of application startup
- Executes `cron.py` at configured interval
- **Interval is environment-specific:** 1s (local dev for real-time updates), may be higher in staging/production
- Logs output to `logs/cron.log`
- Continues until application stops

#### 2. Cron Entry Point (`cron.py`)
**Location:** [route-care/poc/cron.py:12-15](../route-care/poc/cron.py#L12-L15)

```python
import asyncio
from helper.cron import run_tracking, run_delay_check

if __name__ == "__main__":
    asyncio.run(run_tracking())
```

**What it does:**
- Entry point called by shell loop
- Runs async tracking function
- Single execution per call (not a loop itself)

#### 3. Tracking Function (`helper/cron.py`)
**Location:** [route-care/poc/helper/cron.py:778-781](../route-care/poc/helper/cron.py#L778-L781)

```python
async def run_tracking():
    route_ids, trip_ids, route_trip_pairs = await get_active_session()
    if route_trip_pairs:
        await update_tracking(route_trip_pairs)
```

**What it does:**
- Gets active sessions (route/trip pairs that need tracking)
- Calls `update_tracking()` which executes the main SQL query
- Processes all active trips in single execution

### Execution Flow

```
run.sh loop (every 10s)
    ↓
cron.py
    ↓
run_tracking()
    ↓
get_active_session() → [(route_id=1, trip_id=4), ...]
    ↓
update_tracking(route_trip_pairs)
    ↓
route_tracking() SQL query → Updates database
    ↓
Exit (wait 10s, repeat)
```

### Key Technical Details

| Aspect | Value | Location |
|--------|-------|----------|
| Interval | Configurable (1s local, varies by env) | run.sh:170 |
| Execution Model | Synchronous (waits for completion) | run.sh:169 |
| Logging | logs/cron.log | run.sh:169 |
| Active Sessions | Auto-detected via query | helper/cron.py |
| Entry Method | Async | cron.py:15 |

### Example Execution

**Time: 14:27:00**
1. Shell loop triggers `python cron.py`
2. Gets active sessions: `[(route_id=4, trip_id=1)]`
3. Runs tracking query for trip 1
4. Updates vehicle position, stop statuses, reached stops
5. Execution completes in ~500ms
6. Shell waits (interval: 1s in local dev)

**Time: 14:27:01**
- Repeat

---

**Status:** ✅ Complete
**Next Group:** Route & Session Setup

---

## Group 2: Route & Session Setup

### Overview
Builds the route geometry by connecting road network segments and links it to active sessions/trips that need tracking.

### Database Tables

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `route` | Route metadata | id, name, org_id, start_stop, end_stop |
| `session` | Tracking session | id, route_id, vehicle_id, direction (Forward/Backward) |
| `trip` | Individual trip instance | id, session_id, trip_date, start_time, end_time |
| `routeedge` | Route composition | id (PK), route_id, edge_id (FK to roadnetwork), edge_order |
| `roadnetwork` | Road segments | id, source (Point), target (Point), geom (LineString), cost |

**Note:** `roadnetwork.geom` contains the complete path including curves between source and target points.

### SQL CTE: `route_geoms`
**Location:** [helper/query.py:330-347](../route-care/poc/helper/query.py#L330-L347)

```sql
SELECT
    r.org_id,
    r.id as route_id,
    s.vehicle_id,
    s.direction as session_direction,
    r.name as route_name,
    ST_MakeLine(rn.geom ORDER BY re.edge_order) AS route_geom,
    s.id as session_id,
    s.name as session_name,
    t.id as trip_id,
    ST_X(r.start_stop) AS start_lat, ST_Y(r.start_stop) AS start_lon,
    ST_X(r.end_stop) AS end_lat, ST_Y(r.end_stop) AS end_lon,
    CASE WHEN s.direction = 'Forward' THEN 1 ELSE -1 END AS direction_factor
FROM route r
JOIN session s ON r.id = s.route_id
JOIN trip t ON t.session_id = s.id
JOIN routeedge re ON r.id = re.route_id
JOIN roadnetwork rn ON re.edge_id = rn.id
WHERE (r.id = 4 AND t.id = 1)  -- Active route/trip pairs
GROUP BY r.id, s.vehicle_id, r.name, s.id, s.name, t.id, s.direction
```

### How Route Geometry is Built

**Step 1: Route Edges Define Sequence**
```
routeedge table for route_id=4:
route_id | edge_id | edge_order
---------|---------|------------
4        | 12      | 1
4        | 45      | 2
4        | 78      | 3
4        | 23      | 4
```

**Step 2: Fetch Road Geometries**
```
roadnetwork table (simplified):
id  | source        | target        | geom (LineString with all curve points)
----|---------------|---------------|----------------------------------------
12  | (76.5, 9.1)   | (76.51, 9.11) | LineString(76.5 9.1, 76.502 9.103, 76.505 9.106, 76.51 9.11)
45  | (76.51, 9.11) | (76.52, 9.12) | LineString(76.51 9.11, 76.513 9.113, 76.517 9.117, 76.52 9.12)
78  | (76.52, 9.12) | (76.53, 9.13) | LineString(76.52 9.12, 76.525 9.125, 76.53 9.13)
23  | (76.53, 9.13) | (76.54, 9.14) | LineString(76.53 9.13, 76.535 9.135, 76.54 9.14)
```

**Note:** 
- `source` and `target` are endpoints only
- `geom` contains the complete path with intermediate curve points
- Query uses only `geom` column for route construction

**Step 3: ST_MakeLine Connects Them**
```sql
ST_MakeLine(rn.geom ORDER BY re.edge_order)
```
Result: Single LineString from start to end with all curves preserved
```
LineString(
  76.5 9.1, 76.502 9.103, 76.505 9.106, 76.51 9.11,     ← segment 12
  76.51 9.11, 76.513 9.113, 76.517 9.117, 76.52 9.12,   ← segment 45
  76.52 9.12, 76.525 9.125, 76.53 9.13,                  ← segment 78
  76.53 9.13, 76.535 9.135, 76.54 9.14                   ← segment 23
)
```

### Direction Factor

**Purpose:** Determines if vehicle should travel forward or backward on route geometry

| session.direction | direction_factor | Meaning |
|------------------|------------------|---------|
| 'Forward' | 1 | Follow geometry 0.0 → 1.0 |
| 'Backward' | -1 | Follow geometry 1.0 → 0.0 |

### Example Output

**Input:**
- Route 4 with 4 road segments
- Session direction: 'Backward'
- Trip 1 active today
- Vehicle 1 assigned

**Output (route_geoms CTE):**
```
org_id: 1
route_id: 4
vehicle_id: 1
session_direction: 'Backward'
route_name: 'Route 4'
route_geom: LineString(full path)
session_id: 2
session_name: 'Morning Route'
trip_id: 1
start_lat: 9.1, start_lon: 76.5
end_lat: 9.14, end_lon: 76.54
direction_factor: -1
```

### Key Technical Details

**ST_MakeLine Requirements:**
- Must produce `ST_LineString` (not MultiLineString)
- Road segments must connect end-to-end
- `ORDER BY edge_order` is critical
- `HAVING ST_GeometryType(...) = 'ST_LineString'` filters invalid routes

**Active Trip Detection:**
- Only trips without `end_time` are tracked
- Filter applied via `route_trip_pairs` parameter
- Multiple routes/trips processed in single query

---

**Status:** ✅ Complete
**Next Group:** Vehicle Location & Deviation Check

---

## Group 3: Vehicle Location & Deviation Check

### Overview
Fetches current vehicle position and determines if vehicle is on the assigned route or has deviated.

### Vehicle Table
**Key Columns:**
- `current_location` - Current GPS position (Point geometry)
- `previous_location` - Last GPS position (Point geometry)
- `speed` - Current speed in km/h
- `last_update` - Timestamp of last GPS update

**Updated by:** GPS Listener service (runs separately, updates every few seconds)

### SQL CTE: `deviation_check`
**Location:** [helper/query.py:365-428](../route-care/poc/helper/query.py#L365-L428)

```sql
SELECT 
    v.id as vehicle_id,
    v.registration,
    v.previous_location,
    v.current_location,
    v.speed,
    vg.route_id AS assigned_route_id,
    vg.route_name AS assigned_route_name,
    vg.route_geom,
    CASE 
        WHEN v.current_location IS NULL THEN 0                      -- No GPS data
        WHEN NOT ST_DWithin(v.current_location::geography, 
                           vg.route_geom::geography, 50) THEN 1     -- Deviated
        ELSE 2                                                       -- On Route
    END AS status
FROM vehicle v
LEFT JOIN route_geoms vg ON v.id = vg.vehicle_id
WHERE vg.route_geom IS NOT NULL
```

### Status Codes

| Status | Name | Condition | Meaning |
|--------|------|-----------|---------|
| 0 | No Location | `current_location IS NULL` | GPS not available |
| 1 | Deviated | Distance > 50m from route | Vehicle off route |
| 2 | On Route | Within 50m of route | Vehicle on route |

### Deviation Logic

**ST_DWithin Check:**
```sql
ST_DWithin(current_location::geography, route_geom::geography, 50)
```

**What it does:**
- Converts to geography (meters, not degrees)
- Checks if vehicle is within **50 meters** of route geometry
- Returns `true` if within buffer, `false` if outside

### Example Scenario

**Setup:**
```
Vehicle: ID=1, Registration="KA01AB1234"
Current Location: (76.5123, 9.1456)
Route Geometry: LineString from (76.5, 9.1) to (76.55, 9.15)
```

**Case 1: On Route**
```
Vehicle at (76.5123, 9.1456)
Distance to route: 15 meters
ST_DWithin(..., 50) → TRUE
Status: 2 (On Route)
```

**Case 2: Deviated**
```
Vehicle at (76.5789, 9.1234) 
Distance to route: 280 meters
ST_DWithin(..., 50) → FALSE
Status: 1 (Deviated)
```

**Case 3: No GPS**
```
Vehicle current_location: NULL
Status: 0 (No Location)
```

### Direction Calculation (Part of this CTE)

**Purpose:** Determine if vehicle is moving toward route end or start

**Formula (Forward Direction):**
```sql
CASE
    -- Moving toward end (distance to end decreasing)
    WHEN distance(current → end) < distance(previous → end)
         AND distance(current → start) > distance(previous → start)
    THEN 1  -- Forward movement
    
    -- Moving toward start (distance to start decreasing)
    WHEN distance(current → start) < distance(previous → start)
         AND distance(current → end) > distance(previous → end)
    THEN 2  -- Backward movement
    
    ELSE 0  -- Stationary or unclear
END AS direction
```

**For Backward Routes:** Logic is reversed (start ↔ end swapped)

### Output from deviation_check CTE

**Example:**
```
vehicle_id: 1
registration: "KA01AB1234"
previous_location: Point(76.512, 9.145)
current_location: Point(76.5123, 9.1456)
speed: 45.5
assigned_route_id: 4
assigned_route_name: "Morning Route"
route_geom: LineString(...)
status: 2
direction: 1
session_id: 2
trip_id: 1
```

### Key Technical Details

**Buffer Distance:** 50 meters chosen to accommodate:
- GPS accuracy variations (~10-20m typical)
- Road width variations
- Minor route deviations (lane changes, slight detours)

**Geography vs Geometry:**
- `::geography` cast ensures distance in meters
- Without it, distance would be in degrees (not useful)

**NULL Handling:**
- No location takes precedence over other checks
- Prevents errors when GPS data is missing

---

**Status:** ✅ Complete
**Next Group:** Direction Handling

---

## Group 4: Direction Handling (Forward vs Backward Routes)

### Overview
Handles routes that can be traveled in both directions by adjusting calculations based on session direction.

### Session Direction

**Stored in:** `session.direction` column
- `'Forward'` - Vehicle travels route from start → end
- `'Backward'` - Vehicle travels route from end → start

**Route geometry stays the same** - only interpretation changes.

### Direction Factor

**Calculated in:** `route_geoms` CTE
```sql
CASE WHEN s.direction = 'Forward' THEN 1 ELSE -1 END AS direction_factor
```

| direction | direction_factor | Usage |
|-----------|------------------|--------|
| Forward | 1 | Use geometry as-is (0.0 → 1.0) |
| Backward | -1 | Invert calculations (1.0 → 0.0) |

### Example: Same Route, Different Directions

**Route Setup:**
```
Full route has 3 stops (A, B, C) along a geometry built from roadnetwork edges

Route Geometry: LineString from fraction 0.0 → 1.0
├─ Start Point at 0.0
├─ Stop A at fraction 0.2
├─ Stop B at fraction 0.5
├─ Stop C at fraction 0.8
└─ End Point at 1.0

(Between each stop: multiple roadnetwork edges make up the path)
```

**Forward Session (direction_factor = 1):**
- Vehicle should travel: Start(0.0) → Stop A(0.2) → Stop B(0.5) → Stop C(0.8) → End(1.0)
- Stop visit order: A(1st), B(2nd), C(3rd)
- Progress increases: 0.0 → 1.0

**Backward Session (direction_factor = -1):**
- Vehicle should travel: End(1.0) → Stop C(0.8) → Stop B(0.5) → Stop A(0.2) → Start(0.0)
- Stop visit order: C(1st), B(2nd), A(3rd) 
- Progress from route perspective: 1.0 → 0.0
- Progress from session perspective: 0% → 100% (going backward counts as progress)

### SQL CTE: `vehicle_progress`

**Location:** [helper/query.py:444-483](../route-care/poc/helper/query.py#L444-L483)

```sql
SELECT
    dc.vehicle_id,
    dc.direction_factor,
    COALESCE(nr.new_direction_factor, dc.direction_factor) AS effective_direction_factor,
    -- Progress calculation considering direction
    CASE 
        WHEN dc.route_geom IS NOT NULL THEN 
            CASE 
                WHEN dc.direction_factor = 1 THEN 
                    ST_LineLocatePoint(dc.route_geom, dc.current_location)
                ELSE 
                    1 - ST_LineLocatePoint(dc.route_geom, dc.current_location)
            END
        ELSE NULL
    END AS assigned_progress
FROM deviation_check dc
```

### Progress Calculation

**ST_LineLocatePoint:**
- Always returns fraction along geometry (0.0 to 1.0)
- Doesn't know about "forward" or "backward"

### Complete Flow - Who Does What

**Step 1: GPS Listener (Background Service - Separate Process)**
```python
# Updates vehicle table continuously
vehicle.current_location = Point(76.512, 9.145)
vehicle.speed = 45.5
vehicle.last_update = now()
```
- Runs independently of cron
- Just stores GPS data
- Doesn't know about routes or direction

**Step 2: Cron Job Triggers (Every 10 seconds)**
```python
asyncio.run(run_tracking())
```
- Executes single large SQL query
- Everything below happens inside SQL

**Step 3: Inside SQL Query (All At Once)**
```sql
-- A. Get route geometry
route_geom: LineString(A→B)

-- B. Get session direction
direction_factor: -1 (backward)

-- C. Call PostGIS function (database built-in)
ST_LineLocatePoint(route_geom, current_location)
   └─> Returns: 0.3 (raw fraction - always 0.0 to 1.0)

-- D. Our code inverts if needed (immediate calculation)
CASE 
    WHEN direction_factor = 1 THEN 0.3        -- Forward: use raw value
    ELSE 1 - 0.3 = 0.7                         -- Backward: invert it
END

-- E. Store result
assigned_progress: 0.7
```

**Important:** Steps C and D happen in **same SQL query execution** - not separate modules!

### Forward Route (direction_factor = 1):
```
Vehicle at raw position 0.3 on geometry
Progress = 0.3 (use as-is)
```

**Backward Route (direction_factor = -1):**
```
Vehicle at raw position 0.3 on geometry
Progress = 1 - 0.3 = 0.7 (invert)
```

### Real Example

**Setup:**
```
Route 4 geometry: 5km line from Point A to Point B
Session: Backward
Vehicle current location: 2km from Point A (raw fraction = 0.4)
```

**Forward Direction (direction_factor = 1):**
```
Raw fraction: 0.4
Progress: 0.4
Meaning: 40% complete (traveled from A toward B)
```

**Backward Direction (direction_factor = -1):**
```
Raw fraction: 0.4 (still 2km from A)
Progress: 1 - 0.4 = 0.6
Meaning: 60% complete (in backward direction, closer to A means more progress)
```

### Why This Matters

Without direction adjustment:
- Backward route would show wrong progress
- Stops would be in wrong order
- "Next stop" would be incorrect
- ETA calculations would fail

### Effective Direction Factor

**Purpose:** Handle route changes mid-trip

```sql
COALESCE(nr.new_direction_factor, dc.direction_factor) AS effective_direction_factor
```

**Scenarios:**
1. **Normal:** Vehicle on assigned route → use `direction_factor`
2. **Deviated:** Vehicle near different route → use `new_direction_factor`
3. **Helps with:** Route reassignment when vehicle deviates

### Visual Example

**Route with 5 Stops:**
```
[Stop A]-----[Stop B]-----[Stop C]-----[Stop D]-----[Stop E]
   0.0         0.25         0.5          0.75         1.0
   
(Lines = route geometry composed of roadnetwork edges)
```

**Forward Session:**
```
Vehicle currently at Stop C location (raw fraction = 0.5)
assigned_progress = 0.5 (no inversion)
Already visited: Stop A, Stop B
Current/Next: Stop C
Upcoming: Stop D, Stop E
```

**Backward Session:**
```
Vehicle at same physical location (Stop C, raw = 0.5)
assigned_progress = 1 - 0.5 = 0.5 (inverted for backward)
Already visited: Stop E, Stop D
Current/Next: Stop C  
Upcoming: Stop B, Stop A (traveling backward toward A)
```

### Key Technical Details

**Inversion Formula:** `1 - fraction`
- Simple but effective
- Preserves distance proportions
- Makes backward routes compatible with forward logic

**Applied to:**
- Vehicle progress calculation
- Stop fractions (covered in Group 6)
- Distance calculations (covered in Group 9)

---

**Status:** ✅ Complete
**Next Group:** Progress Calculation & Reference Point

---

## Group 5: Progress Calculation & Reference Point

### Overview
Calculates vehicle's current position on route as fraction (0.0 to 1.0) and uses it as reference point for determining stop statuses.

### Important Clarification

**Three Layers in Route System:**

1. **Roadnetwork Edges (Lowest Level)**
   - Individual road segments with geometry
   - Example: edge_id 12, 45, 78, 23
   - Connected via routeedge table with edge_order

2. **Route Geometry (Middle Level)**
   - Single LineString made by joining all roadnetwork edges
   - Goes from fraction 0.0 to 1.0
   - Example: 10km total route

3. **Stops (Highest Level - What Users See)**
   - Named locations where vehicle must stop
   - Each stop has GPS coordinates
   - ST_LineLocatePoint finds where stop sits on route geometry
   - Example: Stop A at 0.2, Stop B at 0.5, Stop C at 0.8
   - **In this documentation, A, B, C, D, E always represent actual stops, not edges!**

**Visual Hierarchy:**
```
Stops:           [Stop A]    [Stop B]    [Stop C]
                    ↓           ↓           ↓
Fractions:         0.2         0.5         0.8
                    ↓           ↓           ↓
Route Geometry: ════════════════════════════════ (LineString 0.0 → 1.0)
                    ↓           ↓           ↓
Roadnetwork:    [12][45][78][23][56][89][34]... (Many edges make up geometry)
```

### CTE: `effective_route`

**Purpose:** Determine vehicle's current position on route (used as reference point for stop calculations)

**Key Field:**
```sql
progress_start  -- Vehicle's current position (0.0 to 1.0)
                -- NOT a session table field!
                -- Calculated as: assigned_progress or new_progress (if deviated)
```

### Critical Correction: What is `progress_start`?

**NOT a session-level boundary!** It's the vehicle's **current position** calculated dynamically:
```sql
CASE
    WHEN status = 1 AND new_route_geom IS NOT NULL THEN new_progress  -- If deviated
    ELSE assigned_progress  -- Normal: vehicle's position on assigned route
END AS progress_start
```

**Why named "progress_start"?**
- It's the **starting point** for calculating distances to upcoming stops
- "Where vehicle currently is" = "start of remaining journey"

### Critical Architectural Point: Route vs Session Relationship

**Route = Reusable Infrastructure** (One route can have many sessions)

**Database Relationship:**
```
route (1) ──< (many) session ──< (many) trip
```

**Real-World Example:**

```
Route 4: School Bus Route (All stops: A → B → C → D → E)
Total length: 10km

Session 1 (Morning Shift):
- Direction: Forward  
- Time: 7:00 AM - 9:00 AM
- Vehicle: Bus #1
- Visits ALL stops A→E in forward order

Session 2 (Afternoon Shift):  
- Direction: Backward
- Time: 3:00 PM - 5:00 PM
- Vehicle: Bus #1
- Visits ALL stops E→A in reverse order

Session 3 (Evening Shift):
- Direction: Forward
- Time: 5:00 PM - 7:00 PM
- Vehicle: Bus #2 (different vehicle, same route)
- Visits ALL stops A→E
```

**Why This Design?**

✅ **Route Reusability:** Build route geometry once, use for multiple time slots/vehicles
✅ **Flexibility:** Different vehicles, different schedules, same infrastructure
✅ **Efficiency:** Don't duplicate stop/edge data

**Important:** 
- Sessions use the **entire route** with all stops
- No partial route capability (no session-level start/end boundaries)
- Differentiation is by: vehicle, time period, direction

**Example Route:**

**What A, B, C, D, E Represent:**
- These are **actual stops** from the `stop` table
- Each stop has GPS coordinates (lat/lon)
- The lines between them (----) represent the **route geometry** (made up of many roadnetwork edges)

```
Route 4:        [Stop A]----[Stop B]----[Stop C]----[Stop D]----[Stop E]
                   ↓            ↓           ↓            ↓           ↓
Stop Fractions:   0.0         0.25        0.5          0.75        1.0
                   ↑                                                 ↑
             Route Start                                        Route End

Underlying route geometry: Composed of ~50 roadnetwork edges
Total route length: 10 km

Forward Session: Visits all stops A→B→C→D→E
Backward Session: Visits all stops E→D→C→B→A
```

**Key Point:** Sessions use the **entire route** with all stops - no partial route capability!

### Assigned Progress Calculation

**Formula:**
```sql
assigned_progress = CASE 
    WHEN direction_factor = 1 THEN ST_LineLocatePoint(route_geom, current_location)
    ELSE 1 - ST_LineLocatePoint(route_geom, current_location)
END
```

**This value represents vehicle's position on full route geometry (0.0 to 1.0)**

### How progress_start is Used

**Purpose:** Reference point for stop calculations

Once vehicle position is known (progress_start = assigned_progress), the query uses it to:

1. **Identify next stops:** Stops where `stop_fraction > progress_start`
2. **Identify previous stops:** Stops where `stop_fraction < progress_start`  
3. **Calculate remaining distance:** From progress_start to each upcoming stop
4. **Determine stop status:** Based on vehicle's current position relative to each stop

### Real Example - Forward Route

**Setup:**
```sql
Route with stops:
[Stop A]-------[Stop B]-------[Stop C]-------[Stop D]
   ↓              ↓              ↓              ↓
  0.0            0.33           0.67           1.0

Session: Forward direction (all stops)
direction_factor: 1
```

**Vehicle at physical location 0.5 (between Stop B and Stop C):**
```sql
-- Step 1: PostGIS returns raw fraction
ST_LineLocatePoint(route_geom, position) = 0.5

-- Step 2: Forward direction (no inversion)
assigned_progress = 0.5

-- Step 3: This becomes progress_start
progress_start = 0.5

-- Step 4: Determine stop statuses
Stop A (0.0):  fraction < progress_start  → Status: Covered (already passed)
Stop B (0.33): fraction < progress_start  → Status: Previous (last stop passed)
Stop C (0.67): fraction > progress_start  → Status: Next (upcoming)
Stop D (1.0):  fraction > progress_start  → Status: Upcoming (after next)
```

### Real Example - Backward Route

**Setup:**
```sql
Same route, same stops, but backward session
direction_factor: -1
```

**Vehicle at same physical location 0.5:**
```sql
-- Step 1: PostGIS raw fraction (doesn't change)
ST_LineLocatePoint() = 0.5

-- Step 2: Invert for backward direction
assigned_progress = 1 - 0.5 = 0.5

-- Step 3: This becomes progress_start
progress_start = 0.5

-- Step 4: Determine stop statuses (relative to backward travel)
Stop D (inverted to high): Already covered
Stop C (inverted to medium): Already covered  
Stop B (inverted to low): Next stop (heading toward it)
Stop A (inverted to lowest): Upcoming

(Actual inversion done via adjusted_stop_order - covered in Group 6)
```

### Key Point

**progress_start is NOT a boundary** - it's simply:
- Where the vehicle currently is
- The starting point for measuring distances to remaining stops
- Used to determine which stops are ahead vs behind

No session-level boundaries exist to limit which portion of route is traveled!

---

**Status:** ✅ Complete  
**Next Group:** Stop Order & Fraction Adjustment

---

## Group 6: Stop Order & Fraction Adjustment

### Overview
Adjusts stop ordering and position fractions to handle backward routes, converting database values into direction-aware sequential integers.

### Database Context

**Stop Table Fields:**
```python
stop.stop_order: Decimal  # Can be: 1, 2, 3 OR 1.5, 2.7, 3.2
stop.geom: Point          # GPS coordinates (lat, lon)
```

**Why Decimal?** Allows inserting new stops between existing ones (e.g., add stop 2.5 between stops 2 and 3).

### CTE: `all_stops`

**Purpose:** Fetch all stops and adjust their order/fractions based on session direction

**Location:** [helper/query.py:517-540](../route-care/poc/helper/query.py#L517-L540)

```sql
SELECT
    er.*,  -- All fields from effective_route (vehicle position, route info)
    s.id AS stop_id,
    s.name AS stop_name,
    s.geom AS stop_location,
    s.stop_order as original_stop_order,
    
    -- Adjusted Stop Order: Re-rank to sequential integers based on direction
    CASE 
        WHEN er.effective_direction_factor = 1 THEN 
            ROW_NUMBER() OVER (PARTITION BY er.route_id ORDER BY s.stop_order ASC)
        ELSE 
            ROW_NUMBER() OVER (PARTITION BY er.route_id ORDER BY s.stop_order DESC)
    END AS adjusted_stop_order,
    
    -- Stop Fraction: Position on route geometry (0.0 to 1.0)
    CASE 
        WHEN er.effective_direction_factor = 1 THEN 
            ST_LineLocatePoint(er.route_geom, s.geom)
        ELSE 
            1 - ST_LineLocatePoint(er.route_geom, s.geom)
    END AS stop_fraction,
    
    -- Proximity check: Is vehicle within 20 meters of this stop?
    CASE WHEN ST_DWithin(s.geom::geography, er.current_location::geography, 20) 
         THEN 1 ELSE 0 END AS is_current_stop
FROM effective_route er
JOIN stop s ON er.route_id = s.route_id
```

### Adjusted Stop Order Logic

**Problem:** 
- Decimal stop_order values (1.5, 2.7) make direct comparisons difficult
- Backward routes need reversed order
- Need clean sequential integers (1, 2, 3...) for next/previous calculations

**Solution:** ROW_NUMBER() with direction-based sorting

**Forward Direction (direction_factor = 1):**
```sql
ROW_NUMBER() OVER (PARTITION BY route_id ORDER BY stop_order ASC)
```

**Backward Direction (direction_factor = -1):**
```sql
ROW_NUMBER() OVER (PARTITION BY route_id ORDER BY stop_order DESC)
```

### Real Example - Forward Route

**Database:**
```
stop table for route_id=4:
stop_id | name    | stop_order | geom (simplified)
--------|---------|------------|------------------
10      | Home    | 1.0        | (76.5, 9.1)
11      | School  | 2.5        | (76.51, 9.11)
12      | Market  | 3.7        | (76.52, 9.12)
13      | Park    | 5.0        | (76.53, 9.13)

Session: Forward
effective_direction_factor: 1
```

**SQL Processing:**
```sql
-- Step 1: Sort by stop_order ASC
ORDER BY: 1.0, 2.5, 3.7, 5.0

-- Step 2: Apply ROW_NUMBER()
ROW_NUMBER() results:

stop_id | original_stop_order | adjusted_stop_order | stop_fraction (calculated)
--------|---------------------|---------------------|---------------------------
10      | 1.0                 | 1                   | 0.0
11      | 2.5                 | 2                   | 0.33
12      | 3.7                 | 3                   | 0.67
13      | 5.0                 | 4                   | 1.0
```

**Result:** Clean sequential integers (1, 2, 3, 4) for forward travel.

### Real Example - Backward Route

**Same database stops, but backward session:**
```
Session: Backward
effective_direction_factor: -1
```

**SQL Processing:**
```sql
-- Step 1: Sort by stop_order DESC (reversed!)
ORDER BY: 5.0, 3.7, 2.5, 1.0

-- Step 2: Apply ROW_NUMBER()
ROW_NUMBER() results:

stop_id | original_stop_order | adjusted_stop_order | stop_fraction (inverted)
--------|---------------------|---------------------|-------------------------
13      | 5.0                 | 1                   | 1 - 1.0 = 0.0  (first stop now!)
12      | 3.7                 | 2                   | 1 - 0.67 = 0.33
11      | 2.5                 | 3                   | 1 - 0.33 = 0.67
10      | 1.0                 | 4                   | 1 - 0.0 = 1.0  (last stop now!)
```

**Result:** 
- Visit order reversed: Park(1) → Market(2) → School(3) → Home(4)
- Fractions inverted: Park is now at 0.0, Home is now at 1.0

### Stop Fraction Calculation

**Purpose:** Position of each stop on route geometry

**Forward:**
```sql
ST_LineLocatePoint(route_geom, stop_location)
```
Returns raw fraction (0.0 to 1.0) where stop sits on route.

**Backward:**
```sql
1 - ST_LineLocatePoint(route_geom, stop_location)
```
Inverts the fraction so stops are measured from opposite end.

### Visual Comparison

**Route Geometry (Physical Reality - Never Changes):**
```
Physical:  [Home]-------[School]-------[Market]-------[Park]
Geometry:   76.5 9.1     76.51 9.11     76.52 9.12     76.53 9.13
```

**Forward Session View:**
```
Stops:               [Home]------[School]------[Market]------[Park]
adjusted_stop_order:   1           2             3             4
stop_fraction:        0.0         0.33          0.67          1.0
Travel direction:     →           →             →             →
Visit order:          1st         2nd           3rd           4th
```

**Backward Session View:**
```
Stops:               [Park]------[Market]------[School]------[Home]
adjusted_stop_order:   1           2             3             4
stop_fraction:        0.0         0.33          0.67          1.0
Travel direction:     ←           ←             ←             ←
Visit order:          1st         2nd           3rd           4th
Physical location:    Park        Market        School        Home
```

### Why This Matters

**The Goal:** Use the same query logic for both forward and backward routes

**Without adjusted_stop_order (decimal stop_order):**
```sql
-- Forward route: Find next stop
SELECT MIN(stop_order) FROM stops 
WHERE stop_order > current_stop_order  -- Use MIN, compare with >

-- Backward route: Find next stop  
SELECT MAX(stop_order) FROM stops
WHERE stop_order < current_stop_order  -- Use MAX, compare with <

-- Problem: Two different query patterns (MIN vs MAX, > vs <)
```

**With adjusted_stop_order (sequential integers):**
```sql
-- Forward OR Backward: Find next stop
SELECT MIN(adjusted_stop_order) FROM stops
WHERE adjusted_stop_order > current_adjusted_stop_order

-- Benefit: Single query pattern works for both directions!
```

**Key Point:** Decimals aren't the problem - maintaining separate forward/backward query logic is. ROW_NUMBER() creates a unified view where "next stop" is always the same calculation regardless of direction.

### Additional Fields

**is_current_stop:**
```sql
CASE WHEN ST_DWithin(stop_geom::geography, vehicle_location::geography, 20) 
     THEN 1 ELSE 0 END
```
- Checks if vehicle is within 20 meters of this stop
- Used later to determine "current stop" status based on proximity

**current_fraction_raw:**
```sql
ST_LineLocatePoint(route_geom, current_location)
```
- Raw vehicle position (NOT adjusted for direction yet)
- Used for validation/debugging

### Key Technical Points

✅ **ROW_NUMBER() ensures sequential integers** regardless of decimal stop_order values
✅ **Direction handling is automatic** via ORDER BY ASC/DESC
✅ **Fractions and orders both adjusted** consistently for backward routes
✅ **Backward route creates "virtual" view** where end becomes start

---

**Status:** ✅ Complete
**Next Group:** Next/Previous Stop Detection

---

## Group 7: Next/Previous Stop Detection

### Overview
Identifies which stop is current (if within proximity), which was previously passed, and which is coming next based on vehicle position and stop sequence.

### Processing Pipeline

The detection happens across 4 CTEs in sequence:

1. **ranked_stops** - Identify current stop by proximity
2. **current_stop_orders** - Get sequence number of current stop
3. **previous_stop_orders** - Find most recent passed stop
4. **progress_with_order** - Calculate next upcoming stop

### CTE 1: `ranked_stops`

**Purpose:** Add current_stop_id using proximity detection

**Location:** [helper/query.py:543-549](../route-care/poc/helper/query.py#L543-L549)

```sql
SELECT *,
    ROW_NUMBER() OVER (PARTITION BY vehicle_id ORDER BY adjusted_stop_order) AS stop_rank,
    MAX(CASE WHEN is_current_stop = 1 THEN stop_id END) OVER (PARTITION BY vehicle_id) AS current_stop_id,
    COUNT(*) OVER (PARTITION BY vehicle_id) AS total_stops, 
    COALESCE(MIN(CASE WHEN progress_start = 0 and adjusted_stop_order = 1 THEN ST_Distance(current_location::geography, stop_location::geography) END) OVER (PARTITION BY vehicle_id), 0) AS initial_distance,
    COUNT(stop_id) FILTER(WHERE stop_fraction > progress_start) OVER (PARTITION BY vehicle_id ORDER BY adjusted_stop_order) as stops_count
FROM all_stops
```

**Key Fields:**

| Field | Calculation | Purpose |
|-------|-------------|---------|
| `current_stop_id` | `MAX(CASE WHEN is_current_stop = 1 THEN stop_id END)` | ID of stop vehicle is currently at (within 20m) |
| `stop_rank` | `ROW_NUMBER() ... ORDER BY adjusted_stop_order` | Sequential rank (1, 2, 3...) for each stop |
| `total_stops` | `COUNT(*)` | Total number of stops on route |
| `initial_distance` | Distance to first stop when progress_start = 0 | Used for ETA when vehicle hasn't entered route yet |
| `stops_count` | `COUNT(...) FILTER(WHERE stop_fraction > progress_start)` | Number of stops remaining ahead of vehicle |

**How current_stop_id Works:**

```sql
MAX(CASE WHEN is_current_stop = 1 THEN stop_id END) OVER (PARTITION BY vehicle_id)
```

- `is_current_stop = 1` when vehicle within 20m of stop
- `MAX()` window function propagates this to all rows for that vehicle
- Result: All rows get same `current_stop_id` value
- If no stop is within 20m, `current_stop_id` = NULL

### CTE 2: `current_stop_orders`

**Purpose:** Convert current_stop_id to its sequence number

**Location:** [helper/query.py:551-554](../route-care/poc/helper/query.py#L551-L554)

```sql
SELECT *,
    MAX(CASE WHEN stop_id = current_stop_id THEN adjusted_stop_order END) OVER (PARTITION BY vehicle_id) AS current_stop_order
FROM ranked_stops
```

**What it does:**
- Finds the row where `stop_id = current_stop_id`
- Extracts its `adjusted_stop_order` value
- Propagates this to all rows as `current_stop_order`

**Example:**
```
Vehicle at Stop B (stop_id = 11):

Row 1: stop_id=10, adjusted_stop_order=1, current_stop_id=11 → current_stop_order=2
Row 2: stop_id=11, adjusted_stop_order=2, current_stop_id=11 → current_stop_order=2 ✓ (matches here)
Row 3: stop_id=12, adjusted_stop_order=3, current_stop_id=11 → current_stop_order=2
Row 4: stop_id=13, adjusted_stop_order=4, current_stop_id=11 → current_stop_order=2
```

### CTE 3: `previous_stop_orders`

**Purpose:** Find the last stop that was passed (not current, behind vehicle)

**Location:** [helper/query.py:556-560](../route-care/poc/helper/query.py#L556-L560)

```sql
SELECT *,
    -- Find previous stop considering direction
    MAX(CASE WHEN stop_fraction < progress_start AND is_current_stop = 0 THEN adjusted_stop_order END) OVER (PARTITION BY vehicle_id) AS previous_stop_order
FROM current_stop_orders
```

**Logic:**
- `stop_fraction < progress_start` → Stop is behind vehicle
- `is_current_stop = 0` → Vehicle not currently at this stop
- `MAX(adjusted_stop_order)` → Highest sequence number meeting conditions = most recent passed stop

**Example:**
```
Route: [A]----[B]----[C]----[D]
Order:  1      2      3      4
Fraction: 0.0  0.33   0.67   1.0

Vehicle: progress_start = 0.5 (between B and C)

Stop A: stop_fraction=0.0 < 0.5, is_current_stop=0 → candidate (order=1)
Stop B: stop_fraction=0.33 < 0.5, is_current_stop=0 → candidate (order=2) ✓ MAX
Stop C: stop_fraction=0.67 > 0.5 → not a candidate
Stop D: stop_fraction=1.0 > 0.5 → not a candidate

Result: previous_stop_order = 2 (Stop B)
```

### CTE 4: `progress_with_order`

**Purpose:** Calculate which stop is next based on vehicle position and status

**Location:** [helper/query.py:563-575](../route-care/poc/helper/query.py#L563-L575)

```sql
SELECT *,
    -- Find next stop considering direction
    CASE WHEN 
        progress_start = 0 AND status != 2 THEN 
            MIN(CASE WHEN stop_fraction = progress_start THEN adjusted_stop_order END) OVER (PARTITION BY vehicle_id)
    WHEN 
        progress_start = 0 AND status = 2 THEN 
            MIN(CASE WHEN is_current_stop = 0 THEN adjusted_stop_order END) OVER (PARTITION BY vehicle_id)
    ELSE 
        MIN(CASE WHEN stop_fraction > progress_start AND (adjusted_stop_order > current_stop_order OR (current_stop_order is NULL and adjusted_stop_order > previous_stop_order))  THEN adjusted_stop_order END) OVER (PARTITION BY vehicle_id) 
    END AS next_stop_order
FROM previous_stop_orders
```

### Next Stop Detection Logic

**Three scenarios handled:**

#### Scenario 1: Vehicle Before Route Starts (Not On Route)
```sql
progress_start = 0 AND status != 2  
→ MIN(CASE WHEN stop_fraction = progress_start THEN adjusted_stop_order END)
```

**Condition:** Vehicle hasn't entered route yet, not "On Route" status
**Logic:** Find stop at position 0.0 (first stop)
**Result:** First stop is next

**Example:**
```
Vehicle: progress_start = 0, status = 0 (No Location)
Stops: A(0.0), B(0.33), C(0.67), D(1.0)

Stop A: stop_fraction = 0.0 = progress_start → adjusted_stop_order = 1 ✓
Stop B: stop_fraction = 0.33 ≠ 0.0 → skip
Stop C: stop_fraction = 0.67 ≠ 0.0 → skip
Stop D: stop_fraction = 1.0 ≠ 0.0 → skip

next_stop_order = MIN(1) = 1 (Stop A)
```

#### Scenario 2: Vehicle Before Route Starts (On Route)
```sql
progress_start = 0 AND status = 2
→ MIN(CASE WHEN is_current_stop = 0 THEN adjusted_stop_order END)
```

**Condition:** Vehicle at route start but "On Route" status
**Logic:** Find first stop that is NOT current (in case already at first stop)
**Result:** If at Stop A, next is Stop B; otherwise Stop A

**Example:**
```
Vehicle: progress_start = 0, status = 2 (On Route), already at Stop A

Stop A: is_current_stop = 1 → skip
Stop B: is_current_stop = 0 → adjusted_stop_order = 2 ✓
Stop C: is_current_stop = 0 → adjusted_stop_order = 3
Stop D: is_current_stop = 0 → adjusted_stop_order = 4

next_stop_order = MIN(2, 3, 4) = 2 (Stop B)
```

#### Scenario 3: Vehicle On Route (Normal Travel)
```sql
ELSE → MIN(CASE WHEN stop_fraction > progress_start 
             AND (adjusted_stop_order > current_stop_order 
                  OR (current_stop_order is NULL and adjusted_stop_order > previous_stop_order))
             THEN adjusted_stop_order END)
```

**Condition:** Vehicle traveling on route
**Logic:** Find nearest stop ahead that is:
- Ahead of vehicle position (`stop_fraction > progress_start`)
- AND either:
  - After current stop (`adjusted_stop_order > current_stop_order`), OR
  - After previous stop if no current stop (`adjusted_stop_order > previous_stop_order`)

**Example 1 - At Stop B:**
```
Vehicle: progress_start = 0.33, current_stop_order = 2

Stop A (order=1): fraction=0.0 < 0.33 → skip (behind)
Stop B (order=2): fraction=0.33 = 0.33 → skip (not >)
Stop C (order=3): fraction=0.67 > 0.33 AND 3 > 2 → candidate ✓
Stop D (order=4): fraction=1.0 > 0.33 AND 4 > 2 → candidate

next_stop_order = MIN(3, 4) = 3 (Stop C)
```

**Example 2 - Between B and C:**
```
Vehicle: progress_start = 0.5, current_stop_order = NULL, previous_stop_order = 2

Stop A (order=1): fraction=0.0 < 0.5 → skip (behind)
Stop B (order=2): fraction=0.33 < 0.5 → skip (behind)
Stop C (order=3): fraction=0.67 > 0.5 AND (current=NULL AND 3 > 2) → candidate ✓
Stop D (order=4): fraction=1.0 > 0.5 AND (current=NULL AND 4 > 2) → candidate

next_stop_order = MIN(3, 4) = 3 (Stop C)
```

### Visual Flow Example

**Setup:**
```
Route: [Home]----[School]----[Market]----[Park]
Order:    1          2           3          4
Fraction: 0.0       0.33        0.67       1.0
```

**Vehicle Journey:**

| Vehicle Position | progress_start | current_stop_id | current_stop_order | previous_stop_order | next_stop_order | Meaning |
|------------------|----------------|-----------------|--------------------|--------------------|-----------------|---------|
| Before route | 0.0 | NULL | NULL | NULL | 1 | Next: Home |
| At Home (20m proximity) | 0.0 | 10 | 1 | NULL | 2 | Current: Home, Next: School |
| Between Home & School | 0.15 | NULL | NULL | 1 | 2 | Previous: Home, Next: School |
| At School | 0.33 | 11 | 2 | 1 | 3 | Current: School, Next: Market |
| Between School & Market | 0.5 | NULL | NULL | 2 | 3 | Previous: School, Next: Market |
| At Market | 0.67 | 12 | 3 | 2 | 4 | Current: Market, Next: Park |
| Between Market & Park | 0.85 | NULL | NULL | 3 | 4 | Previous: Market, Next: Park |
| At Park | 1.0 | 13 | 4 | 3 | NULL | Current: Park, No next |

### Key Technical Details

**Window Functions:** All calculations use window functions to propagate values across all stop rows for each vehicle
**Direction Awareness:** Works with adjusted_stop_order (already direction-corrected in Group 6)
**Sequence Validation:** `adjusted_stop_order > current_stop_order` prevents marking earlier stops as "next"
**Null Handling:** `current_stop_order IS NULL` handled by checking `previous_stop_order` instead

### Common Edge Cases

**Case 1: Vehicle at last stop**
- `current_stop_order = 4` (Park)
- No stops have `adjusted_stop_order > 4`
- `next_stop_order = NULL`

**Case 2: Vehicle near stop but past it**
- Vehicle at fraction 0.35 (just past School at 0.33)
- Not within 20m, so `current_stop_id = NULL`
- `previous_stop_order = 2` (School)
- `next_stop_order = 3` (Market)

**Case 3: Backward route**
- Adjusted order already reversed: Park(1), Market(2), School(3), Home(4)
- Logic works identically with adjusted values
- Physical Park is sequence 1, Home is sequence 4

---

**Status:** ✅ Complete
**Next Group:** Stop Status Assignment


