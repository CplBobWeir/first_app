
# Concrete Pour Planner

## Purpose
A planning tool for a concrete pour. The user outlines a slab on
satellite imagery (or types its dimensions), and the app computes how
much concrete to order, how many truckloads it takes, how many trucks the
batch plant needs to keep the pour going without a break, and when each
truck should arrive.

## Users
A construction management student, superintendent, or project engineer
planning a slab pour and calling in the order to the batch plant.

## Platform
- A single index.html with plain HTML, CSS, and JavaScript.
- Map: Leaflet 1.9.4 from unpkg, with Esri World Imagery tiles
  (https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x})
  and the required attribution. No API keys.

## Inputs
- Slab outline, either:
  - Select map drawing or typed dimensions with a two-option radio group. The
    map opens centered on BYU campus (40.2518, -111.6493) at zoom 18.
  - Drawn on the map: use a "Draw slab" button, click at least three distinct
    corners, then double-click (or click the first point) to finish. Corners
    can be dragged afterward. The outline uses map coordinates (latitude
    -85.05 to 85.05 degrees, longitude -180 to 180 degrees) and must form a
    valid polygon. The example slab shown on load is the default.
  - Typed: enter length and width in feet in number fields (0.1 ft steps,
    range 1-1,000 ft for each dimension). No dimension defaults; both fields
    are required in typed mode. The app draws a rectangle of those dimensions
    at the map center.
- Slab thickness: number field in inches, range 1-36 in, 0.25 in steps,
  default 6.
- Waste allowance: number field in percent, range 0-25%, 1% steps, default 5.
- Concrete price: number field in USD per cubic yard, default $150.00/cy,
  minimum $0.00, $0.01 steps, and no maximum. The price is required and must
  be nonnegative and valid to the nearest cent.
- Truck capacity: number field in cubic yards, range 1-20 cy, 0.5 cy steps,
  default 10.
- One-way travel time from the batch plant: number field in minutes, range
  0-240 min, 1 min steps, default 20.
- Placement rate the crew can sustain: number field in cubic yards per hour,
  range 1-120 cy/hr, 1 cy/hr steps, default 30.
- Washout and reload time at the plant: number field in minutes, range
  1-120 min, 1 min steps, default 10.
- Time the first truck should arrive: local-time field with minute precision,
  range 00:00-23:59, default 7:00 AM.
- Download Truck Schedule: no new calculation inputs are required. The export
  uses the current slab and schedule settings and is unavailable when they do
  not produce a valid schedule.

## Calculations
- Slab area (ft^2): for a typed rectangle, area = length x width. For a drawn
  outline, use
  Let (lat_i, lon_i) denote polygon vertex coordinates in degrees, ordered
  around the outline; convert them to radians for the formula. Each
  edge uses consecutive vertices (lat1, lon1) and (lat2, lon2), including
  the closing edge from the last vertex to the first.
  A = R^2 x |sum over edges of (lon2 - lon1) x (2 + sin lat1 + sin lat2)| / 2
  with angles in radians and R = 6,378,137 m, converted to ft^2
  (1 m^2 = 10.7639 ft^2).
- Volume to order (cubic yards) =
  area x thickness / 12 / 27 x (1 + waste / 100), rounded up to the next
  0.5 cy.
- Estimated concrete cost (USD) = the rounded volume to order (cy) multiplied
  by the concrete price ($/cy), formatted to two decimal places. This is a
  basic concrete material estimate; it does not include taxes or other fees.
- Truckloads = volume to order / capacity, rounded up. Let N be the number of
  truckloads and q_k the volume in truckload k: q_k = capacity for k < N,
  and q_N = volume to order - capacity x (N - 1). Thus the last load is full
  when the ordered volume is an exact multiple of capacity.
- Unload time for a full truck = capacity / placement rate (hours). For
  timing calculations, convert this to minutes: U_full = 60 x capacity /
  placement rate. For truckload k, U_k = 60 x q_k / placement rate minutes;
  this gives the shorter unload time for a partial final load.
- Trucks arrive one full-truck unload time apart, so the crew never waits.
  Start is the first truckload's arrival time. Truckload k (k = 1, 2, ...)
  arrives at start + (k - 1) x U_full, with time additions in minutes.
