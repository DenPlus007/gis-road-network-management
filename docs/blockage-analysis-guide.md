# 🚧 Blockage Analysis Guide — Road Network

Methodology for detecting, validating and resolving road access barriers
that affect routing in the road network.

---

## Overview

Road blockages and barriers (gates, cattle guards) can disable road segments
or junctions in the Network Dataset, causing unexpected routing detours.
This guide covers the full process from detection to GIS update and
routing re-verification.

---

## Types of Barriers in the Network

| Feature | Description | GIS layer |
|---|---|---|
| **Gates** (Tranqueras) | Controlled access points on roads | `FC_Cam_Bloqueos` |
| **Cattle guards** (Guardaganados) | Ground-level barriers for livestock control | `FC_Cam_Bloqueos` |
| **Blockages** (Bloqueos) | Temporary or permanent road closures | `FC_Cam_Bloqueos` |

**Key attribute:**

```
Enabled = YES  →  barrier is passable  →  no routing impact
Enabled = NO   →  barrier blocks access  →  routing detour generated
```

---

## Stage 1 — Initial Detection

### Step 1.1 — Check Geometry

Run `Check Geometry` on the roads layer before blockage analysis to ensure
geometry is clean:

```
ArcGIS Pro → Geoprocessing → Check Geometry
Input: Roads feature class (exported to shapefile or local GDB)
Method: OGC
Output: geometry_errors table
```

Fix all geometry errors before proceeding.

### Step 1.2 — Count Overlapping Features

```
ArcGIS Pro → Geoprocessing → Count Overlapping Features
Input: Roads feature class
Output: overlapping_roads layer
```

Filter result: `COUNT_ > 1` → overlapping segments → fix before proceeding.

---

## Stage 2 — Barrier Identification

### Step 2.1 — Load barrier layers

Load into the same map:
- Roads layer (`FC_Cam_Caminos`)
- Blockages layer (`FC_Cam_Bloqueos`)
- Any additional barrier layers (gates, cattle guards)

### Step 2.2 — Spatial Join: barriers × roads

```
ArcGIS Pro → Geoprocessing → Spatial Join
Target: Barriers layer
Join: Roads layer
Join operation: Join one to one
Match option: Within a distance (recommended: 10–20 m)
Output: barriers_with_road_info
```

Result: each barrier record includes the road ID it affects.

### Step 2.3 — Identify disabled barriers

Filter the spatial join result:

```sql
Enabled = 'NO'
```

These are the barriers causing routing detours.

---

## Stage 3 — Validation with Security Team

Before updating GIS, validate the status of disabled barriers:

- Contact security / field team to confirm actual barrier status
- Confirm whether `Enabled = NO` is current or outdated
- Document confirmation: date, contact, new status

**Decision logic:**

```
Field confirmation: barrier IS closed  →  keep Enabled = NO  →  update route planning
Field confirmation: barrier IS open    →  update to Enabled = YES  →  re-run routing
No confirmation available             →  flag for follow-up, keep current status
```

---

## Stage 4 — GIS Update

After confirmation:

1. Update `Enabled` field on the barrier feature
2. Update adjacent road segment `Enabled` field if road is fully blocked
3. Update bifurcation `Enabled` field if junction is blocked
4. Save edits
5. Rebuild Network Dataset (if already built)

---

## Stage 5 — Routing Verification

After GIS update, verify routing resolves correctly:

```
ArcGIS Pro → Analysis → Network Analysis → Route
```

1. Create start and end stop points
2. Run route
3. Inspect generated path — verify it no longer uses the detour
4. If detour persists: re-check bifurcation `Enabled` + endpoint IDs
5. After verification: remove routing layers (Discard) to keep project clean

---

## Stage 6 — Duplicate Geometry Check on Barriers

Duplicate barrier geometries cause incorrect counts in the spatial join.
Check before reporting:

```python
# See scripts/blockage_analysis.py for automated detection
```

Manual check in ArcGIS Pro:
```
Geoprocessing → Find Identical
Input: Barriers layer
Fields: Shape (geometry)
Output: identical_barriers table
```

Delete duplicates keeping one representative feature per location.

---

## Common Issues

| Issue | Cause | Solution |
|---|---|---|
| Unexpected detour after barrier = YES update | Road segment still has Enabled = NO | Check road segment attribute directly |
| Barrier not detected in spatial join | Barrier too far from road | Increase spatial join tolerance |
| Route error: Network Dataset not found | Network Dataset path broken | Rebuild or re-link Network Dataset |
| Error 030033 on Network rebuild | Geometry inconsistency in roads | Re-run Check Geometry + fix before rebuild |

---

## Output Report Fields

The blockage analysis report (`blockage_report.xlsx`) contains:

| Field | Description |
|---|---|
| `Barrier_ID` | Unique barrier identifier |
| `Barrier_Type` | Gate / Cattle guard / Blockage |
| `Road_ID` | Road segment affected |
| `Enabled` | Current status in GIS |
| `Confirmed_Date` | Date of field/security confirmation |
| `Confirmed_By` | Team or person who confirmed status |
| `Action_Required` | YES / NO |
| `Observations` | Notes |

---

## Script

See [`scripts/blockage_analysis.py`](../scripts/blockage_analysis.py) for
automated detection of disabled barriers and generation of the blockage report.
