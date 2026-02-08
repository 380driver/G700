# G700 SYMMETRY AVIONICS SUITE
## Complete Synoptic Suite & Flight Deck Display Architecture

---

## Overview

The Gulfstream G700 features the **Honeywell Primus Epic (Symmetry)** avionics suite with 10 integrated touchscreen displays. This documentation describes the complete synoptic suite, display architecture, and integration strategy.

### Key Characteristics
- **Refresh Rates:** Hardware-like LCD stepping (PFD 60Hz, MFD 30Hz, Synoptics 30Hz)
- **Color Scheme:** Honeywell Professional (Cyan, Amber, Red, Dark Grey)
- **Font:** B612 monospace (EASA-approved legibility)
- **Architecture:** Event-driven (G700Bus) with SimVar polling and publish/subscribe pattern
- **Interactivity:** Phase-of-flight intelligence, gesture recognition, sensed checklists

---

## PART 1: SYNOPTIC SUITE

The complete avionics system includes 5 high-fidelity synoptic pages, each displaying a different aircraft system in real-time.

### 1.1 FUEL SYSTEM SYNOPTIC

**Display:** Fuel quantity, distribution, and flow

**Components:**
- **3-Tank System:** Left, Center, Right
- **Boost Pumps:** Independent electrical pumps per tank
- **Center Feed Valve:** Animation showing cross-feed flow
- **Flow Meters:** Real-time fuel consumption rate
- **System Isolation:** Fuel shutoff capability per engine
- **Quantity Indicators:** Animated tank fill levels with numeric readout

**State-Based Coloring:**
- 🟢 Cyan: Nominal operation (> 500 lb per tank)
- 🟡 Amber: Low fuel warning (< 500 lb)
- 🔴 Red: Critical low (< 100 lb)

**Refresh Rate:** 30Hz (via `G700_REFRESH_TICK_MFD`)

**SimVars:**
```typescript
'FUEL TANK LEFT QUANTITY'
'FUEL TANK CENTER QUANTITY'
'FUEL TANK RIGHT QUANTITY'
'FUEL TANK LEFT QUANTITY' // actual burn
'FUEL CENTER VALVE OPEN' // 0-1
```

**Example Flow:**
```
User opens MFD → selects FUEL synoptic
→ Component subscribes to SimVar updates
→ RefreshRateManager publishes REFRESH_TICK_MFD every 33ms
→ updateStatus() polls fuel levels
→ Tank rects animate height proportional to quantity
→ Colors change based on pressure thresholds
```

---

### 1.2 ELECTRICAL SYSTEM SYNOPTIC

**Display:** Generator status, bus distribution, power flow

**Components:**
- **Generators:** Left Engine, Right Engine, APU (3 sources)
- **AC/DC Buses:** Primary distribution with voltage readout
- **Tie Bus Logic:** Cross-tying capability for redundancy
- **Battery:** 28V backup with amp-hours remaining
- **Load Display:** Real-time electrical load in amps
- **System Status:** Animated pulse on active generators

**State-Based Coloring:**
- 🟢 Cyan: Generator online & supplying
- 🟡 Amber: Generator online but standby
- 🔴 Red: Generator failure or offline
- 🟠 Orange: APU Gen available for starting

**Voltage Readouts:**
- AC: 115V nominal (85-145V acceptable)
- DC: 28V nominal (22-32V acceptable)

**Refresh Rate:** 30Hz

**SimVars:**
```typescript
'ELECTRICAL GENERATOR LEFT ACTIVE' // 0-1
'ELECTRICAL GENERATOR RIGHT ACTIVE'
'ELECTRICAL GENERATOR APU ACTIVE'
'ELECTRICAL AC VOLTAGE' // Bus voltage
'ELECTRICAL DC VOLTAGE'
'ELECTRICAL TIE BUS CONNECTED' // 0-1
```

**Example Alert Flow:**
```
Right Gen fails (no power output)
→ ElectricalSynoptic detects 0 voltage
→ Updates right gen circle to red
→ Publishes ELEC_FAILURE event
→ CAS_Manager receives ELEC_FAILURE
→ Displays RED CAS alert "Right Generator Failed"
→ Publishes CHIME_PLAY → AudioManager plays triple-chime
```