- For a truckload arriving at A_k, one-way travel time T, unload time U_k,
  and washout/reload time W (all in minutes), its schedule segments are:
  travel out from A_k - T to A_k; unloading from A_k to A_k + U_k; return
  from A_k + U_k to A_k + U_k + T; and washout/reload from A_k + U_k + T
  to A_k + U_k + T + W.
- The schedule CSV introduces no new planning calculations. For load k, use
  the existing arrival A_k, unload duration U_k, travel time T, and washout
  time W. The assigned truck is ((k - 1) mod M) + 1, where M is the number of
  trucks in rotation. Unloading starts at A_k and ends at A_k + U_k; the
  return trip starts at unload end and ends at unload end + T; washout/reload
  completion is return end + W. Format times as local clock time and append
  (+1 day) to next-day times when a schedule crosses midnight.
- Round trip for a full truck = 2 x T + U_full + W, in minutes. Trucks needed
  in rotation = round trip / U_full, rounded up (but no more than N loads).
  Loads are assigned to the trucks in turn: truck 1, truck 2, and so on,
  then truck 1 again.
- Pour duration = (N - 1) x U_full + U_N minutes, from the first truckload's
  arrival until the last truckload is empty. Finish time = start + pour
  duration.

## Outputs
- The slab drawn on the map with its area in ft^2 and its waste-adjusted,
  rounded volume to order in cubic yards on a label.
- Summary cards: concrete to order (cy), truckloads, trucks in rotation,
  pour duration (hours and minutes), and finish time (local clock time).
  Times past midnight continue into the next calendar day and are indicated
  as such.
- A summary card for estimated concrete cost in USD, calculated from the
  rounded volume to order and price per cubic yard. The card updates
  immediately when inputs change. If the price is invalid, clear only the
  cost estimate while keeping valid pour and schedule results visible.
- A truck schedule as a timeline (like a Gantt chart): one row per
  truckload showing travel out, unloading, the trip back, and washout,
  in different colors, on a local clock-time axis. Use the segment timings
  from Calculations; spacing between truckload arrivals is U_full minutes.
  The timeline continues into the next day when needed. Hovering shows each
  segment's start and end times.
- A "Download Truck Schedule" button by the timeline that downloads the
  current schedule as an Excel-friendly CSV file named `truck-schedule.csv`.
  Include a header row with these columns: load number, assigned truck,
  arrival time, unloading start, unloading end, return start, return end, and
  washout/reload completion time. Use local clock times and append `(+1 day)`
  to next-day times. Disable the button when the current slab or schedule
  inputs are invalid so an invalid or stale schedule cannot be downloaded.
  An invalid concrete price alone does not disable the download when the
  schedule remains valid.
- An "Order summary" box with a plain-text order for the batch plant
  (volume in cubic yards, number of loads, first arrival as local time, and
  spacing between truckload arrivals in minutes) and a "Copy" button.
- Everything updates immediately whenever an input changes.

## Look and feel
- Opens with a 40 ft x 60 ft rectangular example slab already drawn at the
  center of the map, using the existing default input values, so it is useful
  the moment it opens.
- Clean and professional, like a commercial construction app: white
  cards with soft shadows, one accent color, a modern system font, large
  readable numbers on the summary cards.
- Map and inputs on the left; summary, timeline, and order on the right.
  On mobile screens, sections stack in this order: map, inputs, summary
  cards, truck timeline, and order summary.
- A short hint tells the user what to do next.

## Validation
1. Typed slab 40 ft x 60 ft x 6 in, 5% waste, 10 cy trucks, 20 min
   travel, 30 cy/hr, 10 min washout, first truck 7:00 AM:
   - Area 2,400 ft^2; 44.44 cy; with waste 46.67 cy; order 47.0 cy.
   - 5 truckloads (four of 10 cy and one of 7 cy).
   - Unload time 20 min, so trucks arrive at 7:00, 7:20, 7:40, 8:00,
     and 8:20.
   - Round trip 2 x 20 + 20 + 10 = 70 min, so 4 trucks in rotation.
   - Last truck holds 7 cy and unloads in 14 min, so the pour finishes
     at 8:34 AM. Pour duration 1 h 34 min.
2. Area: an outline with corners (40.2500, -111.6500),
   (40.2510, -111.6500), (40.2510, -111.6490), (40.2500, -111.6490) has
   an area of about 101,800 ft^2 (within 1%).
