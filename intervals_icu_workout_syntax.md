# Intervals.icu Workout Text Syntax — Complete Reference

Use this document to translate workout builder output into valid Intervals.icu plain-text format for syncing structured workouts.

---

## 1\. Basic step structure

Every workout step starts with a dash `-`. The general pattern is:

```
- [cue text]  [duration OR distance]  [intensity target]  [optional cadence]
```

- **Cue text** — any words that appear *before* the first time or intensity token become the on-device prompt shown to the athlete.  
- A step must have either a duration or a distance, plus at least one intensity target.

**Examples:**

```
- 5m 60%
- Warmup 10m 55-65% 90rpm
- Sprint 30s 130%
- Easy run 2km 70-75% Pace
```

---

## 2\. Duration formats

### Time

| Token | Meaning | Example |
| :---- | :---- | :---- |
| `h` | hours | `1h` |
| `m` | minutes | `10m`, `5m` |
| `s` | seconds | `30s`, `90s` |
| combined | h \+ m \+ s | `1h2m30s`, `5m30s` |
| `'` | shorthand minutes | `5'` |
| `"` | shorthand seconds | `30"` |
| combined shorthand | min \+ sec | `1'30"`, `1m30` |

### Distance

| Token | Meaning | Example |
| :---- | :---- | :---- |
| `mtr` | meters | `500mtr`, `400mtr` |
| `km` | kilometers | `2km`, `10km` |
| `mi` | miles | `1mi`, `4.5mi` |

⚠️ **Critical:** `m` means **minutes**, NOT meters. For meters always use `mtr`. For sub-kilometer distances use `0.4km` instead of `400m`.

---

## 3\. Intensity targets

### Power (cycling)

