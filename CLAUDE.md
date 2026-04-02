# Peds Onc Schedule Balancer — Project Context Document

## Project Overview

This is a nurse scheduling tool built for a **pediatric oncology unit** (likely St. Jude). It manages 6-week shift cycles for a day-shift nursing staff, handling auto-balancing, charge nurse rotation, and Excel export for sharing. The tool is used by **two head nurses** (Ryan and his partner Lexie) who co-manage the unit's scheduling.

The current implementation is a **standalone HTML file** (`PedsOnc_Schedule_Balancer.html`) using React 18 via CDN + Babel in-browser transpilation. Data persists in `localStorage`. There is also a Claude artifact version (`.jsx`) that uses `window.storage` for persistence.

---

## Unit Structure & Staffing Rules

### Staff Tiers (3 distinct groups)

#### 1. Head Nurses (2 people)
- **Ryan McBride** and **Lexie Bednarek**
- Each works ~2 days/week as **charge nurse**
- Their schedules are entered manually and **never moved by the balancer**
- They have **no floor nurse requirements** (no shift count, Monday/Friday/weekend targets)
- When a head nurse works a day, they automatically serve as charge — no floor nurse needs to pull charge duty that day
- On days a head nurse is charge, the unit needs **8 floor-level nurses** (not 9)

#### 2. Floor Nurses (~21 nurses, varies)
- Regular weekday nursing staff
- Have shift requirements based on their **category** (see below)
- The balancer can move their shifts to even out daily staffing
- Some are **charge-eligible** (marked with ★) — can serve as charge on days no head nurse is present
- Each nurse tracks: name, charge eligibility (`ce`), category (`cat`), PRN target (`prnTarget`), and their 42-day schedule grid

#### 3. Weekender Nurses (~7 nurses)
- Work **every Saturday and Sunday** by default (pre-filled as locked 🔒)
- Some also work a **fixed weekday** (e.g., every Wednesday or Thursday) — these shifts should be lockable so the balancer doesn't move them
- Their weekday cells are fully editable (not auto-locked) because some weekenders pick up mid-week shifts
- Some are **charge-eligible** — can serve as weekend charge
- Weekender schedules are **not moved by the balancer**
- They do NOT have the same shift requirements as floor nurses

### Nurse Categories (Floor Nurses Only)
Each floor nurse has a category that determines their shift requirements:

| Category | Total Shifts | Mondays | Fridays | Weekends | Notes |
|----------|-------------|---------|---------|----------|-------|
| **Full-time** | 18 | 3 | 3 | 1 | Default for all floor nurses |
| **Part-time** | 12 | 2 | 2 | 1 | Reduced requirements |
| **PRN** | Custom | Scaled | Scaled | Conditional | User sets a target number; Mon/Fri scale proportionally; weekend required if ≥6 shifts |
| **Leave** | 0 | 0 | 0 | 0 | Completely skipped by balancer; grayed out in UI |

**IMPORTANT:** These requirements are **soft goals, not hard rules**. The balancer treats them as preferences — if moving a nurse from a day with 12 staff to a day with 5 staff means they end up with 4 Mondays and 2 Fridays instead of 3/3, the balancer should make that trade. Overall schedule balance across all days takes priority over individual nurse requirements.

---

## Daily Staffing Targets

Every day (weekday and weekend) has the same staffing structure:

| Metric | Target | Acceptable Minimum |
|--------|--------|--------------------|
| Floor-level nurses | 8 | 7 |
| Charge nurse | 1 | 1 |
| **Total on unit** | **9** | **8** |

- When a **head nurse is working** that day → they cover charge → unit needs 8 floor-level staff (target) / 7 (minimum)
- When **no head nurse is working** → a charge-eligible floor/weekender nurse takes charge → unit needs 9 floor-level staff (target) / 8 (minimum), because one of them becomes charge

The "floor-level" count includes both floor nurses AND weekender nurses scheduled that day.

---

## Cell States (Schedule Grid)

Each cell in the schedule grid can be in one of 4 states:

| State | Value | Display | Meaning |
|-------|-------|---------|---------|
| OFF | 0 | Empty | Not scheduled |
| ON | 1 | "X" or "C" | Scheduled (moveable by balancer) |
| UNAVAILABLE | 2 | "—" (red) | Nurse is unavailable (out of town, requested off) — balancer cannot schedule here |
| LOCKED | 3 | "🔒" | Scheduled AND immovable (used for weekender fixed shifts) |

