Yes. Let us freeze the **current working of the physical AERIS prototype exactly as your documents describe it**, without adding the future advanced-avoidance ideas.

The current prototype is an **autonomous, human-supervised disaster-response UAV**. It can autonomously search, detect, geo-tag, report, navigate to an authorized response target, perform basic obstacle response, and return/resume. The human remains the authority for the consequential response/payload action. 

# 1. Current AERIS working — one complete flow

```text
                OPERATOR
                   │
                   │ Search Area / Mission
                   ▼
            ┌───────────────┐
            │   PRECHECK    │
            └───────┬───────┘
                    │
                    ▼
               AUTONOMOUS
                 TAKEOFF
                    │
                    ▼
          GPS / WAYPOINT NAVIGATION
                    │
                    ▼
          ┌─────────────────────┐
          │   SEARCH PATTERN    │
          │  Lawn-mower / Grid  │
          └──────────┬──────────┘
                     │
             ┌───────┴────────┐
             │                │
             ▼                ▼
          CAMERA          TFMini Plus
             │                │
             ▼                ▼
        YOLO11n          Distance sensing
             │                │
             ▼                ▼
         Detection       Obstacle logic
             │                │
             ▼                │
        ByteTrack             │
             │                │
             ▼                │
       3-of-5 confirm         │
             │                │
             ▼                │
          GEO-TAG             │
             │                │
             ▼                │
         PRIORITY             │
             │                │
             ▼                │
        INCIDENT ─────────────┤
             │
             ▼
       GROUND STATION
             │
             ▼
        HUMAN REVIEW
             │
       ┌─────┴─────┐
       │           │
   Continue      Response
    Search       Authorized
                    │
                    ▼
             RESPONSE TARGET
                    │
                    ▼
             AUTONOMOUS NAVIGATION
                    │
              obstacle detected?
                 /       \
               NO        YES
               │          │
               │       basic avoidance /
               │       stop / reroute
               │          │
               └────┬─────┘
                    ▼
                TARGET
                    │
                    ▼
           HUMAN RELEASE AUTH
                    │
                    ▼
             SMALL PAYLOAD
                    │
                    ▼
          RETURN / RESUME SEARCH
```

The final hackathon plan explicitly gives **search → survivor detection → geo-tag → alert → operator authorization → response → release → return/resume** as the intended physical mission. 

---

# 2. Step 1 — Operator gives the mission

The ground operator defines either:

```text
Search Area
```

or

```text
Target Coordinate
```

The current physical prototype supports autonomous mission execution and grid/lawnmower-style search. 

Example:

```text
Search Area:
50 m × 50 m

Altitude:
configured mission altitude

Pattern:
lawnmower/grid
```

The Pi mission software and/or ground station prepares the mission, while **Pixhawk + ArduPilot executes the actual flight**. 

---

# 3. Step 2 — Preflight and autonomous takeoff

Before the mission:

```text
Battery
GPS
EKF
RC
Compass
Camera
TFmini
Telemetry
```

are checked.

Then:

```text
TAKEOFF
   ↓
TRANSIT
   ↓
SEARCH
```

The flight controller owns stabilization, navigation, waypoint following, RTH and safety-critical failsafes. 

This is why the Pi can fail without becoming the aircraft's flight controller.

---

# 4. Step 3 — The drone searches the area

The search path is a deterministic **lawnmower/grid pattern**.

Conceptually:

```text
→ → → → → → → →
                ↓
← ← ← ← ← ← ← ←
↓
→ → → → → → → →
                ↓
← ← ← ← ← ← ← ←
```

The current master specification describes the route generation as:

```text
Search polygon
      ↓
Parallel sweep lines
      ↓
Clip to polygon
      ↓
Alternate directions
      ↓
Waypoints
```

with camera footprint and lane overlap used to determine spacing. 

---

# 5. Step 4 — Camera continuously observes the environment

The Camera Module 3 produces frames.

```text
Camera
  ↓
Frame
  ↓
Pi
  ↓
Hailo-8L
```

The camera is the input to the AI perception pipeline. The physical build plan specifies the Pi 5 + AI HAT+ + Camera Module 3 arrangement. 

---

# 6. Step 5 — YOLO11n identifies objects

The current software specification freezes:

**Custom YOLO11n**

with the initial physical classes:

```text
PERSON
VEHICLE
DEBRIS
```

