
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
- A truck schedule as a timeline (like a Gantt chart): one row per
  truckload showing travel out, unloading, the trip back, and washout,
  in different colors, on a local clock-time axis. Use the segment timings
  from Calculations; spacing between truckload arrivals is U_full minutes.
  The timeline continues into the next day when needed. Hovering shows each
  segment's start and end times.
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

## Out of scope
- Reinforcement, finishing crews, pump trucks, weather, and costs.
- Saving plans.
