# Task 3 — Body Redesign (CNC Plywood Body, Trunk Lid, Electronics Enclosures)

**Milestone:** M12 – Krabby-Uno V0.2 Build
**Estimated effort:** ~1 week (part-time; builds on leg dimensions from Task 1 and yaw motor geometry from Task 2)

---

## Narrative

The V0.1 body was a plywood box screwed together with no internal organization — batteries, wiring, and controllers were loosely packed inside. V0.2 replaces this with a fully **CNC-cut plywood body** with diagonal corner braces for rigidity. The body must house all electronics and batteries in organized, accessible compartments, and the lid must open on a hinge like a large trunk for easy access during development.

The body is **entirely CNC wood** — all panels, braces, and dividers are cut from plywood on the CNC (see BOM §5 for panel sizes). The only 3D-printed parts in the body are the small **electronics enclosures** for the 6× H-bridge boards and 3× Arduino Mega + shield stacks.

The body footprint stays at approximately **28″ × 48″** (per BOM §5) but the internal layout is fully redesigned to give every component a dedicated, labeled mount area.

---

## Subtasks

### 3.1 — Body frame CAD

**Deliverable:** Full parametric CAD of the body frame including all panels, braces, and internal dividers.

Requirements (panel sizes per BOM §5):

- **Bottom panel:** 3/8″ plywood, 28″ × 48″, CNC-cut with:
  - 6× leg-attachment holes/slots (matching hinge-pin and yaw-axis geometry from Tasks 1 & 2)
  - Mounting holes for the marine bus bar, circuit breaker, and battery tie-downs
  - Wire pass-through holes for leg harnesses (DB-25 bulkhead or slot)
- **Short side panels (×2):** ~10″ × 28″, CNC-cut from 13/16″ plywood (must support hip-mount loads — reinforce at each leg-attachment point).
- **Long side panels (×2):** ~10″ × 48″, CNC-cut from 3/8″ plywood (lighter duty — rigidity comes from corner braces).
- **Diagonal corner braces (×8):** CNC-cut plywood triangles (per BOM §5) that tie each wall-to-wall corner and wall-to-bottom joint. Screwed into both panels with 1.5″ wood screws. These provide the primary rigidity of the structure.
- **Lateral braces (×4):** 24″ CNC-cut plywood crossbars spanning the short dimension inside the body (per BOM §5). Support the lid and divide the interior into bays.
- **Battery dividers (×2):** 15″ lateral plywood braces separating the two battery bays (per BOM §5).
- **Motor mount braces (2):** 28"x5" plywood braces to mount motors at appropriate distance from  hip flange. Should be screwed to bottom for support.
- **Panel-to-panel joints:** Wood screws + CNC-cut finger joints or lap joints at panel edges. The body should be assembleable with a screwdriver — no glue required for structural joints.

### 3.2 — Internal compartment layout

**Deliverable:** CAD layout on bottom board showing mount locations, clearances, and cable routing for every major component.

The body interior is divided into zones, see pictures of existing Krabby v0.1 for rough positioning.

```
                     FRONT
┌─────────────────────────────────────────────┐
│        |     MEGA1          ORIN   │        |
|  [L1]  |                           |  [L4]  |
│        │           POWER           │        |
│        │            BUS            │        |
│  [L2]  ├───────────────────────────┤  [L5]  │
│        │         BATTERY 2         │        │
│  MEGA2 |                           │ MEGA3  |
|        |───────────────────────────┤        |
│  [L3]  |         BATTERY 1         |  [L6]  |
│        |                           |        │
└─────────────────────────────────────────────┘
                     BACK
```

Requirements:

- **Battery bays (×2):** Place centrally in plywood slots. Battery terminals upward. Clearance for 2 AWG cables to run from terminals to the bus bar.
- **Bus bar + breaker zone:** Forward-center area. The marine-style bus bar ([Amazon B0FL28CCC9](https://www.amazon.com/dp/B0FL28CCC9)) mounts to the bottom panel with 1″ wood screws (per BOM §5). 150 A breaker mounted within 7″ of the second battery's + terminal.
- **Arduino Mega bays (×3):** Close to legs as appropriate. Each Mega + shield sits in its 3D-printed enclosure which itself screws to bottom or side as appropriate. DB-25 connectors face outward toward the nearest legs. Serial JST connectors face front toward MEGA1.
- **H-bridge bays (×6):** Mounted on wall near each leg. Molex Mega-Fit power connectors face toward the bus bar. DB-25 control connector faces toward the corresponding Mega. Actuator Micro-Fit and JST XA connectors face toward the leg wire pass-throughs.
- **Orin bay (×1):** Front of the body, away from bus bar. May need to consider airflow, slats on front as 'eyes'? Mounts on appropriate sized M* pegs on bottom panel. Ethernet, USB, and HDMI ports accessible, USB plugged to MEGA1, power to bus.
- **Yaw motor mounts (×6):** Mount to separators at appropriate offset into left/right of body, at each leg-attachment point (per Task 2). The motor body points upward so cam follower and slot linkage is visible.
- **Cable management:** Clean management of all cables with appropriate lengths. Short enough that probably no ties are needed, but tie as required.

### 3.3 — Trunk-style hinged lid

**Deliverable:** Lid assembly that opens on a hinge like a large trunk/chest, with gas struts or a prop rod to hold it open.

Requirements:

- **Lid panel:** 3/8″ plywood top panel (28″ × 48″), CNC-cut. May be 1/2″ if flex is a concern when the robot is flipped for service.
- **Hinges:** 2× standard 4″ door hinges along one long edge (the "back" of the robot). 
- **Gas struts or prop rod:** The lid is heavy (~5–8 lb of plywood); it must stay open hands-free. 1-2× inexpensive 50–100 N gas struts (e.g. 10″ extended length) mounted to the inside of the body wall and the underside of the lid.
- **Latch:** 2× draw latches on the front long edge to hold the lid closed during operation. Must be operable without tools.
- **Top-side features:** CNC-engrave or stencil "KRABBY-UNO V0.2" and a small Krabby Co logo on the lid exterior. 

---

## Acceptance Criteria

Task 3 is accepted when:

1. Body frame CAD files are committed to the repo (preferred: `krabby-research/hardware/cad/v0.2/body/`).
2. CNC toolpath files for bottom panel, side panels, corner braces, lateral braces, dividers, and lid are committed.
3. Body frame is physically assembled (panels + braces + dividers + lid) and verified to accept all electronic enclosures and battery bays with correct clearances.