The detector produces:

```text
class
confidence
bounding box
timestamp
```

For example:

```text
person
confidence = 0.91
bbox = [x1,y1,x2,y2]
```

The current master specification uses YOLO11n as the primary physical detector and Hailo-8L as the inference accelerator. 

---

# 7. Step 6 — ByteTrack follows the detected person

The detector sees objects frame-by-frame.

ByteTrack connects those detections:

```text
Frame 1 → person → Track 12
Frame 2 → person → Track 12
Frame 3 → person → Track 12
Frame 4 → person → Track 12
```

So AERIS knows:

> “This is a persistent target, not a completely new person every frame.”

The current specification uses **ByteTrack + event de-duplication** for this purpose. 

---

# 8. Step 7 — Detection is confirmed

A single frame should not automatically become an incident.

The current rule is:

```text
3 detections
within
5 inference frames
```

So:

```text
Frame 1  ✅
Frame 2  ❌
Frame 3  ✅
Frame 4  ✅
Frame 5  ❌

3 of 5 = CONFIRMED
```

This is temporal confirmation, not another neural network. 

---

# 9. Step 8 — AERIS calculates the approximate location

Now the system needs:

> “Where is that person?”

The camera gives a pixel location.

The system combines:

```text
Camera pixel
+
Camera calibration
+
Camera mounting orientation
+
Aircraft attitude
+
Altitude
+
GPS
```

to estimate the point on the ground.

The master specification uses:

$$
r_c=K^{-1}[u,v,1]^T
$$

then transforms the ray into the world frame and intersects it with the ground plane. 

The result is approximately:

```text
Target:
Latitude  = ...
Longitude = ...
Geo quality = APPROXIMATE
```

This is **not simply copying the drone's GPS into the target record**.

---

# 10. Step 9 — AERIS creates an incident

The system builds an incident record containing things such as:

```text
event_id
timestamp
class
confidence
track_id
drone position
target position
priority
flight mode
operator status
```

The event is stored locally and sent to the dashboard.

The overall data flow in the master specification is:

```text
Camera
 ↓
YOLO
 ↓
ByteTrack
 ↓
confirmed track
 ↓
geo-tag
 ↓
priority
 ↓
incident event
 ↓
dashboard + SQLite/JSONL
```



---

# 11. Step 10 — Priority is calculated

The current system does not train a separate “rescue priority AI.”

It uses a transparent weighted rule.

The current specification defines:

$$
Priority =
100(0.55c+0.20t+0.15s+0.10m)
$$

where:

* \(c\) = confidence
* \(t\) = persistence
* \(s\) = class severity
* \(m\) = mission-context severity

Then it maps the score to:

```text
≥80      CRITICAL
60–79    HIGH
40–59    NORMAL
<40      LOG ONLY
```



---

# 12. Step 11 — The ground operator receives the incident

The ground station gets:

```text
Person detected
Confidence
Coordinates
Priority
Timestamp
Mission status
```

The operator sees the event on the map/dashboard.

The prototype build plan explicitly includes a ground station for map, mission planning, telemetry and AI alerts. 

---

# 13. Step 12 — The drone does NOT decide to deliver by itself

This is the human-supervision boundary.

The drone can autonomously do:

```text
search
detect
geo-tag
report
```

But when it comes to a consequential response:

```text
Incident
  ↓
Operator review
  ↓
Operator authorizes response
```

The verified incident becomes a **response waypoint**. 

---

# 14. Step 13 — Response mission begins

The target coordinate becomes:

```text
RESPONSE TARGET
```

Then:

```text
Pi Mission Manager
        ↓
allowed MAVLink command
        ↓
Pixhawk
        ↓
ArduPilot
        ↓
GPS navigation
        ↓
Target
```

During the RESPONSE state, Pixhawk controls the aircraft while the Pi watches:

```text
battery
telemetry
mission state
obstacle state
```



---

# 15. Step 14 — Current prototype obstacle behavior

This is the part we were discussing earlier.

The physical prototype currently has:

**TFmini Plus**

It measures forward distance.

```text
TFmini
 ↓
distance samples
 ↓
filter
 ↓
obstacle state
```

The current specification defines initial states roughly as:

```text
> 8 m        NORMAL

6.5–8 m      WARNING

≤ 6.5 m      HOLD
```

Those are starting values that must be tuned from testing. 

