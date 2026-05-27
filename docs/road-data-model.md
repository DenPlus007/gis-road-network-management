# 🗺️ Road Network Data Model

Reference guide for the road network GIS data model — feature classes,
attribute standards and editing rules.

---

## Feature Dataset Structure

All feature classes must be inside the same Feature Dataset in a File GDB,
sharing a single projected coordinate system (UTM recommended).

```
RoadNetwork.gdb
└── RoadNetworkFD  (Feature Dataset — shared CRS)
    ├── FC_Cam_Caminos          (Polyline — road segments)
    ├── FC_Cam_Bifurcaciones    (Point — junction nodes)
    ├── FC_Cam_Mojones          (Point — kilometer markers)
    ├── FC_Cam_Obras            (Point — road infrastructure)
    ├── FC_Cam_Bloqueos         (Point — barriers and blockages)
    ├── FC_Cam_Giros            (Turn — turn restrictions)
    └── RoadNetwork_Topology    (Topology)
```

---

## FC_Cam_Caminos — Road Segments

| Field | Type | Description | Domain / Rule |
|---|---|---|---|
| `Road_ID` | Integer | Unique road identifier | Consecutive; no nulls |
| `Name` | String(100) | Road name | Bifurcación 1 / Bifurcación 2 / named road |
| `Hierarchy` | String(20) | Road classification | Main / Secondary / Tertiary |
| `Surface` | String(20) | Surface type | Asphalt / Gravel / Earth |
| `Endpoint_1` | Integer | Origin bifurcation ID | Must match FC_Cam_Bifurcaciones |
| `Endpoint_2` | Integer | Destination bifurcation ID | Must match FC_Cam_Bifurcaciones |
| `Enabled` | String(3) | Operational status | YES / NO |
| `Width_m` | Float | Width in meters | Default 8m for secondary |
| `Direction` | String(2) | Traffic direction | 00=both / 01=E1→E2 / 10=E2→E1 |
| `Security` | String(3) | Security access required | YES / NO |
| `Slope` | Float | Road slope | Default 0 |
| `Jurisdiction` | String(20) | Administrative level | National/Provincial/Local/Internal |
| `SAP_Code` | String(20) | SAP asset code | Nullable for non-SAP roads |
| `Internal_Control` | String(3) | Internal QC flag | YES / NO |
| `Comments` | String(250) | Free text notes | Optional |

### Editing Rules

**Hierarchy:**
- Roads to wells and installations → `Secondary`
- Roads to plants and batteries → `Main`
- Municipal/provincial connections → `Local` or `Provincial`
- Third-party roads → separate layer (no SAP code)

**Endpoint assignment:**
- Every road segment must have Endpoint_1 and Endpoint_2 pointing to
  existing bifurcation IDs
- Roads to well locations: if no installation point exists, use a
  `Estacas` layer point as the endpoint reference

**Width defaults:**
- When width is available in CAO or field form: use that value
- Otherwise: use the standard for the road type
- Roads to wells: inherit width from the access road connecting to the location

**SAP merge rule:**
- Adjacent road segments sharing the same SAP code → apply Merge tool
  to consolidate into a single feature

---

## FC_Cam_Bifurcaciones — Junction Nodes

| Field | Type | Description |
|---|---|---|
| `Bifurcation_ID` | Integer | Unique junction identifier |
| `Road_ID` | Integer | Associated road segment ID |
| `Enabled` | String(3) | Operational status — YES / NO |
| `Description` | String(250) | Optional notes |
| `Observations` | String(250) | QC or field observations |

**Placement rules:**
- One bifurcation at every road intersection, junction or dead end
- Roads entering a plant: place bifurcation at plant entry, then
  connect roads to all major installations from that bifurcation
- If a road splits at a T-junction: one bifurcation at the split point,
  two outgoing road segments

---

## FC_Cam_Mojones — Kilometer Markers

| Field | Type | Description |
|---|---|---|
| `Mojon_ID` | Integer | Unique marker ID |
| `Road_ID` | Integer | Road this marker belongs to |
| `KM` | Float | Kilometer value |
| `Description` | String(100) | Notes |

**Rules:**
- Consistent spacing along the road segment
- Duplicate IDs must be detected and resolved (see QC workflow)

---

## FC_Cam_Obras — Road Infrastructure

| Field | Type | Description |
|---|---|---|
| `Obra_ID` | Integer | Unique infrastructure ID |
| `Road_ID` | Integer | Road segment this work is on |
| `Type` | String(50) | Bridge / Culvert / Drain / Other |
| `Condition` | String(20) | Good / Fair / Poor |
| `Description` | String(250) | Notes |

---

## FC_Cam_Bloqueos — Barriers and Blockages

| Field | Type | Description |
|---|---|---|
| `Blockage_ID` | Integer | Unique barrier ID |
| `Road_ID` | Integer | Road segment affected |
| `Type` | String(30) | Gate / Cattle guard / Closure |
| `Enabled` | String(3) | Passable status — YES / NO |
| `Confirmed_Date` | Date | Last status confirmation date |
| `Observations` | String(250) | Notes |

> `Enabled = NO` → barrier blocks routing → generates detour in Network Dataset.
> Always confirm status with field/security team before updating.

---

## FC_Cam_Giros — Turn Restrictions

Turn features control allowed and prohibited movements at intersections
within the Network Dataset.

| Field | Type | Description |
|---|---|---|
| `Turn_ID` | Integer | Unique turn ID |
| `Angle` | Float | Turn angle in degrees |
| `Direction` | String(20) | Left / Right / U-turn / Straight |
| `Restriction` | String(20) | Allowed / Prohibited |

---

## Third-Party Roads

Roads belonging to municipalities, provinces or other operators must be:
- Stored in a **separate feature class** (e.g. `FC_Cam_Terceros`)
- **Not assigned a SAP code**
- Clearly identified with `Jurisdiction = External`

These roads are **not included** in the internal road network dataset.

---

## Network Dataset Configuration

| Parameter | Value |
|---|---|
| Connectivity policy | End point (roads connect at endpoints) |
| Elevation model | None (flat network) |
| Edge weight | `Length_m` (Euclidean length in meters) |
| Turn restrictions | FC_Cam_Giros |
| Barriers | FC_Cam_Bloqueos (Enabled = NO) |
| Direction field | `Direction` on road segments |
