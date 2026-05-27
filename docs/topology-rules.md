# 🔷 Topology Rules — Road Network

Reference guide for the topology rules applied to validate the road network
feature classes in ArcGIS Pro.

---

## What is topology and why does it matter?

Topology ensures the **spatial and geometric integrity** of road features —
that lines connect correctly, don't overlap where they shouldn't, and that
junctions exist where segments meet. It is a prerequisite for a valid Network
Dataset and for reliable routing analysis.

> Topology validates *geometry relationships*. It does NOT require a Network Dataset.
> Build topology first, then build the Network Dataset on the clean data.

---

## Setup

**Requirements:**
- All feature classes in the same Feature Dataset inside a GDB
- Shared coordinate system across all feature classes
- Feature classes: Roads (`FC_Cam_Caminos`) + Bifurcations (`FC_Cam_Bifurcaciones`)

**Steps to create topology in ArcGIS Pro:**
1. Right-click the Feature Dataset → New Topology
2. Select feature classes: Roads + Bifurcations
3. Add topology rules (see table below)
4. Set cluster tolerance (recommended: 0.001–0.005 m)
5. Finish → Validate topology
6. Load topology layer to map
7. Review errors + correct manually

---

## Topology Rules Applied

### Rule 1 — Must Not Overlap (Roads)

**What it checks:** No road segment should occupy the same space as another
road segment in the same feature class.

**Error type:** Line error where segments overlap.

**Common cause:** Duplicate digitizing, copy-paste errors, import issues.

**Correction:**
- Select overlapping segment
- Delete duplicate or merge with the base segment

---

### Rule 2 — Must Not Intersect (Roads)

**What it checks:** Road lines should not cross or overlap each other unless
they share an endpoint (at a bifurcation).

**Error types:**
- Line error where segments overlap
- Point error where lines cross without a shared junction

**Common cause:** Two roads crossing without a bifurcation at the intersection.

**Correction:**
- Add a bifurcation point at the crossing
- Split segments at the intersection point
- Verify endpoint IDs match the new bifurcation

---

### Rule 3 — Must Not Intersect With (Roads × Roads)

**What it checks:** Same as Rule 2 but applied as a cross-check between
two road subtypes or layers (e.g. main roads vs secondary roads).

**Correction:** Same as Rule 2.

---

### Rule 4 — Must Not Have Dangles (Roads)

**What it checks:** The endpoint of each road segment must touch another
road or itself. Dangling endpoints indicate disconnected segments.

**Error type:** Point error at unconnected line endpoints.

**Common cause:** Road was digitized without snapping to the adjacent segment
or bifurcation. Also occurs at dead-end roads (cul-de-sacs) — these should
be marked as exceptions in the topology.

**Correction:**
- Extend endpoint to touch the adjacent road or bifurcation
- Mark legitimate dead-ends as topology exceptions

---

### Rule 5 — Must Not Have Pseudo Nodes (Roads)

**What it checks:** A node that connects exactly two road segments with
no change in road attributes. Pseudo nodes indicate unnecessary splits
in what should be a single continuous road segment.

**Error type:** Point error at the pseudo node location.

**Common cause:** Road was accidentally split mid-segment; copy operations
that introduced extra vertices.

**Correction:**
- Merge the two road segments (Editor > Merge) if they belong to the same road
- Keep the pseudo node only if the two segments have different attribute values
  (e.g. different surface type or jurisdiction)

---

### Rule 6 — Must Be Single Part (Roads)

**What it checks:** Each road feature must be a single polyline, not a
multipart geometry.

**Error type:** Feature-level error on multipart geometries.

**Common cause:** Explode or multipart-to-singlepart operations not applied
after data import.

**Correction:**
- Use Multipart to Singlepart geoprocessing tool
- Review resulting features for attribute consistency

---

### Rule 7 — Must Not Self Overlap (Roads)

**What it checks:** A road segment must not cross or overlap itself.

**Error type:** Line error where the segment self-intersects.

**Common cause:** Editing error — vertex dragged across the segment line.

**Correction:**
- Edit vertices and remove the self-intersection
- Re-digitize the segment if too complex to fix manually

---

### Rule 8 — Endpoint Must Be Covered By (Roads → Bifurcations)

**What it checks:** Every road endpoint must coincide with a bifurcation point.
This ensures full topological connectivity between the road and junction layers.

**Error type:** Point error at road endpoints not covered by a bifurcation.

**Common cause:** Road was digitized but no bifurcation was placed at the endpoint;
or bifurcation was moved after road digitizing.

**Correction:**
- Place a bifurcation at the uncovered endpoint
- OR extend the road to reach an existing bifurcation

---

## Correction Workflow

```
Run Validate Topology
        │
        ▼
Count errors by rule type
        │
        ▼
Prioritize:
  Rule 8 (endpoints)  →  fix first (affects Network Dataset)
  Rule 4 (dangles)    →  fix second (connectivity)
  Rule 7 (self-overlap) → fix third (geometry integrity)
  Rules 1-3 (overlap/intersect) → fix fourth
  Rule 5 (pseudo nodes) → evaluate each — may be intentional
        │
        ▼
Open Error Inspector → zoom to each error
        │
        ▼
Edit and correct
        │
        ▼
Re-run Validate Topology
        │
        ▼
Repeat until 0 errors (or all remaining are exceptions)
```

---

## Marking Exceptions

Not all topology errors need to be corrected. Legitimate cases:

| Situation | Rule | Action |
|---|---|---|
| Dead-end road (cul-de-sac) | Must Not Have Dangles | Mark as exception |
| Road ending at map boundary | Must Not Have Dangles | Mark as exception |
| Intentional segment split with different attributes | Must Not Have Pseudo Nodes | Mark as exception |

To mark an exception in ArcGIS Pro:
1. Select the error in Error Inspector
2. Right-click → Mark as Exception
3. Document the reason in the topology exception notes

---

## References

- [ArcGIS Pro — Topology rules](https://pro.arcgis.com/en/pro-app/latest/help/editing/geodatabase-topology-rules-for-line-features.htm)
- [ArcGIS Pro — Validate topology](https://pro.arcgis.com/en/pro-app/latest/help/editing/validate-a-topology.htm)
- Script: [`scripts/check_road_geometry.py`](../scripts/check_road_geometry.py)