### Important:

Your **final hackathon plan** calls the physical prototype capability:

> “Basic avoidance/stop/re-route logic; not 360° advanced avoidance.” 

So the current physical system does **not** claim full 3D autonomous obstacle avoidance.

It can perform a **basic obstacle response**, potentially including a tested reroute behavior, but the single TFMini does not provide complete 3D environmental perception. The document explicitly notes that limitation. 

---

# 16. Step 15 — After obstacle response, mission can continue

This is important to your original idea.

The physical build plan explicitly describes:

```text
Obstacle detected
     ↓
Avoid / response
     ↓
Continue mission
```

and Example A says the drone can search the area, detect a person, report it, **continue searching**, and later return when the mission/battery condition requires it. 

For a response mission, the intended workflow is:

```text
Target
 ↓
Obstacle
 ↓
Basic avoidance / reroute
 ↓
Continue toward target
 ↓
Target
```

The current architecture supports this at the **basic prototype level**, not as the advanced 3D avoidance system of the full concept.

---

# 17. Step 16 — Arrival at the target

When AERIS reaches the response location:

```text
Target reached
 ↓
Check vehicle state
 ↓
Check battery
 ↓
Check GPS/EKF
 ↓
Check obstacle state
 ↓
Check payload state
```

Only when the safety/interlock conditions pass does the payload path become available.

---

# 18. Step 17 — Human-authorized payload release

The system deliberately does:

```text
AI detection
      ✕
      ↓
No direct servo command
```

Instead:

```text
Operator authorization
       ↓
Mission Manager
       ↓
MAVLink
       ↓
Pixhawk
       ↓
Servo
       ↓
Payload release
```

The final build plan says the payload is only used if safe thrust margin is demonstrated, and the master architecture requires explicit human authorization. 

---

# 19. Step 18 — Return or resume

After the response:

```text
RESPONSE COMPLETE
        ↓
RETURN
        OR
RESUME SEARCH
```

The full mission chain explicitly includes **return-to-home/failsafe**, and the search→response→continue example explicitly says the drone returns and resumes the search.  

---

# 20. What happens if something fails?

This is where the autonomy boundary becomes very important.

### Battery problem

```text
Battery low
 ↓
ArduPilot battery failsafe
 ↓
RTL
```

### RC loss

```text
RC loss
 ↓
ArduPilot failsafe
 ↓
configured safety action
```

### Pi crash

```text
Pi crash
 ↓
AI unavailable
 ↓
Pixhawk remains flight controller
 ↓
mission abort/return according to policy
```

### Hailo failure

```text
AI inference failure
 ↓
No valid AI events
 ↓
Do not pretend AI is healthy
```

### GPS/EKF problem

```text
GPS/EKF unhealthy
 ↓
Do not conduct autonomous response
 ↓
safety action
```

These failure modes are explicitly included in the master specification. 

---

# 21. So what is actually “AI” and what is “autonomy”?

This is the cleanest way to explain AERIS to your mentor.

### AI

```text
YOLO11n
   ↓
"What do I see?"

ByteTrack
   ↓
"Is this the same object?"

Detection confirmation
   ↓
"Is it persistent enough?"

Priority logic
   ↓
"How important is it?"
```

### Robotics/autonomy

```text
Coverage planner
   ↓
"Where should I search?"

Geo-tagging
   ↓
"Where is the detected target?"

Mission FSM
   ↓
"What mission state am I in?"

Obstacle logic
   ↓
"Is the current route blocked?"

ArduPilot
   ↓
"How do I physically fly the mission?"
```

### Human supervision

```text
Operator
   ↓
"Should the response/payload action happen?"
```

---

# 22. The current prototype in one sentence

**AERIS autonomously flies a planned search mission, processes camera images with a trained YOLO11n detector, tracks and confirms detected people/objects, estimates their approximate coordinates, creates and sends an incident to the operator, accepts an operator-authorized response waypoint, navigates autonomously toward it with basic forward obstacle sensing/avoidance, performs a human-authorized small payload release when validated, and then returns or resumes the mission.** 

That is the **current working system**.

The thing we should **not** claim yet is:

> “The physical F450 has full 360°/3D autonomous obstacle avoidance.”

Your own plan explicitly limits the physical prototype to basic obstacle response and reserves advanced 3D avoidance for the full-scale/digital-twin direction. 