---

### 1.3 HYDRAULIC SYSTEM SYNOPTIC

**Display:** Three independent hydraulic systems with pressure and temperature

**Components:**
- **System 1 (Left):** Engine-driven pump, 3000 PSI nominal
- **System 2 (Right):** Engine-driven pump, 3000 PSI nominal
- **System 3 (Aux):** Electric pump, 2500 PSI nominal (standby)
- **Pressure Gauges:** Real-time PSI readout for each system
- **Temperature Readouts:** Hydraulic fluid temperature
- **Crossfeed Valve:** Isolation per system
- **Reservoir Levels:** Animated tank fill for backup supply
- **Pump Status:** Active indicator (green) or failed (red)

**Pressure Thresholds:**
- 🟢 Cyan: Nominal (≥ 2900 PSI)
- 🟡 Amber: Low pressure warning (1500-2899 PSI) — activate backup system
- 🔴 Red: Critical failure (< 1500 PSI) — system offline

**Temperature Limits:**
- Nominal: 40-85°C
- Caution: > 85°C (reduce load)
- Critical: > 100°C (system shutdown)

**Refresh Rate:** 30Hz

**SimVars:**
```typescript
'HYD PRESS LEFT' // psi
'HYD PRESS RIGHT'
'HYD PRESS AUX'
'HYD TEMP LEFT' // celsius
'HYD TEMP RIGHT'
'HYD TEMP AUX'
'HYD PRESSURE RELIEF' // valve status
```

**Real-World Logic:**
- Pump continuously cycles to maintain 3000 PSI
- Low pressure triggers backup pump engagement
- Crossfeed valve allows Left system to supply Right actuation if Right pump fails
- Temperature reduces due to high flow (actuator demand) or external cooling

---

### 1.4 ENVIRONMENTAL CONTROL SYSTEM (ECS) SYNOPTIC

**Display:** Cabin pressurization, oxygen, and temperature zones

**Components:**
- **Cabin Pressure:** Differential pressure gauge (0-8.6 PSI max)
- **Cabin Altitude Indicator:** Display actual or target cabin altitude (0-8,000 ft)
- **Pressurization Mode:** AUTO / MANUAL selection
- **Outflow Valve:** Modulating valve position (0-100%)
- **Oxygen System:** Tank pressure, flow to crew, passenger supply
- **Emergency Oxygen:** Automatic drop-down masks
- **Temperature Zones:** 5 independent zones
  - Flight Deck (Pilot + Copilot)
  - Upper Cabin
  - Main Cabin
  - Lower Cargo
  - Aft Cabin
- **Bleed Air Sources:** Pack 1/2 status from engine bleed or APU
- **Anti-Ice System:** Wing/tail protection armed/disarmed
- **Humidity Control:** Moisture removal system status

**Refresh Rate:** 30Hz

**SimVars:**
```typescript
'CABIN PRESSURE ALT' // feet
'CABIN PRESSURE DIFF' // psi
'O2 PRESSURE' // psi
'O2 FLOW' // lb/h
'TEMP DECK' // celsius (5 zones)
'TEMP UPPER'
'TEMP MAIN'
'TEMP LOWER'
'TEMP AFT'
```

**Temperature Control Logic:**
- Each zone has independent thermal management
- Pilot can set setpoint (+/- 5°C from schedule)
- Automatic bleed modulation maintains target temps
- RAM air inlet provides free cooling at high altitude/speed

---

### 1.5 FLIGHT CONTROLS SYNOPTIC

**Display:** Real-time control surface deflection visualization

**Components:**
- **Primary Flight Controls:**
  - Ailerons (±25° max): Roll control, separated Left/Right indicators
  - Elevators (±20° max): Pitch control, separated L/R indicators
  - Rudder (±30° max): Yaw control, single indicator
  
- **Secondary Controls:**
  - Spoilers (0-60°): Deployed asymmetrically for roll control or symmetrically for speedbrakes
  - Flaps (0-45°): 5 positions (0°, 15°, 25°, 35°, 45°)
  - Trim (±13°): Elevator trim for pitch trim control
  