| Format | Example |
| :---- | :---- |
| % of FTP — single | `75%` |
| % of FTP — range | `95-105%` |
| Absolute watts | `220w` |
| Watt range | `200-240w` |
| Power zone | `Z2`, `Z3-Z4` |
| Custom zone | `CZ1`, `CZ2-CZ3` |
| % of MMP | `60% MMP 5m` (% of athlete's 5-min mean max power) |
| % of MMP range | `50-60% MMP 3m` |

### Heart rate

| Format | Example |
| :---- | :---- |
| % of max HR | `70% HR` |
| % of max HR range | `75-80% HR` |
| % of threshold HR (LTHR) | `95% LTHR` |
| % of LTHR range | `90-95% LTHR` |
| HR zone | `Z2 HR` |
| HR zone range | `Z2-Z3 HR` |

⚠️ Always append `HR` or `LTHR` suffix. Without it, a bare `%` is interpreted as FTP percentage.

### Pace (running / swimming)

| Format | Example |
| :---- | :---- |
| % of threshold pace | `78-82% Pace` |
| Pace zone | `Z2 Pace`, `Z3-Z4 Pace` |
| Absolute pace — per km | `5:00/km Pace` |
| Absolute pace — per mile | `8:00/mi Pace` |
| Absolute pace — per 100m | `3:00/100m Pace` |
| Absolute pace — per 100y | `1:45/100y Pace` |
| Absolute pace range | `4:00/100m-5:00/100m Pace` |

Supported absolute pace units: `/km`, `/mi`, `/100m`, `/100y`, `/500m`, `/400m`, `/250m`.

⚠️ Always append `Pace` suffix. Without it, `%` is interpreted as FTP.

---

## 4\. Cadence (cycling only)

Cadence is appended after the intensity target.

```
- 10m 75% 90rpm           single target
- 12m 85% 90-100rpm       cadence range
- 15m ramp 60-90% 85rpm   cadence with a ramp step
```

---

## 5\. Ramps

Use the `ramp` keyword for steps where intensity changes gradually from one value to another. Case-insensitive.

```
- 10m ramp 50%-75%          power ramp up
- 10m ramp 75%-50%          power ramp down
- 15m ramp 60%-90% 85rpm    ramp with cadence
- 10m ramp 60-80% Pace      pace ramp
- 8m ramp 200w-280w         absolute watt ramp
```

### Freeride (ERG off)

```
- 20m freeride
```

Disables ERG mode on smart trainers. Use for unstructured or free-effort blocks.

---

## 6\. Repeats / interval sets

Two valid forms:

### A) Repeat count in the section header

```
Main Set 5x
- 3m 120%
- 2m Z1
```

Generates on-device cues like "Main Set 1/5", "Main Set 2/5", etc.

### B) Standalone `nx` line before a step block

```
5x
- 30s 130%
- 30s 50%
```

⚠️ Rules:

- Always leave **one blank line** before and after every repeat block.  
- **Nested repeats are not supported** — do not place a repeat block inside another repeat block.

---

## 7\. Section headers

Any line that does not start with `-` and is not a standalone `nx` is treated as a section header (label). Headers are displayed as section names in the workout timeline.

```
Warmup
- 10m 55-65%

Main Set 4x
- 3m 100%
- 2m 55%

Cooldown
- 8m 55%
```

---

## 8\. Timed text prompts (in-step messages)

You can schedule text messages to appear at specific seconds within a single step. Useful for form reminders, gear shifts, or lap cues.

### Syntax

```
- [message at 0s]  [seconds]^[message]  [seconds]^[message]  <!>  [duration] [target]
```

### Example

```
- Start easy  30^Pick up cadence  90^Hard effort now  <!>  3m ramp 60-100%
```

Rules:

- Prompt times are in **seconds from the start of that step**.  
- `<!>` is **required** when any timed prompts are used — it separates the prompts from the actual step definition.  
- The first message (at 0s) has no prefix — it appears immediately when the step starts.

---

## 9\. Cosmetic / display formatting

These markdown elements are **ignored by the parser** but improve readability in the workout editor. Safe to include.

```
# Heading H1
### Heading H3
---                      horizontal separator
**bold text**
*italic text*
[link text](https://url)
| col1 | col2 |          tables
```

HTML span classes also work for color:

```html
<p class="text-red">Warning message</p>
<span class="d-none">Hidden text</span>
```

---

## 10\. Complete workout examples

### Cycling VO2max intervals

```
Warmup
- 10m ramp 50%-65% 90rpm

Main Set 5x
- Hard effort 3m 120% 100rpm
- Easy spin 2m Z1 85rpm

Cooldown
- 8m ramp 55%-40% 80rpm
```

### Running — HR-based long run

```
Warmup
- 2km 70-75% HR

Main Set 3x
- 5km 80-85% HR
- 1km 65-70% HR

Cooldown
- 2km 65-70% HR
```

### Running — pace zones with distance

```
Warmup
- 1mi Z1 Pace

Main Set 4x
- 1mi Z4 Pace
- 0.5mi Z1 Pace

Cooldown
- 1mi Z1 Pace
```

### Mixed cycling block with freeride and timed prompt

```
Warmup
- Breathe easy  30^Stay seated  <!>  10m ramp 50-70% 85rpm

Main Set
- 20m freeride

Threshold block 3x
- 8m 95-100%
- 4m 55%

Cooldown
- 8m 55% 80rpm
```

---

## 11\. Parsing rules summary (for code generation)

| Rule | Detail |
| :---- | :---- |
| Step lines | Must start with `-` |
| Section headers | Any line without `-` |
| Repeat marker | `nx` in a header (`Main Set 4x`) or standalone (`4x`) |
| Blank lines | Required before and after every repeat block |
| Intensity suffix | `HR`, `LTHR`, `Pace` must be appended where applicable |
| Distance unit | `mtr` for meters, never bare `m` |
| Ranges | `a-b` format: `80-90%`, `200-240w`, `70-80% HR` |
| Ramp | `ramp a-b` before the target values |
| Timed prompts | `<!>` required as separator before duration/target |
| Nested repeats | NOT supported — flatten them |
| Keywords | Case-insensitive (`Ramp`, `ramp`, `RAMP` all valid) |

---

## 12\. Device compatibility notes

| Feature | Garmin | Zwift |
| :---- | :---- | :---- |
| Cadence ranges | ✓ Supported | ✗ Not supported |
| Ramp steps | ✓ Supported | Partial |
| Step cue text | ✓ Shown on device | Shown in app |
| `%` targets | Uses athlete FTP from Intervals.icu | Uses Zwift FTP setting |
| Absolute `w` targets | Direct watts | Overrides Zwift FTP |

---

*Sources: [Intervals.icu forum — Workout Builder Syntax Quick Guide](https://forum.intervals.icu/t/workout-builder-syntax-quick-guide/123701) and [ZonePace — Intervals.icu Workout Format](https://zonepace.cc/intervals-workout-format)*  
