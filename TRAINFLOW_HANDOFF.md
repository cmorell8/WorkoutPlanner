# TrainFlow — Agent Handoff Document
**Owner:** Carlos (carlos.morell@clarity.ai)  
**Last updated:** April 2026  
**Primary deliverable:** `trainflow.html` — a single-file mobile-first training app

---

## 1. Project Overview

TrainFlow is a **personal training planning app** (single HTML file, no backend, no build step) that replicates core TrainingPeaks functionality for a single athlete. It connects live to **intervals.icu** via their REST API and syncs planned workouts to **Garmin Connect** through intervals.icu's native Garmin push.

**Phase 1 (done):** Personal prototype for Carlos.  
**Phase 2 (future):** Multi-user product.

**How to run:**
```bash
# Must be served via HTTP for API calls to work (CORS blocks file://)
cd /path/to/folder
python3 -m http.server 8080
# Then open: http://localhost:8080/trainflow.html
```

---

## 2. File Inventory

| File | Size | Description |
|------|------|-------------|
| `trainflow.html` | ~71 KB | The entire app — open this in a browser |
| `TRAINFLOW_HANDOFF.md` | this file | Handoff documentation |

**No other files are needed.** The HTML is fully self-contained. All dependencies load from CDN (React 18, Babel Standalone, Tailwind CSS).

---

## 3. Configuration (hardcoded in HTML)

```javascript
const ATHLETE_ID = "i459250";
const API_KEY    = "fbfaxfky3o03as11afy65xxj";
const BASE_URL   = "https://app.intervals.icu/api/v1";
const AUTH       = "Basic " + btoa("API_KEY:" + API_KEY);
```

The API uses **Basic auth** where the username is literally the string `"API_KEY"` and the password is the API key value. This is intervals.icu's documented format.

---

## 4. Tech Stack

| Layer | Technology |
|-------|-----------|
| UI Framework | React 18 (loaded from CDN via `esm.sh`) |
| JSX transpilation | Babel Standalone (in-browser, `type="text/babel"`) |
| Styling | Tailwind CSS 3 (CDN Play) |
| State persistence | `localStorage` only |
| Data source | intervals.icu REST API |
| No build step | Everything in one `<script type="text/babel">` tag |

---

## 5. App Architecture

### Navigation Pattern
Full history stack — not tab-based routing:
```javascript
const [stack, setStack] = useState([{screen:"home"}]);
const cur    = stack[stack.length-1];
const canBack = stack.length > 1;
const push   = e  => setStack(s => [...s, e]);
const goBack = () => setStack(s => s.length > 1 ? s.slice(0,-1) : s);
const goTab  = id => setStack([{screen: id}]);
```
Each entry in the stack is `{screen: "name", ...props}`. The `Back` button appears automatically when `canBack` is true.

### Screens
| Screen ID | Description |
|-----------|-------------|
| `home` | Dashboard: recent activities, upcoming planned, fitness snapshot |
| `activities` | Full activity list from intervals.icu |
| `plan` | Monthly calendar + weekly summary bar + training phases |
| `analytics` | CTL/ATL/TSB chart, weekly load bar, HR zones, sport mix |
| `settings` | API status, Garmin sync explanation, CORS fix instructions |
| `library` | Workout library (from localStorage + DEFAULT_LIB) |
| `builder` | Workout step editor (creates/edits workouts) |
| `wkt_detail` | Workout detail view with structure chart + "Use for date" button |

### Data Flow
```
intervals.icu API
  ├── GET /activities?oldest=90d → setActivities([])
  ├── GET /events?oldest=7d&newest=60d → setEvents([])   ← planned workouts
  └── GET /wellness?oldest=90d → setFitness([])           ← CTL/ATL/TSB

localStorage
  ├── tf_lib     → workout library (DEFAULT_LIB seed + user additions)
  ├── tf_lib_ver → "v2-fit15" (version key; changing this resets the library)
  └── tf_blocks  → training phases [{phase, name, start, end}]
```

