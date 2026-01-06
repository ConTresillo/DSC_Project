
---

# 🚗 SMART PARKING SYSTEM

## Final Consolidated Hardware–Software Specification (Checkpoint Draft)

---

## 1️⃣ SYSTEM OVERVIEW

### 1.1 Purpose

The system provides:

- Automated parking access
    
- Fair, buffer-aware billing
    
- Real-time availability
    
- Hybrid **online + offline** parking support
    
- Minimal infrastructure dependency
    
- High scalability and modularity
    

### 1.2 Explicit Design Choices

The system **intentionally avoids**:

- RFID
    
- Piezoelectric sensors
    
- Car–slot identity binding
    
- Continuous vehicle tracking inside the lot
    

Instead, it relies on:

- **Camera-based entry/exit detection**
    
- **Slot occupancy sensors (state only)**
    
- **Session-based billing**
    
- **Exception-driven enforcement (only for pre-booked slots)**
    

These are _design decisions_, not limitations.

---

## 2️⃣ HIGH-LEVEL ARCHITECTURE

### 2.1 Hardware Components

- Entry Camera Unit
    
- Exit Camera Unit
    
- Slot Sensor Units
    
- LED Indicator Units
    
- Embedded Controller (ESP32 / Raspberry Pi)
    
- Power System (Solar + Battery + optional Grid)
    
- Network (Wi-Fi / Ethernet)
    

### 2.2 Software Components

- Camera Processing Service
    
- Event Ingestion Layer
    
- Core Domain Layer
    
- Application Services
    
- Persistence Layer
    
- Mobile Application
    
- Admin / Security Interface
    

---

## 3️⃣ HARDWARE LAYER

### 3.1 Camera Units (Entry / Exit)

**Responsibilities**

- Detect vehicle entry / exit
    
- Capture number plate (best effort)
    
- Generate time-stamped events
    

**Outputs**

```json
VehicleEntryEvent {
  plate_id | TEMP_ID,
  timestamp,
  camera_id
}

VehicleExitEvent {
  plate_id | TEMP_ID,
  timestamp,
  camera_id
}
```

**Failure Handling**

- Plate unreadable → TEMP_ID
    
- Camera offline → manual fallback
    

---

### 3.2 Slot Sensor Units

**Purpose**

- Detect slot state only: `EMPTY / OCCUPIED`
    
- No identity, no billing logic
    

**Output**

```json
SlotStateEvent {
  slot_id,
  state,
  timestamp
}
```

**Important Rule**

> Slot sensors **never participate** in billing or identity logic.

---

### 3.3 LED Indicator System (FINAL, POWER-OPTIMIZED)

#### 🔹 Core Principle

> **LEDs remain OFF during normal operation.  
> They activate only for exceptions or attention-worthy states.**

---

## 4️⃣ LED DESIGN (FINAL & FROZEN)

### 4.1 Slot-Level LED (Per Slot)

|Slot State|LED|Beep|Meaning|
|---|---|---|---|
|**FREE**|OFF|No|Slot available|
|**RESERVED**|🟡 Yellow|No|Pre-booked, awaiting verification|
|**OCCUPIED_VALID**|OFF|No|Legit parking (online or offline)|
|**OCCUPIED_INVALID**|🔴 Red (Blink)|Delayed|Violation detected|
|**ESCALATED**|🔴 Red (Solid)|Yes|Security notified|

📌 **Key Rule**  
Offline parking in **FREE** slots is always allowed.  
**No LED. No alarm. No policing.**

---

### 4.2 Floor-Level LED (Aggregate)

|Condition|LED|
|---|---|
|At least one slot free|OFF|
|All slots full|🔴 Red|

**Logic**

```text
if free_slots > 0 → OFF
else → RED
```

Purpose: simple _“Can I park here or not?”_ signal.

---

## 5️⃣ EVENT INGESTION LAYER

**Responsibilities**

- Receive raw hardware events
    
- Deduplicate
    
- Normalize timestamps
    
- Filter noise / bounce
    
- Queue safely
    

**Queues**

- ENTRY_QUEUE
    
- EXIT_QUEUE
    
- SLOT_QUEUE
    

---

## 6️⃣ CORE DOMAIN LAYER (BUSINESS LOGIC)

### 6.1 ParkingSession

```text
session_id
plate_id
entry_time
exit_time
buffer_duration
billable_duration
amount
status (ACTIVE / CLOSED / ABANDONED)
```