3. Typed slab 20 ft x 20 ft x 4 in, 0% waste, 10 cy trucks: 400 ft^2,
   4.94 cy, order 5.0 cy, 1 truckload, 1 truck, pour duration 10 min.
4. Same inputs as case 1, except the first truck arrives at 11:00 PM:
   arrivals are 11:00 PM, 11:20 PM, 11:40 PM, 12:00 AM, and 12:20 AM the
   next day; the pour finishes at 12:34 AM the next day.
5. Same slab and delivery inputs as case 1 with the default price of
   $150.00/cy: rounded order volume 47.0 cy and estimated concrete cost
   $7,050.00.
6. Same slab and delivery inputs as case 1 with a price of $0.00/cy:
   estimated concrete cost is $0.00.
7. Same slab and delivery inputs as case 1 with a negative price or a price
   not in $0.01 increments: the price is invalid and the cost estimate is
   cleared, while valid pour and schedule results remain visible.
8. Export the schedule for case 1: the CSV has a header row and five load
   rows; arrivals are 7:00 AM, 7:20 AM, 7:40 AM, 8:00 AM, and 8:20 AM, with
   assigned trucks 1, 2, 3, 4, and 1.
9. Check the fifth CSV row for case 1, the 7-cy partial final load: arrival
   and unloading start 8:20 AM, unloading end and return start 8:34 AM,
   return end 8:54 AM, and washout/reload completion 9:04 AM.
10. Export the schedule for case 4, starting at 11:00 PM: all times after
    midnight include `(+1 day)`. The fifth row has arrival and unloading
    start at 12:20 AM (+1 day), unloading end and return start at 12:34 AM
    (+1 day), return end at 12:54 AM (+1 day), and washout/reload completion
    at 1:04 AM (+1 day).
11. With an invalid slab or schedule input, the schedule is unavailable and
    the download button is disabled; no stale schedule can be exported. An
    invalid concrete price alone leaves a valid schedule export available.

## Out of scope
- Taxes, delivery charges, and other fees. The basic concrete material cost
  estimate is in scope.
- Reinforcement, finishing crews, pump trucks, and weather.
- Native `.xlsx` workbooks, user-selected dates or time zones, export filters,
  and additional reporting or calculations.
- Saving plans.

## Build stages
1. **Static layout mockup (no real calculations):** Arrange the map area and
   inputs on the left, with summary cards, timeline, and order summary on the
   right. Use placeholders and apply the specified visual style.
   **Test:** Confirm the desktop layout is clear, all planned sections are
   visible, and mobile sections stack in the specified order.
2. **Map and slab input:** Add Leaflet and attributed imagery tiles, show the
   40 ft x 60 ft example slab, and support drawing/editing an outline and
   entering typed dimensions.
   **Test:** Confirm the map opens at the specified BYU location and zoom;
   drawing, dragging corners, and typed rectangles work; and latitude limits
   are respected. Outputs may remain placeholders.
3. **Core calculations and summary cards:** Implement area for drawn and
   typed slabs, volume rounding, truckloads, unload intervals, required truck
   rotation, pour duration, and finish time.
   **Test:** Match validation cases 1-3, including the 7 cy final load and
   14-minute unload in case 1.
4. **Truck timeline and order summary:** Show each load's travel, unloading,
   return, and washout/reload segments; add the order text and copy action.
   **Test:** Confirm segment times and arrival spacing match case 1; case 4
   crosses midnight and finishes at 12:34 AM the next day.
5. **Final validation and polish:** Verify input limits/defaults, immediate
   updates, map attribution, and the complete responsive layout.
   **Test:** Re-run all validation cases, confirm the required mobile order,
   and check that invalid or out-of-range inputs cannot produce misleading
   results.
6. **Concrete cost estimate:** Add the USD-per-cubic-yard price input and
   estimated-cost summary card using the rounded order quantity.
   **Test:** Confirm validation cases 5-7: $7,050.00 at $150.00/cy, $0.00 at
   zero price, and invalid prices clearing only the cost estimate.
7. **Download truck schedule:** Add a CSV download for the existing schedule
   with per-load truck assignment and time segments; disable it when the
   current schedule is invalid.
   **Test:** Confirm validation cases 8-11, including the 7-cy final load,
   overnight `(+1 day)` values, and disabled export for invalid schedules.