### Key Constants / Lookup Tables
```javascript
// Sport definitions
const SPORTS = {
  Ride: {label:"Ride", emoji:"🚴", color:"#f97316", bg:"#fff7ed"},
  Run:  {label:"Run",  emoji:"🏃", color:"#3b82f6", bg:"#eff6ff"},
  TrailRun: {label:"Trail Run", emoji:"🌲", color:"#10b981", bg:"#ecfdf5"},
  Swim: {label:"Swim", emoji:"🏊", color:"#06b6d4", bg:"#ecfeff"},
  // + Walk, Hike, WeightTraining, Yoga, Rowing, VirtualRide, Workout
};

// Step types
const STYPES = {
  warmup:   {label:"Warm-up",   color:"#f59e0b", bg:"#fffbeb", dIF:0.60},
  active:   {label:"Active",    color:"#3b82f6", bg:"#eff6ff", dIF:0.80},
  rest:     {label:"Recovery",  color:"#10b981", bg:"#ecfdf5", dIF:0.50},
  cooldown: {label:"Cool-down", color:"#8b5cf6", bg:"#f5f3ff", dIF:0.55},
};

// Training phases
const PHASES = {
  Base:     {label:"🌱 Base",     color:"#10b981", bg:"#ecfdf5", desc:"Aerobic base building"},
  Build:    {label:"🔨 Build",    color:"#3b82f6", bg:"#eff6ff", desc:"Intensity & volume"},
  Peak:     {label:"⚡ Peak",     color:"#f59e0b", bg:"#fffbeb", desc:"Race-specific work"},
  Taper:    {label:"🪶 Taper",    color:"#8b5cf6", bg:"#f5f3ff", desc:"Reducing load"},
  Race:     {label:"🏁 Race",     color:"#ef4444", bg:"#fef2f2", desc:"Race day"},
  Recovery: {label:"💤 Recovery", color:"#6b7280", bg:"#f9fafb", desc:"Rest & recover"},
};
```

---

## 6. TSS Calculation

```javascript
const calcTSS = (steps, thr = 170) => {
  let total = 0;
  const walk = ss => ss.forEach(s => {
    if (s.type === "repeat") {
      walk(Array(s.repeatCount).fill(s.steps).flat());
      return;
    }
    const h = (s.duration || 0) / 3600;
    let IF = 0;
    if (s.targetType === "hr_bpm" && s.targetLow && s.targetHigh)
      IF = ((s.targetLow + s.targetHigh) / 2) / thr;
    else if (s.targetType === "hr_pct")
      IF = ((s.targetLow + s.targetHigh) / 2) / 100;
    else
      IF = (STYPES[s.type] || {dIF: 0.75}).dIF;
    total += h * IF * IF * 100;
  });
  walk(steps);
  return Math.round(total);
};
```

Formula: `TSS = Σ (duration_hours × IF² × 100)` where `IF = avgHR / thresholdHR`.

---

## 7. Workout Data Structure

Each workout in the library follows this shape:
```javascript
{
  id: "fit-001",               // unique ID
  name: "6x3´ z5 subida",      // display name
  sport: "Run",                // matches SPORTS keys
  source: "fit_import",        // "fit_import" | "built"
  thresholdHR: 170,            // used for TSS calc
  description: "",             // optional
  steps: [                     // array of step objects
    {
      id: "s1",
      type: "warmup",          // warmup | active | rest | cooldown | repeat
      name: "Warm Up",
      duration: 900,           // seconds
      targetType: "hr_bpm",    // hr_bpm | hr_pct | pace | open
      targetLow: 133,          // bpm (only when targetType = hr_bpm)
      targetHigh: 143,
    },
    {
      id: "s2",
      type: "repeat",
      repeatCount: 6,
      steps: [                 // nested steps (same format, no nesting inside repeat)
        { id: "s2a", type: "active", duration: 180, targetType: "hr_bpm", targetLow: 170, targetHigh: 186, ... },
        { id: "s2b", type: "rest",   duration: 60,  targetType: "hr_bpm", targetLow: 126, targetHigh: 146, ... },
      ]
    }
  ]
}
```

---

## 8. DEFAULT_LIB — 15 Pre-loaded Workouts (from FIT files)