### 6.2 Buffer Policy

- Removes searching + maneuvering time
    
- Applies only at **session level**
    

```text
billable_time = max(0, exit - entry - buffer)
```

### 6.3 Billing Policy

- Time-based rate
    
- Grace period
    
- Penalties (optional)
    

---

## 7️⃣ APPLICATION SERVICES

### 7.1 SessionManager

- Handle ENTRY / EXIT events
    
- Create / close sessions
    
- Auto-timeout abandoned sessions
    

### 7.2 AvailabilityService

- Count free slots
    
- Drive floor-level LED
    
- Update mobile app
    

### 7.3 ReservationService (Optional but Defined)

```text
reservation_id
plate_id
slot_id
expiry_time
status
```

---

## 8️⃣ FINAL SLOT STATE MACHINE (AUTHORITATIVE)

```text
FREE
 ├─ prebook → RESERVED
 ├─ offline park → OCCUPIED_VALID

RESERVED
 ├─ user cancels → FREE
 ├─ user confirms → OCCUPIED_VALID
 ├─ wrong car parks → OCCUPIED_INVALID
 └─ timeout → FREE

OCCUPIED_VALID
 └─ car leaves → FREE

OCCUPIED_INVALID
 ├─ resolved → FREE
 └─ ignored → ESCALATED

ESCALATED
 └─ security clears → FREE
```

Finite, deterministic, safe.

---

## 9️⃣ USER INTERACTION FLOWS (FINAL)

### Case 1 — User prebooks, system asks “Will you arrive?”

- **NO** → RESERVED → FREE
    
- **YES + pays extension** → RESERVED (expiry extended)
    
- **YES + no pay** → FREE
    

---

### Case 2 — User cancels booking

- RESERVED → FREE
    
- No escalation
    

---

### Case 3 — Someone parks in a FREE slot

- FREE → OCCUPIED_VALID
    
- **No LED, no alarm, no prompt**
    

This is intentional.

---

### Case 4 — Sensor glitch / ambiguity

- No escalation
    
- Silent logging
    
- Admin review only
    

---

### Case 5 — User A prebooks and parks

Flow:

- Slot RESERVED → Yellow ON
    
- App prompt appears
    

Outcomes:

- YES → OCCUPIED_VALID → LED OFF
    
- NO → warning + re-prompt
    
- No response after grace → OCCUPIED_INVALID
    

Escalation is **timeout-based**, not click-based.

---

### Case 6 — User B parks in A’s reserved slot

Flow:

- Sensor detects occupancy
    
- Prompt A: _“Is this your vehicle?”_
    

Outcomes:

- A says NO → OCCUPIED_INVALID → Red blink → Escalation
    
- A says YES → OCCUPIED_VALID → LED OFF
    
- No response → OCCUPIED_INVALID → Escalation
    

Misuse is handled via abuse policy.

---

## 🔟 ABUSE & MISUSE POLICY

**Tracked per user / plate**

```text
strike_count
```

|Strike|Action|
|---|---|
|1|Warning|
|2|Fine|
|3|Temporary ban|
|4+|Permanent ban|

Triggers:

- False confirmations
    
- Ignoring verification repeatedly
    
- Repeated reserved-slot misuse
    

---

## 1️⃣1️⃣ ESCALATION (DEFINED CLEARLY)

Escalation means:

- Security notified with:
    
    - Slot ID
        
    - Timestamp
        
    - Plate(s)
        
- Red LED solid
    
- Beep after delay (e.g., 60s)
    

No automatic towing.  
No aggressive automation.

---

## 1️⃣2️⃣ POWER STRATEGY

- LEDs OFF by default
    
- Beep only for violations
    
- Night dimming optional
    
- Solar-friendly, scalable
    

---

## 🧠 FINAL PROJECT STATEMENT

> This project implements a modular smart parking system using camera-based entry–exit tracking, buffer-aware session billing, and an event-driven architecture.
> 
> The system deliberately avoids fragile per-slot vehicle identification and instead enforces rules **only for pre-booked slots**, enabling a hybrid online-offline parking model that is power-efficient, scalable, abuse-aware, and operationally realistic.

---

If you want, next logical steps are:

- **State diagram (ASCII or UML)**
    
- **Database schema freeze**
    
- **Viva-ready justification answers**
    
- **Minimal ESP32 + backend API contract**
    

Say which one you want.