- **Actuator Status:**
  - Hydraulic pressure to each actuator
  - Redundancy available (dual-actuated surfaces)
  - Reversion mode if hydraulics lost

- **Mode Indicators:**
  - Normal (FBW): Envelope protection engaged
  - Alternate: One hydraulic system failed
  - Direct: Direct mechanical linkage (emergency-only)

**Visualization:**
- Vector lines show neutral position
- Animated deflection bars show real-time movement
- Color coding: Green (normal), Amber (caution), Red (failure)
- Numeric readouts for precise degree measurement

**Refresh Rate:** 50Hz (faster for control feedback)

**SimVars:**
```typescript
'AILERON LEFT' // degrees
'AILERON RIGHT'
'ELEVATOR' // both are synchronized
'RUDDER'
'SPOILER LEFT'
'SPOILER RIGHT'
'FLAPS SET' // degrees
'TRIM ELEVATOR' // degrees
```

---

## PART 2: FLIGHT DISPLAYS

The three primary flight displays (PFD, MFD, TSC) form the cockpit interface.

### 2.1 PRIMARY FLIGHT DISPLAY (PFD)

**Layout:** Full-screen flight data display (typical 8-10" diagonal)

**Components:**

#### Left Side: Airspeed Tape
- Animated vertical tape, 0-250 knots
- Precision: 10-knot major marks, 5-knot minor marks
- Center bug shows current airspeed
- V-speed reference lines (Vne, Vno, Va, Vs1, Vs0) shown as color bands:
  - Red line (Vne): Never-exceed speed
  - Yellow arc (Caution): Careful operation zone
  - Green arc (Normal): Cruise speed range
  - White arc (Flaps): Below this, flaps extended safe

#### Right Side: Altitude Tape
- Animated vertical tape, 0-35,000 feet
- Precision: 1,000-ft major marks, 100-ft minor marks
- Bug shows target altitude
- Trend arrow shows climb/descent rate

#### Center: Attitude Indicator & HSI
- **Pitch Ladder:** ±30° in 10° increments (visual horizon reference)
- **Roll Scale:** ±180° showing wing bank angle
- **Broken-Wing Flight Director:** Gulfstream's iconic FD symbol with asymmetric winglets
- **HSI (Heading Indicator):**
  - 360° compass rose
  - Current magnetic heading (top, green bug)
  - Desired track (yellow bug)
  - Glideslope indicator (centered dot shows glide path) at ±2.5° scale
  - ILS localizer scale (centered needles show left/right deviation)
  - Distance-to-station readout

#### Bottom: Performance Data
- **Vertical Speed (V/S):** Feet per minute, ±3,000 fpm range
- **Mode Indicator:** LNAV (lateral nav), VLOG (vertical nav), AP (autopilot), FD (flight director)
- **Wind:** Wind direction and speed from inertial data
- **DH/MDA:** Decision height or minimum descent altitude
- **Navigation Source:** GPS, ILS, DME, VOR status
- **Time to Destination:** From FMS

#### Background: Synthetic Vision System (SVS)
- Placeholder for 3D terrain model (RenderToTexture integration)
- Shows terrain elevation overlay
- Alerts if aircraft descending toward terrain
- Future: Weather cells, airspace, traffic overlay

**Refresh Rate:** 60Hz (RefreshRateManager publishes `G700_REFRESH_TICK_PFD`)

**Key SimVars:**
```typescript
'AIRSPEED INDICATED' // knots
'INDICATED ALTITUDE' // feet
'HEADING MAGNETIC' // degrees
'TRACK' // degrees (over ground)
'VERTICAL SPEED' // fpm
'PITCH' // degrees (attitude)
'BANK' // degrees (attitude)
'AUTOPILOT ACTIVE VALUE' // 0-1
'AUTOPILOT NAV SELECTED' // 0-1
```

---

### 2.2 MASTER FUNCTION DISPLAY (MFD)

**Layout:** 2×2 Pane Window Manager (modular, configurable)

**Architecture:**
```
┌──────────────────────┬──────────────────────┐
│  UPPER-LEFT          │  UPPER-RIGHT         │
│  (I-NAV Map)         │  (EICAS)             │  
│                      │                      │
│  Terrain, Traffic,   │  Engine & System     │
│  Waypoints, Charts   │  Monitoring          │
│                      │                      │
├──────────────────────┼──────────────────────┤
│  LOWER-LEFT          │  LOWER-RIGHT         │
│  (Synoptic)          │  (Video)             │
│                      │                      │
│  Fuel/Elec/Hyd/ECS   │  Weather Radar,      │
│  Flight Controls     │  Cameras, CCTV       │
│                      │                      │
└──────────────────────┴──────────────────────┘
```

**Pane 1: I-NAV Moving Map**
- Terrain mapping (digital elevation model)
- Active flight plan with waypoint sequence
- Aircraft symbol (green triangle, center)
- Compass rose, scale indicator
- Real-time wind barb
- Navigation info: heading, track, ground speed, ETA
- Pushbutton menu for zoom, pan, chart selection

**Pane 2: EICAS (Engine Indication & Crew Alerting System)**
- Live engine parameters:
  - N1 (fan speed), N2 (core speed)
  - EPR (engine pressure ratio) or thrust indicator
  - ITT (inter-turbine temperature)
  - Fuel flow (lb/h per engine)
- System alerts (integrated with CAS_Manager)
- Fuel endurance time-to-empty
- Gross weight, center of gravity

**Pane 3: System Synoptic (Selectable)**
- User selects: Fuel, Electrical, Hydraulic, ECS, or Flight Controls
- Shows schematic + numeric readouts
- Color state-based indication (OK, Caution, Warning, Failure)
- Touch-to-select pane functionality

**Pane 4: Video Feed (Selectable)**
- Weather radar (animated sweep pattern)
- Forward/aft CCTV cameras
- Thermal imaging (if equipped)
- Display selection buttons at bottom

**Interaction Model:**
- Tap a pane to select it
- Long-press to get menu of available pages
- Swipe to scroll within pane (map pan, list scroll)
- Buttons at bottom for quick-select common configurations

**Phase-of-Flight Auto-Switching (via PhaseManager):**
- TAXI: Map pane shows high-res airport chart
- TAKEOFF/CLIMB: EICAS pane highlights engine data
- CRUISE: Map shows route with weather overlay
- DESCENT/APPROACH: EICAS + Navigation (ILS), lower pane shows ECS
- LANDING: Video feed (forward camera), terrain svs, approach timing

**Refresh Rate:** 30Hz (RefreshRateManager publishes `G700_REFRESH_TICK_MFD`)

---

### 2.3 TOUCHSCREEN CONTROLLER (TSC)

**Display:** 8" diagonal touch interface mounted on glare shield or pedestal

**Interface Sections:**

**1. FMS Tab**
- Flight plan entry
- Waypoint/approach selection
- Performance input (weight, temperature, winds)
- Equipment status

**2. Radio Tab**
- COM1/COM2 frequency ststandby/active
- 12-key numeric keypad with backspace
- Decimal point entry
- NAV frequency selector

**3. Performance Tab**
- Weight input (for V-speed calculation)
- Temperature/altitude for performance calculations
- Auto-calculated V1/VR/V2 display
- Flex temperature input

**4. Electronics/Checklists Tab**
- Electronic checklist navigation
- Sensed item auto-completion
- Manual item selection
- Inhibit logic (can't advance before prior section complete)

**Touch Interaction:**
- All buttons emit 900Hz click sound (AudioManager.playTone)
- Critical buttons (fuel shutoff, engine start) use slide-confirm gesture
  - 80px drag threshold triggers readiness chime (650Hz)
  - Release at threshold triggers confirmation chime (880Hz)
  - Prevents accidental activation on turbulent flights

**Status Indicators:**
- Connection status to FMS
- Autopilot engagement status
- Navigation mode
- Terrain warning (if TAWS equipped)

---

## PART 3: ROUTING & INTEGRATION

### 3.1 URL Parameter Router (App.tsx)

All 10 screens are served from a single bundle via URL parameters:

**Flight Displays:**
```
http://localhost:5173/?type=PFD    → Primary Flight Display
http://localhost:5173/?type=MFD    → Master Function Display
http://localhost:5173/?type=TSC    → Touchscreen Controller
http://localhost:5173/?type=EIS    → Engine Indication System
```

**Synoptic Pages (Standalone or via MFD):**
```
http://localhost:5173/?type=FUEL          → Fuel System
http://localhost:5173/?type=ELEC          → Electrical System
http://localhost:5173/?type=HYD           → Hydraulic System
http://localhost:5173/?type=ECS           → Environmental Control
http://localhost:5173/?type=FLIGHTCTL     → Flight Controls
```

**Overhead/Pedestal Panels (Future):**
```
http://localhost:5173/?id=Overhead_TSC    → Overhead panel controls
http://localhost:5173/?id=Pedestal_TSC    → Pedestal FMS rotary
```

**MSFS Panel.cfg Example:**
```ini
[Window Titles]
Window1=G700 PFD
Window2=G700 MFD
Window3=G700 TSC
Window4=G700 EIS

[PFD]
size_mm=210,157
position=0,0
url=http://localhost:5173/?type=PFD

[MFD]
size_mm=210,157
position=210,0
url=http://localhost:5173/?type=MFD

[TSC]
size_mm=210,157
position=420,0
url=http://localhost:5173/?type=TSC

[EIS]
size_mm=210,157
position=630,0
url=http://localhost:5173/?type=EIS
```

---

### 3.2 Event Flow Architecture

**Example: Electrical System Failure → CAS Alert → Audio**

```
ElectricalSynoptic POLLS
├─> G700Bus.getSimVar('ELECTRICAL GENERATOR RIGHT ACTIVE')
│   └─> Returns 0 (generator offline)
│
├─> updateStatus() detects right gen = 0
│   └─> rightPump.setAttribute('stroke', '#FF0000')
│   └─> G700Bus.publish('ELEC_RIGHT_GEN_FAILED')
│
CAS_Manager SUBSCRIBES
├─> G700Bus.subscribe('ELEC_RIGHT_GEN_FAILED')
│   └─> Creates CAS alert struct:
│       { priority: 'RED', text: 'Right Generator Failed', sound: 'triple-chime' }
│
├─> CAS alert added to stack
│   └─> renderAlert() displays RED box with amber text
│   └─> G700Bus.publish('CHIME_PLAY', { kind: 'triple-chime' })
│
AudioManager SUBSCRIBES
├─> G700Bus.subscribe('CHIME_PLAY')
│   └─> onChime(detail) receiving { kind: 'triple-chime' }
│   └─> playBeep(930Hz) three times with 120ms spacing
│   └─> Web Audio API synthesizes tones via oscillator nodes
│
Cockpit Crew HEARS
└─> Three distinct 930Hz tones (Gulfstream master warning signature)
    └─> Responds to alert visually on CAS stack
    └─> Checks electrical synoptic for root cause
```

---

## PART 4: DISPLAY HIERARCHY & REFRESH RATES

### Refresh Rate Philosophy

Real avionics displays do NOT all refresh at 60Hz. Hardware manufacturers optimize per display type:

| Display | Refresh | Reason | Implementation |
|---------|---------|--------|-----------------|
| **PFD** | 60Hz | Fast attitude/airspeed updates | RefreshRateManager subscribes to `G700_REFRESH_TICK_PFD` |
| **MFD** | 30Hz | Page/pane updates less critical | RefreshRateManager subscribes to `G700_REFRESH_TICK_MFD` |
| **EIS** | 50Hz | Engine parameters sampled quickly | RefreshRateManager subscribes to `G700_REFRESH_TICK_EIS` |
| **Synoptics** | 30Hz | System state changes slower | Same as MFD |
| **TSC** | 33Hz | Touch polling cadence | Fast enough for responsive UI |

### Implementation

All RefreshRateManager subscriptions handled in `src/main.ts`:
```typescript
import { RefreshMgr } from './system/RefreshRateManager';
// RefreshMgr singleton activates on module load
```

Components subscribe to appropriate tick:
```typescript
export class G700_PFD {
  public onAfterRender() {
    G700Bus.subscribe('G700_REFRESH_TICK_PFD', () => this.updateFromSim());
  }
}
```

---

## PART 5: PROFESSIONAL SPECIFICATIONS

### Fonts
- **Primary:** B612 (EASA-certified, high-contrast sans-serif)
- **Fallback:** System UI fonts if B612 unavailable
- **Weights:** Bold for headings, regular for data

### Colors (Honeywell Standard)
```css
--honeywell-cyan:  #00FFFF (Primary data, OK status, active systems)
--honeywell-amber: #FFD700 (Caution, warning, low thresholds, advisory)
--honeywell-red:   #FF0000 (Failure, emergency, critical)
--g700-dark:       #1A1A1A (Background, contrast fields)
--g700-contrast:   #111111 (Minimal contrast for dark backgrounds)
```

### Typography Rules
- Headings: 14-18pt, bold, Cyan
- Data: 12-14pt, regular, Cyan
- Labels: 9-11pt, regular, Light Grey (#CCCCCC)
- Warnings: Amber or Red depending on severity

### Button Styling
- All buttons rounded (radius 4px)
- Border stroke 1-2px, color-coded
- Touch-sensitive (min 44x44 px for fingers)
- Active state: stroke-width 3, slightly brighter outline

### SVG Rendering
```css
svg {
  shape-rendering: geometricPrecision;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
```

---

## PART 6: TEST SCENARIOS

### Scenario 1: Normal Cruise (ELEC + HYD Monitoring)
1. PFD shows: FL250, 450 knots, heading 180°
2. MFD Upper-Left: Navigation map showing route
3. MFD Upper-Right: EICAS shows both engines at 82% N1
4. MFD Lower-Left: Electrical synoptic shows both gens online
5. MFD Lower-Right: Weather radar showing light turbulence ahead
6. TSC: Performance tab showing current fuel endurance 5:30

**Expected Behavior:**
- RefreshRateManager ticks PFD @ 60Hz (smooth tape animation)
- RefreshRateManager ticks MFD @ 30Hz (slightly stepped pane updates)
- All systems display green (nominal)
- No system alerts in CAS

---

### Scenario 2: Engine Failure During Climb

1. Left generator fails (N=0)
2. ElectricalSynoptic detects 0 voltage
3. Right gen remains nominal (3 sources → 2 sources)
4. CAS alert: "Left Generator Failed" (RED)
5. Audio: triple-chime warning
6. MFD EICAS shows Left N1 declining
7. MFD Lower-Left: Electrical now shows left gen as RED circle

**Event Flow:**
```
ELECTRICAL GENERATOR LEFT ACTIVE = 0
→ ElectricalSynoptic.updateStatus()
→ G700Bus.publish('ELEC_LEFT_GEN_FAILED')
→ CAS_Manager receives event
→ Creates RED CAS alert
→ G700Bus.publish('CHIME_PLAY', { kind: 'triple-chime' })
→ AudioManager plays 930Hz warning tone
```

**Crew Action:**
1. Silence master warning (button on TSC)
2. Check EICAS for engine parameters
3. Evaluate weather/terrain for alternate airports
4. Declare emergency if necessary

---

### Scenario 3: Sensed Checklist Progression (Ground)

1. Before-Start checklist displayed on TSC
2. Pilot cycles master switch → "Master ON" auto-marks as AUTO
3. Avionics switch ON → "Avionics Master" auto-marks as AUTO
4. When all items in Before-Start = AUTO, section completes
5. Before-Takeoff section now visible (was inhibited)
6. Pilot completes Before-Takeoff items
7. Ready for engine start

**ECL Inhibiting Logic:**
```
BEFORE_START items all AUTO/DONE?
├─ YES: Before-Takeoff items visible
├─ NO: Before-Takeoff section grayed out
└─ User cannot scroll past current section
```

---

## PART 7: FUTURE ENHANCEMENTS

### High-Priority
1. **Real MSFS SDK Integration:** Replace demo injector with live SimVars
2. **GIS Chart Server:** Pull real airport charts into I-NAV map
3. **Weather Integration:** Real METAR/TAF parsing into synoptics
4. **Full TAWS Terrain:** 3D terrain model (RenderToTexture)

### Medium-Priority
1. **Weather Radar Overlay:** Animated storm cell display on map
2. **Traffic Integration:** Mode-C target blips on HSI
3. **Flight Plan Optimization:** Altitude/route suggestions from FMS
4. **System Failure Cascades:** Real multi-system failures

### Nice-to-Have
1. **Overhead Panel TSC:** Bleed air,  hydraulic isolation, electrical crossfeed
2. **Pedestal FMS Rotary Encoder:** Full FMS control from rotary input
3. **Haptic Feedback:** Joystick vibration on terrain warning
4. **Thermal Imaging:** Integrated forward-looking infrared

---

## PART 8: QUICK REFERENCE

### Component File Locations
```
src/g700/
├── components/
│   ├── G700_PFD.tsx                    (Primary Flight Display)
│   ├── G700_MFD.tsx                    (Basic MFD, 8 pages)
│   ├── G700_MFD_Enhanced.tsx            (4-pane window manager)
│   ├── G700_TSC.tsx                    (Touchscreen)
│   ├── EngineEIS.tsx                   (Engine display)
│   ├── FuelSynoptic.tsx                (Fuel system)
│   ├── ElectricalSynoptic.tsx          (Electrical system)
│   ├── HydraulicSynoptic.tsx           (Hydraulic system)
│   ├── ECS_Synoptic.tsx                (Environmental)
│   ├── FlightControls_Synoptic.tsx     (Control surfaces)
│   ├── CAS_Manager.tsx                 (Crew alerting)
│   ├── ECLManager.tsx                  (Electronic checklists)
│   ├── BrokenWingFD.tsx                (Flight director)
│   └── CVS_Placeholder.tsx             (Synthetic vision)
├── system/
│   ├── RefreshRateManager.ts           (Display refresh scheduling)
│   ├── PhaseManager.ts                 (Flight phase detection)
│   ├── CCDManager.ts                   (Cursor control device)
│   ├── SystemInterdependency.ts        (Electrical/thermal coupling)
│   ├── PerformanceCalculator.ts        (V-speed auto-calc)
│   ├── TouchInteractionRig.ts          (Gesture recognition)
│   └── ...
├── bootstrap/
│   ├── App.tsx                         (Router with documentation)
│   └── G700_SDK_Integrator.ts
├── utils/
│   └── G700_Bus.ts                     (Event/SimVar hub)
├── audio/
│   └── AudioManager.ts                 (Chime synthesis)
├── styles/
│   └── G700_Global.css                 (Colors, fonts, animations)
└── demo/
    └── G700_SimInjector.ts             (Sim data generation)
```

### Key SimVars Reference
```typescript
// Flight Data
'AIRSPEED INDICATED' 'INDICATED ALTITUDE' 'HEADING MAGNETIC' 'TRACK'
'VERTICAL SPEED' 'PITCH' 'BANK'

// Engine Data
'ENG1_N1' 'ENG1_EPR' 'ENG1_ITT' 'ENG1_FUEL_FLOW'
'ENG2_N1' 'ENG2_EPR' 'ENG2_ITT' 'ENG2_FUEL_FLOW'

// Fuel System
'FUEL TANK LEFT QUANTITY' 'FUEL TANK CENTER QUANTITY' 'FUEL TANK RIGHT QUANTITY'

// Electrical
'ELECTRICAL GENERATOR LEFT ACTIVE' 'ELECTRICAL GENERATOR RIGHT ACTIVE'
'ELECTRICAL AC VOLTAGE' 'ELECTRICAL DC VOLTAGE'

// Hydraulic
'HYD PRESS LEFT' 'HYD PRESS RIGHT' 'HYD PRESS AUX'
'HYD TEMP LEFT' 'HYD TEMP RIGHT' 'HYD TEMP AUX'

// Controls
'AILERON LEFT' 'AILERON RIGHT' 'ELEVATOR' 'RUDDER'
'SPOILER LEFT' 'SPOILER RIGHT' 'FLAPS SET' 'TRIM ELEVATOR'

// Flight Phase  
'SIM ON GROUND' 'AIRSPEED' (PhaseManager uses these)
```

---

**Document Version:** 2.0  
**Last Updated:** February 2026  
**G700 Avionics Project**