All parsed from Carlos's Garmin FIT files. HR values decoded with `raw - 100` offset (Garmin encoding). Threshold HR = 170 bpm.

| ID | Name | Structure |
|----|------|-----------|
| fit-001 | 6x3´ z5 subida + Monte | warmup → 6×(3'Z5 + rest) → 60'easy → cooldown |
| fit-002 | 7x2´ Z4 (2´rec) | warmup → 7×(2'Z4 + 2'rec) → cooldown |
| fit-003 | 4x4´ Z5 + rodaje 20´ | warmup → 4×(4'Z5 + 2'rec) → 20'cooldown |
| fit-004 | 5x4´ Z4 | warmup → 5×(4'Z4 + 2'rec) → cooldown |
| fit-005 | 4x5´ subidas z4 | warmup → 4×(5'Z4 + rest) |
| fit-006 | 5x5´ subidas z4 | warmup → 5×(5'Z4 + rest) |
| fit-007 | Ritmo subida 2x25´ | warmup → 2×(25'Z3/4 + 18'rest) → cooldown |
| fit-008 | 3x12´ subida progresiva | warmup → 3×(9'Z2/3 + 3'Z4 + rest) [flat, no repeat block] |
| fit-009 | 3x8´ Z4 ondulado | warmup → 3×(8'Z4 + 4'rec) → cooldown |
| fit-010 | 3x15´ Series "N" Z4 | warmup → 3×(15'Z4 + rest) → 15'cooldown |
| fit-011 | 2x20´ subida Z3/Z4 | warmup → 1×(20'Z3 + descent) → 5'rest → 1×(20'Z3 + descent) |
| fit-012 | 15'z2 + 4x30" z5Up | 15'Z2 → rest → 4×(30"Z5 + 1'rec) → 10'Z2 → 4×(30"Z5 + 1'rec) → 10'Z2 |
| fit-013 | Fartlek 5x (4' z2 vs 6' z3) | 5×(4'Z2 + 6'Z3) → cooldown |
| fit-014 | Fartlek 5x (5' z2 vs 5' z3) | 5×(5'Z2 + 5'Z3) → cooldown |
| fit-015 | 15'z2 + 6x30" z5Up | 15'Z2 → rest → 6×(30"Z5 + 1'rec) → 10'Z2 → 6×(30"Z5 + 1'rec) → 10'Z2 |

**Library versioning:** `localStorage.getItem("tf_lib_ver")` must equal `"v2-fit15"` for the cached library to be used. On mismatch, DEFAULT_LIB is loaded and the version key is written. Change `LIB_VER` constant to force a reset.

---

## 9. intervals.icu API

**Base URL:** `https://app.intervals.icu/api/v1/athlete/{ATHLETE_ID}`  
**Auth:** `Authorization: Basic btoa("API_KEY:" + apiKey)`

| Endpoint | Method | Usage |
|----------|--------|-------|
| `/activities` | GET | Fetch completed activities. Params: `oldest`, `newest` (YYYY-MM-DD) |
| `/events` | GET | Fetch planned events. Params: `oldest`, `newest` |
| `/events` | POST | Create a planned workout event |
| `/wellness` | GET | Fetch CTL/ATL/TSB fitness data. Params: `oldest`, `newest` |

**Event POST payload** (to plan a workout):
```javascript
{
  start_date_local: "2026-04-15",    // YYYY-MM-DD
  category: "WORKOUT",
  type: "Run",                        // sport type
  name: "5x4´ Z4",
  description: "...",
  moving_time: 3600,                  // seconds (optional)
  icu_training_load: 75               // TSS (optional)
}
```

**CORS note:** The API blocks requests from `file://` origins. The app must be served via `http://localhost` for API calls to work. When CORS blocks a call, the app falls back to demo data (activities) or local-only storage (events). An amber warning banner appears when a planned workout fails to sync.

---

## 10. Weekly Summary Bar

Located at the top of the Plan screen. Shows stats for the selected week (defaults to current week).

**Navigation:** ‹ › buttons change week offset (`wkOff` state, integer, 0 = this week).

**Metrics:**
| Column | Source | Notes |
|--------|--------|-------|
| TSS | `wkA` (past) or `wkE` (future weeks) | Shows planned TSS when `wkOff > 0` |
| Hours | `wkT / 3600` | Total moving time in hours |
| 🏃 km | `runDist / 1000` | Run + TrailRun distance combined |
| Elev | `total_elevation_gain` sum | Meters |

**Key fix applied:** Activities are filtered by date part only (`substring(0,10)`) to avoid Sunday being excluded (intervals.icu timestamps include time, causing lexicographic comparison to fail on Sunday).

---

## 11. Garmin Sync

Workouts planned in the app are POSTed to intervals.icu as events. intervals.icu then pushes them to Garmin Connect nightly via its native Garmin integration.

**Carlos needs to enable this once in intervals.icu:** Settings → Garmin → "Upload planned workouts".

---

## 12. Known Issues & Pending Work

### Bugs to fix
- **`open` duration steps**: Some FIT steps have `duration: 0` because they use `duration_type: "open"` (no fixed time, e.g. "run until you feel ready"). These show as 0 min in the builder. Could display as "Open" instead of 0.
- **fit-008 (3x12' progresiva)**: No repeat block — the FIT file had no repeat marker, so the 3 sets are laid out flat as 10 separate steps. Works correctly but could be tidied up manually.

### Features not yet built (v2 ideas Carlos mentioned)
- **Import more FIT files** from the Library screen (file picker in-app)
- **Pace targets** (currently HR-only; `targetType: "pace"` is parsed but not displayed in workout chart)
- **Week planner view** (currently shows calendar + week summary, but no drag-and-drop or weekly grid view)
- **Multi-user support** (Phase 2 — would need a real backend)
- **Push notifications** for planned workouts
- **Dark mode**
- **Export to PDF** week plan

### Performance notes
- With `file://` opening, all API calls fail silently and demo data is used
- localStorage has ~5MB limit; with 15 workouts (~11KB) and blocks, this won't be an issue
- The Babel in-browser transpilation adds ~1s cold-start on mobile

---

## 13. Code Patterns Reference

### Adding a new screen
1. Add case to the render block:
   ```jsx
   {cur.screen === "myscreen" && <MyScreen {...cur} goBack={goBack} push={push} />}
   ```
2. Add to `SCREEN_TITLES` object
3. Navigate to it with `push({screen: "myscreen", someParam: value})`

### Adding a new metric to weekly summary
In `PlanScreen`, the metrics array is inside the weekly bar JSX:
```javascript
[
  {l:"TSS",  done: wkOff>0 ? plTSS : wkTSS, plan: wkOff>0 ? 0 : plTSS, u:""},
  {l:"Hours",done: (wkT/3600).toFixed(1), plan:0, u:"h"},
  {l:"🏃 km",done: (runDist/1000).toFixed(1), plan:0, u:""},
  {l:"Elev", done: wkElev, plan:0, u:"m"},
]
```

### API wrapper
```javascript
const api = {
  get:  async (path, params={}) => { ... fetch with GET ... },
  post: async (path, body)      => { ... fetch with POST ... },
};
```

---

## 14. Session History Summary

| Session | What was built |
|---------|---------------|
| 1 | Initial app scaffolding: home, activities, plan (calendar), analytics, settings. intervals.icu API integration with CORS fallback. Navigation stack pattern. |
| 2 | Workout Library screen, Workout Builder (step editor), FIT file parsing prototype, WktDetail screen with structure chart. First FIT workout embedded in DEFAULT_LIB. |
| 3 | Blank page bug fixed (JSX syntax error). Weekly summary bar added to Plan screen with TSS/hours/run/elev. Back button navigation. Sport legend on calendar. |
| 4 | 15 FIT files parsed and embedded into DEFAULT_LIB. Week navigation arrows (browse future/past weeks). Run metric changed to km (Run+TrailRun). Sunday filter bug fixed. Hours calculation fixed (was showing minutes). Library versioning added (clears old localStorage cache). Sync warning banner when API call fails. |