**Click behavior for floor nurses:** off → on → unavailable → off (3-state cycle)
**Click behavior for weekender locked cells:** locked → unavailable → locked (can't accidentally remove)
**Click behavior for weekender unlocked cells:** off → on → unavailable → off (normal cycle)
**Right-click on weekender cells:** toggles lock/unlock (ON ↔ LOCKED)
**Head nurse cells:** simple toggle on/off (shows "C" for charge when on)

---

## Auto-Balance Algorithm

### Philosophy
- **Minimize total moves** — don't move anyone unless necessary
- **Distribute moves fairly** — when moves are needed, spread them evenly so no one nurse gets disproportionately shuffled
- **Requirements are soft** — daily staffing balance takes priority over individual nurse Mon/Fri/weekend targets
- **Conservative** — only move floor nurses; never touch head nurse or weekender schedules

### Algorithm (current implementation)
```
For up to 800 iterations:
  1. Calculate daily staffing counts across all 42 days (floor + weekenders)
  2. Determine each day's minimum and target based on head nurse presence
  3. Collect all understaffed days (below minimum) and overstaffed days (above target)
  4. If no understaffed days exist, switch to smoothing mode:
     collect days below target as "under" and days above target as "over"
  5. If nothing to fix, stop
  6. Sort: worst deficit first, worst surplus first
  7. Try ALL overstaffed→understaffed day pairs to find best move:
     - Skip nurses on leave
     - Nurse must be ON (not locked/unavailable) on the overstaffed day
     - Nurse must be OFF (not unavailable) on the understaffed day
     - Score = -(imbalance improvement × 10) + (requirement damage × 2) + (nurse's move count × 1)
     - Pick the move with the lowest score
  8. Execute the best move, increment that nurse's move counter
  9. Repeat
```

### What the balancer does NOT do:
- Add shifts that don't exist (if a nurse has 10 shifts entered, it stays at 10)
- Move head nurse schedules
- Move weekender schedules (locked or unlocked)
- Move shifts to/from unavailable days
- Move nurses on Leave category

### Known limitations:
- Cannot do **chain moves** (move A from day X→Y, then B from Y→Z) in a single iteration — though the iterative nature partially compensates
- If all nurses on an overstaffed day are also scheduled on every understaffed day, no direct move is possible
- With many part-time/PRN/leave nurses, the total shift pool may be too small to fill every day to target — this is a data problem, not an algorithm problem

---

## Charge Nurse Assignment

### Weekday Logic
1. If a head nurse is working → they take charge automatically
2. Otherwise, pick the charge-eligible floor nurse on duty with the **fewest charge shifts** so far (even rotation)

### Weekend Logic (Sat/Sun Pairing)
- **Critical rule:** The same charge nurse covers both Saturday AND the following Sunday (continuity of care)
- Weekend pairs: Sat (day index `w*7+6`) pairs with Sun (day index `(w+1)*7`)
- Day 0 (first Sunday of cycle) is standalone since there's no prior Saturday
- Priority order for weekend charge:
  1. Head nurse (if working that weekend)
  2. Charge-eligible weekender nurse (preferred — they're already there both days)
  3. Charge-eligible floor nurse working both days (fallback, used when it helps balance charge distribution)
- The two weekend charge nurses **mostly alternate** weekends: Weekender A gets Week 1, Weekender B gets Week 2, etc. — driven by "fewest charge shifts" selection
- If no single candidate can cover both Sat+Sun, falls back to individual day assignment

### Charge Distribution Tracking
- The Charge Roster tab shows who's assigned charge each day across the 6-week cycle
- A Distribution section shows charge shift counts per eligible nurse
- Goal: even distribution across all charge-eligible staff

---

## Data Model

### State Shape (what gets saved to localStorage/storage)
```javascript
{
  heads: [                          // Array of 2 head nurses
    { name: "Ryan McBride", shifts: [false, false, ...] }  // 42-element boolean array
  ],
  wkrs: [                           // Array of ~7 weekender nurses
    { name: "Byard, Stacie", ce: false }  // ce = charge eligible
  ],
  wS: [                             // Weekender schedule: array of 42-element arrays
    [3, 0, 0, 0, 0, 0, 3, ...]     // 3=LOCKED, 0=OFF, 1=ON, 2=UNAVAIL
  ],
  fN: [                             // Array of ~21 floor nurses
    { name: "Burns, Lillian", ce: true, cat: "full", prnTarget: 0 }
  ],
  fS: [                             // Floor schedule: array of 42-element arrays
    [0, 1, 1, 0, 1, 1, 0, ...]     // 0=OFF, 1=ON, 2=UNAVAIL
  ],
  sd: "2026-04-26",                 // Start date (Sunday)
  mc: [0, 0, 2, 1, ...]            // Move counts per floor nurse
}
```

### Storage Keys
- Claude artifact version: `peds-onc-sched-v1` (window.storage)
- Offline HTML version: `peds-onc-offline-v2` (localStorage)

---

## UI Structure

### Header
- App title, staff counts, start date picker
- Action buttons: ⚡ Balance, 📥 Excel, 🔄 New Cycle, ⚙ Staff

### Staff Management Panel (⚙ Staff)
- Togglable panel below header
- Three columns: Floor Nurses, Weekenders, Head Nurses
- Floor nurses have: ★ charge toggle, name input, category dropdown (Full-time/Part-time/PRN/Leave), PRN target input (if PRN), ✕ remove button
- Weekenders have: ★ charge toggle, name input, ✕ remove button
- Head nurses have: 👑 icon, name input
- Add buttons for floor nurses and weekenders

### Tabs
1. **Schedule Grid** — main 42-column grid with all three staff tiers visible
   - Head nurses (blue rows) at top
   - Floor nurses (default rows) in middle
   - Weekenders (green rows) at bottom
   - TOTAL row showing daily staffing counts (color-coded: green=target, yellow=acceptable, red=under)
   - CHARGE row showing who's charge each day (👑/★/⚠)

2. **Requirements** — table showing each floor nurse's stats vs their category requirements
   - Badges: green=met, yellow=close, red=not met
   - Shows moves count and charge shift count
   - Weekenders listed separately below with their own stats

3. **Charge Roster** — 6 weekly cards showing daily charge assignments
   - Distribution summary at bottom

4. **Daily Summary** — 6 weekly cards with bar charts showing staffing levels
   - Target line indicator
   - 👑 icon on days with head nurse coverage

5. **Changes Log** — list of all moves made by the balancer
   - Move Fairness section showing per-nurse move counts

### Confirmation Modal
- Used for destructive actions: New Cycle, Remove Nurse
- Replaces browser `confirm()` which doesn't work in artifact environments

### Footer
- Legend: color/icon meanings
- Requirement summary

---

## Excel Export

The 📥 Excel button generates an `.xls` file (actually HTML tables that Excel opens natively) with multiple sheets:

1. **Schedule** — full grid with color coding (blue=head nurse charge, light blue=scheduled, green=locked weekender, yellow=charge duty, red=unavailable)
2. **Requirements** — per-nurse stats with category labels, showing targets based on their category
3. **Charge** — 42-row table of daily charge assignments

---

## New Cycle Behavior
When 🔄 New Cycle is clicked (with confirmation):
- Clears all floor nurse schedules (resets to OFF)
- Resets weekender schedules to default (weekends locked, weekdays off)
- Clears head nurse schedules
- Resets move counts to 0
- Clears change log
- **PRESERVES:** All nurse names, charge eligibility settings, categories, PRN targets, staff structure

---

## Technical Details

### Current Stack
- React 18 (CDN: `react.production.min.js`)
- Babel standalone (in-browser JSX transpilation)
- No build step, no bundler, no node_modules
- Single HTML file, ~54KB
- localStorage for persistence

### Calendar Layout
- Week starts on **Sunday** (day index 0 of each week)
- 6 weeks × 7 days = 42 total days
- Day index mapping: Sun=0, Mon=1, Tue=2, Wed=3, Thu=4, Fri=5, Sat=6 (within each week)
- Global day index: `weekNumber * 7 + dayOfWeek`
- Weekend pairs for charge: Saturday (`w*7+6`) pairs with the FOLLOWING Sunday (`(w+1)*7`), NOT the same week's Sunday

### Known Variable Name Mapping (code uses short names)
| Short | Meaning |
|-------|---------|
| `TD` | Total Days (42) |
| `TGT` | Target floor nurses per day (8) |
| `MN` | Minimum floor nurses per day (7) |
| `DN` | Day Names short |
| `DNF` | Day Names Full |
| `OFF/ON/UNA/LCK` | Cell states (0/1/2/3) |
| `fN` | Floor Nurses (metadata array) |
| `fS` | Floor Schedule (2D array) |
| `wS` | Weekender Schedule (2D array) |
| `wkrs` | Weekender metadata array |
| `mc` | Move Counts |
| `sd` | Start Date |
| `ce` | Charge Eligible |
| `cg` | Charge data (assignments, floor counts, weekender counts) |
| `hd/heads` | Head nurses |
| `ds()` | Date string formatter |
| `gStats()` | Get nurse stats (total, mon, fri, wknd counts) |
| `chkReq()` | Check requirements against targets |
| `asnCharge()` | Assign charge nurses |
| `isW()` | Is weekend day |
| `isActive()` | Is cell ON or LOCKED |
| `reqScore()` | Score how far a nurse's schedule deviates from requirements |

---

## Real Staff (as of last data export)

### Head Nurses
1. Ryan McBride
2. Lexie Bednarek

### Floor Nurses (with categories based on last export)
| Name | Charge Eligible | Likely Category | Last Known Shifts |
|------|----------------|-----------------|-------------------|
| Burns, Lillian | ★ | Full-time | 18 |
| Davis, Christy | ★ | Part-time | 11 |
| Desai, Nikita | — | PRN | 2 |
| Green, Kelsey | — | Full-time | 17 |
| Hagen, Isabella | — | Full-time | 16 |
| Harle, Erica | ★ | PRN/Leave | 1 |
| Hooper, Liz | — | PRN | 7 |
| Jeitz, Kirsten | — | Full-time | 17 |
| Linville, Dusti | ★ | Part-time | 12 |
| Madden, Hannah | — | Full-time | 18 |
| Maksin, Sarah | — | Part-time | 14 |
| McMillian, Maddie | — | Full-time | 18 |
| Mitchell, Grace | — | PRN | 10 |
| Nelson, Magen | — | Full-time | 18 |
| Patillo, Kristen | — | Full-time | 18 |
| Quinn, Dailon | ★ | Leave | 0 |
| Reardon, Taylor | — | Full-time | 17 |
| Rushing, Sunnie | ★ | Full-time | 18 |
| Stroud, Chandlor | ★ | Full-time | 17 |
| Wilkinson, Molly | — | Full-time | 18 |
| Wolf, Alli | ★ | Full-time | 18 |

### Weekender Nurses
| Name | Charge Eligible | Shifts (last export) |
|------|----------------|---------------------|
| Byard, Stacie | — | 15 |
| Chenge, Kate | — | 12 |
| Everett, Tiffany | ★ | 12 |
| Ford, Sara | — | 9 |
| Joyce, Brooke | ★ | 12 |
| Klusmeyer, Pam | — | 8 |
| Parrish, Devin | — | 15 |

---

## Outstanding Issues & Future Work

### Balancer Improvements Needed
- The balancer still sometimes produces large discrepancies (12 on one day, 5-6 on another). The algorithm scans all over→under pairs but may need **chain move** capability or a **global optimization** approach (simulated annealing, constraint satisfaction) instead of greedy iterative moves.
- Consider: when no direct move is possible from overstaffed→understaffed, try a 2-hop: move nurse A from overstaffed→neutral day, then nurse B from neutral→understaffed day.

### Feature Requests (discussed but not yet built)
- **JSON export/import** for syncing data between the Claude version and standalone HTML version (or between two people's browsers)
- **Standalone version for partner** — the HTML file can be shared on OneDrive; partner opens it in her browser. Currently they'd be separate data silos unless JSON sync is added.

### Technical Debt
- Variable names are extremely terse (fN, fS, wS, mc, etc.) — consider more readable names in a refactor
- The HTML file uses Babel in-browser transpilation which shows a console warning; for production, should precompile
- React + Babel loaded from CDN means first-open requires internet; could embed React inline for true offline-first
- No test coverage
- No error boundaries — React errors crash the whole app

---

## Workflow (How Ryan Uses This)

1. **Start of 6-week cycle:** Hit 🔄 New Cycle (keeps names, clears shifts)
2. **Set start date** to the first Sunday of the new cycle
3. **Enter head nurse schedules** — Ryan and Lexie click their charge days
4. **Nurses submit their requested schedules** — Ryan enters each nurse's requested days into the grid (clicking cells to ON)
5. **Mark unavailability** — click cells to "—" for nurses who are out of town or have days off
6. **Set categories** — mark part-time, PRN, or leave nurses in the ⚙ Staff panel
7. **Hit ⚡ Balance** — algorithm evens out daily staffing
8. **Review** — check Requirements tab for individual nurse stats, Daily Summary for overall picture, Changes Log to see what moved
9. **Manual adjustments** — Ryan tweaks anything the balancer couldn't optimize
10. **Export to Excel** — share with Lexie on OneDrive for review
11. **Finalize** — make any last changes, re-export final version
