# 171 Lelawood — Home Assistant Config

> **For future Claude instances:** This file is the source of truth for how this system is architected, what's already been decided, and what NOT to do. Read it before suggesting changes. Update it when architecture decisions change.

---

## Project overview

Smart home automation for 171 Lelawood (Nashville). Goal is motion- and lux-driven lighting that "just works" without manual intervention, with local control preferred over cloud dependency. Lighting is the first system; cameras (Frigate), locks, thermostats, and additional sensors come next.

**The user is Leo.** Partner is Kyle. Kyle may have independent morning routines — bedroom automations should accommodate that.

---

## Architecture

Devices → **Home Assistant** (automation brain) → **HomeKit Bridge** → **Apple Home** (UI only)

HA does the thinking. Apple Home is the daily UI. Anything we automate should expose cleanly through HomeKit Bridge.

| Layer | What it does |
|---|---|
| HA | All logic, blueprints, scenes, automations, integrations |
| HomeKit Bridge | Exposes HA entities and scenes to Apple Home |
| Apple Home | Daily UI, voice control via Siri, manual scene triggers |
| Apple TVs | HomeKit hubs (for remote access and reliability) |

---

## Infrastructure

- **Host:** GMKtec NucBox G10 mini PC running Windows
- **HA runtime:** Home Assistant OS in **VirtualBox** with **Bridged Adapter** networking (must point to Ethernet, not WiFi)
- **Network:** TP-Link Deco mesh, separate Main and IoT networks
- **Repo:** This repo. Public. `secrets.yaml`, `.storage/`, and the database are gitignored.
- **Editing:** Studio Code Server add-on inside HA. Commits/pushes from its built-in terminal.

### Known infrastructure pain

- VirtualBox is a persistent source of instability. Dirty VM shutdowns can wipe Matter commissioning fabric for the Leviton dimmers. Bare-metal migration is on the roadmap but not yet done.
- USB passthrough for the Zigbee dongle fails after host reboots/sleep. Fix: replug while VM is running, verify USB filter active, cold restart if needed.
- mDNS/multicast for Matter requires Ethernet (not WiFi) on the HA host AND VirtualBox **Promiscuous Mode = Allow All**.

---

## Key devices and integrations

| Device | Count | Integration | Notes |
|---|---|---|---|
| Leviton D26HD WiFi dimmers | ~20 | Matter | Firmware 2.x.x required (via MyLeviton Matter Early Access) |
| Philips Hue bulbs | many | Hue Bridge integration | Survives HA restarts cleanly (cloud-backed pairing) |
| Aqara FP2 presence sensor | 1 (living room) | HomeKit Controller (WiFi) | Primary motion + lux source for living room |
| Aqara Zigbee motion sensors | several (e.g. gym) | ZHA | First-gen Zigbee |
| Inswift ZBM-MG24 Zigbee dongle | 1 | ZHA (EZSP, 115200 baud) | Silicon Labs chip |
| Apple TVs | several | HomeKit | Hubs |

### Active integrations in HA
ZHA · Matter · Hue Bridge · Aqara (via HomeKit Controller) · HomeKit Bridge

---

## Automation engine

**Blacky's Sensor Light blueprint** — `Blackshome/sensor-light.yaml`

Chosen over Derek Seaman's blueprint and other alternatives. The blueprint handles motion triggers, lux-based brightness, sun elevation, auto-off timers, night lights, and bypass internally. **One blueprint instance per light group per behavior mode** — don't try to consolidate.

### Blueprint architecture rules

- One instance per group per mode (e.g., living room lamps have two instances: daytime + nightlight)
- **Bypass switches** (Low Light, Cooking, Movie modes) handle intentional overrides — exposed to Apple Home
- **Whole-home scenes** are reserved for Good Night and Away only
- **Manual control** (tap dimmer or voice) is the intended user-facing override. NO software kill switches, NO toggle helpers unless absolutely necessary

### Blueprint gotchas (learned the hard way)

| Gotcha | Rule |
|---|---|
| Light Groups in blueprints | Target individual entities directly in the blueprint config. `expand()` on Light Group Helpers can fail silently. |
| Standalone Sun Elevation Gate | Disable it. It's redundant with the Dynamic Lighting section's built-in sun fields and causes problems (passes when sun is *below* threshold). |
| Outdoor lux sensors | Incompatible with blueprint lux fields — readings far exceed slider max. Use indoor sensors only. |
| Min brightness on overheads | Keep at 0% so overheads turn off naturally at 5° sun elevation. |
| Lux feedback loop | FP2 reads its own light output. Mitigate by raising `dynamic_lighting_min_lux` and the minimum brightness floor. |

---

## Entity topology

### Lighting entity reference

| Entity | Type | Notes |
|---|---|---|
| `light.living_room_overhead` | Light Group | Entry, Fireplace, Sitting, Sliding Door overheads — **do not target this group in the blueprint, target individuals** |
| `light.entry_overhead`, `light.fireplace_overhead`, `light.sitting_overhead`, `light.sliding_door_overhead` | Leviton Decora | Living room overheads — target individually in Main Overhead Dynamic |
| `light.dining_overhead` | Leviton Decora | Dining room overhead |
| `light.brown_lamp`, `light.silver_lamp`, `light.guzzini`, `light.parquet_lamp`, `light.square_lamp`, `light.thin_lamp`, `light.big_lamp`, `light.small_lamp`, `light.uplight`, `light.desk_lamp` | Hue / smart plug | Living room lamps |
| `light.hue_white_lamp_15` | Hue | Dining room |
| `light.sconces` | Light Group | `light.hue_white_lamp_10` through `light.hue_white_lamp_14` (guest room) |
| `light.kyle`, `light.kyle_2`, `light.leo`, `light.leo_2` | Hue | Bedroom bedside lamps |
| `light.leo_bedside`, `light.kyle_bedside`, `light.pagoda`, `light.ceramic` | Hue / plug | Bedroom lamps (used in Bedroom Wake Up) |
| `light.levds_dimmer_c766` | Leviton Decora | Bedroom overhead |
| `light.fixture`, `light.bar` | Leviton / Hue | Kitchen accent lights — `light.fixture` treated as a lamp at 100% |
| `light.d26hd` | Leviton Decora | Kitchen overhead |
| `light.chandelier` | Leviton Decora | Dining chandelier |
| `light.gym` | Hue / smart plug | Gym light (uses HA stock motion_light blueprint, not Blacky) |
| `light.hue_white_lamp_16`, `light.hue_white_lamp_16_3` | Hue | Hallway overheads, friendly names "Hall 1"/"Hall 2" — entity IDs predate the rename, **not** `light.hall_1`/`light.hall_2` (verified live 2026-07-08). Targeted individually in Hallway Overhead (custom dim-standby). Both belong in the Hallway area (not yet assigned as of 2026-07-08 — assign in UI). |
| `light.front_porch`, `light.back_porch`, `light.front_flood`, `light.back_flood` | Exterior | Only appeared in the Good Night scene until 2026-09-04 — no automation referenced them. Now driven by Vacation Mode - Exterior Lights. **Entity IDs taken from `scenes.yaml`, not verified live.** |
| **`light.silver_lamp`** | — | **Excluded from occupied/vacation looks** |

### Sensors

| Sensor | Entity | Notes |
|---|---|---|
| Living room presence + lux | `binary_sensor.living_room_motion_devices` + `sensor.presence_sensor_fp2_0fdb_light_sensor_light_level` | Aqara FP2 — reports **continuous presence** (no discrete edges) |
| Kitchen motion | `binary_sensor.kitchen_motion` | Motion trigger for Kitchen Overhead and Kitchen Nightlight (replaced `binary_sensor.kitchen_occupancy` 2026-07-29). Kitchen Overhead still reads the **living room** FP2 for lux — second FP2 still needed |
| Gym motion | `binary_sensor.gym_motion` | First-gen Aqara |
| Hallway motion | `binary_sensor.hallway_motion` | Single sensor entity (replaced the `binary_sensor.hallway_1` / `hallway_2` pair 2026-07-29). Aqara Zigbee (ZHA), Hallway area, detection interval 2s (minimum) — do not change sensor config |

### Helpers and controls

| Helper | Purpose |
|---|---|
| `input_boolean.input_boolean_good_night` | Gates all night behavior. Flipped ON by Good Night scene, OFF by Good Morning 5AM automation. **The double-prefix in the entity ID is NOT a bug — do not assume it should be `input_boolean.good_night`.** |
| `input_boolean.input_boolean_away_mode` | Bypass for all blueprints (Turn Lights OFF). Written only by Vacation Mode - Sync Away Mode; otherwise flipped by hand. |
| `input_boolean.vacation_mode` | Gates vacation mode. **UI helper — note this one has NO double prefix**, unlike `good_night` and `away_mode`. Create as Helper → Toggle named `Vacation Mode`. |
| `scene.good_night` | Apple Home / Siri-triggered. Flips `good_night` ON, turns off overheads and non-nightlight lamps. |

---

## Good Night / Good Morning system

### Good Night
- Triggered from Apple Home / Siri
- Flips `input_boolean.input_boolean_good_night` ON
- Turns OFF all overheads + non-nightlight lamps
- **Nightlight entities are excluded from the scene** so Apple Home shows scene as active while the blueprint runs lamps at low brightness on motion

### Good Morning
- `automation.good_morning_5am` — fires at 5 AM, flips `good_night` OFF, then triggers retrigger
- `automation.good_morning_retrigger_daytime_lamps` — targets `automation.main_lamps_dynamic_n`
- **Why the retrigger exists:** The FP2 reports continuous presence and never produces discrete off→on edges. State_control-gated automations (like Main Lamps) won't re-fire when `good_night` flips OFF without an explicit `automation.trigger` call.
- **Why overheads don't need it:** Time-gated automations (like Main Overhead Dynamic, with `after_time: 05:00:00`) get re-evaluated by Blacky's blueprint internal time trigger at the configured `after_time`. No retrigger needed.
- **Rule for future automations:** Add to the retrigger only if the automation is state_control-gated AND has no `after_time`/`before_time` fallback.

### Bedroom wake-up
- Separate standalone automation (`Bedroom Wake Up - Weekdays`) — gradual ramp, weekdays only
- Kept independent from Good Morning because Kyle may have different routines
- No toggle helpers or kill switches — manual dimmer/voice is the override

---

## Vacation mode

**Built 2026-09-04.** Previously this section described a design that was never implemented — no `vacation` automations existed in `automations.yaml`. Now five automations plus three gating fixes.

Distinct from `away_mode`: **away = short trip, house goes dark. Vacation = house should read as occupied.** Vacation flips `away_mode` ON as its first act, which bypasses every Blacky instance to OFF and gives the vacation automations a clean canvas — the blueprints and the vacation look never contend for the same lamp.

### The five automations

| # | Automation | Behavior |
|---|---|---|
| 1 | Vacation Mode - Sync Away Mode | Vacation ON → `away_mode` ON; OFF → OFF. **The only automation that writes `away_mode`** — everything else reads it. |
| 2 | Vacation Mode - Exterior Lights | Sunset on / sunrise off. Brightness matches the Good Night scene (front porch 65%, front flood 75%, back porch + flood 100%). |
| 3 | Vacation Mode - Evening Lamps | `light.brown_lamp` + `light.guzzini` at 20% from sunset, off at a **random point between 23:30 and 00:30** so it doesn't read as clockwork. |
| 4 | Vacation Mode - Motion Override | Any of the three motion sensors → lamps to 60% for 30 min, then restore baseline. Kitchen overhead only while the sun is up. `mode: restart`. |
| 5 | Vacation Mode - Deactivation Cleanup | Vacation OFF → clear exterior + occupied-look lamps. Interior blueprints resume via #1. |

### Gating fixes shipped alongside

Three automations had no away/vacation gate and would have run away with the house empty:

| Automation | Was | Now |
|---|---|---|
| Bedroom Wake Up - Weekdays | Fired 4:50 AM Mon–Fri and ramped to 80% / 60%. **Nothing in it ever turns the lights off** — only the Siri-triggered Good Night scene does. Left the bedroom lit 24/7 from the first weekday away. | Skipped when `away_mode` OR `vacation_mode` is on |
| Sitting Room Lamps | Bare 06:00 trigger with `conditions: []` — switched on daily during a trip, off only via Good Night scene or `away_mode`→ON. `away_mode` did **not** survive it. | 06:00 branch gated; `arrive_on` branch deliberately left ungated |
| Hallway Overhead | No away gate at all — held 20% day / 5% evening standby indefinitely, never reaching night mode (`good_night` is cleared daily by Good Morning - 5AM and nothing sets it) | Vacation branch first in both `choose` blocks: motion → 30%, standby → **off** |

### Notes

- `light.silver_lamp` excluded from the occupied look (per entity topology).
- Vacation branches are checked **before** `good_night` in Hallway Overhead — `good_night` is stuck OFF during a trip, so a `good_night`-first ordering would never reach the vacation case.
- `Gym lights` (stock `motion_light` blueprint) has no bypass input and stays motion-driven during vacation. Deliberate — a visitor walking through gets light.
- Den deferred.

---

## Critical "don't do this" rules

**Read these before suggesting any of the following:**

1. **Never re-enroll a Leviton dimmer in the MyLeviton cloud app after factory reset.** It rolls firmware back to pre-Matter 1.6.6. Correct sequence: update to 2.x.x via MyLeviton Matter Early Access → capture QR code → factory reset → commission directly to HA → never go back through the Leviton app.
2. **Never restore from backup after Matter commissioning issues.** It can corrupt the fabric state for working nodes.
3. **Never use Light Group entities in Blacky blueprint config.** Target individual lights.
4. **Never enable the standalone Sun Elevation Gate in Blacky.** Use only the Dynamic Lighting section's sun elevation fields.
5. **Never use outdoor lux sensors in blueprint lux fields.** Use indoor sensors only.
6. **Never add automations to "fix" what the blueprint already handles internally.** Bypass switches handle overrides. Whole-home scenes are reserved for Good Night and Away.
7. **Never assume `input_boolean.input_boolean_good_night` is an error** — the double-prefix is the actual entity ID.
8. **Never add time-gated automations to the Good Morning retrigger.** Blacky's blueprint handles their morning transition via its internal time trigger. Only state_control-gated automations with no time fallback need the retrigger.
9. **Never add a bare time-triggered automation without an away/vacation gate.** Bedroom Wake Up and Sitting Room Lamps both ran away with an empty house because a `time` trigger fires whether or not anyone is home, and neither had an off step. If an automation turns lights on by clock, it needs either a gate or its own off step.
10. **Never assume `input_boolean.vacation_mode` has a double prefix.** `good_night` and `away_mode` do; `vacation_mode` does not.

---

## Current state

### Working
- Main Overhead Dynamic (living room overheads, Option 2 inverted lux)
- Main Lamps (living room lamps, daytime instance)
- Lamps Nightlight (living room lamps, nightlight instance gated on `good_night`)
- Sitting Room Lamps (morning + arrive on, goodnight + away off)
- Chandelier | Day
- Kitchen Overhead
- Kitchen Nightlight
- Gym lights (HA stock motion_light blueprint)
- Good Morning - 5AM
- Bedroom Wake Up - Weekdays
- Hallway Overhead (custom YAML dim-standby — standby/boost: 20%/60% day, 5%/30% evening, off/1% night, off/30% vacation; not a Blacky instance, has its own time triggers so NOT in the Good Morning retrigger)

### Written but not yet verified on hardware
- Vacation mode (5 automations + 3 gating fixes, committed 2026-09-04). **Requires `input_boolean.vacation_mode` to be created in the UI first** — the automations reference an entity that does not exist yet and will error until it does. Exterior light entity IDs need a live check.

### Known active issues
| # | Issue | Status |
|---|---|---|
| 1 | Six Leviton dimmers went unavailable after a power surge (same Matter commissioning batch) | Resolution: verify WiFi via Deco app, breaker cycle if offline; Matter re-interview not in UI |
| 2 | Kitchen Overhead still uses the living room FP2's **lux** sensor for dynamic lighting (motion now comes from `binary_sensor.kitchen_motion`) | Needs second FP2 in kitchen |
| 3 | Existing retrigger targets `automation.main_lamps_dynamic_n` — verify this entity_id still resolves (legacy from rename?) | Needs verification |
| 4 | `input_boolean.vacation_mode` does not exist yet — create as Helper → Toggle named `Vacation Mode`, then confirm the entity ID has no double prefix | Blocks vacation mode |
| 5 | Exterior light entity IDs (`light.front_porch`, `back_porch`, `front_flood`, `back_flood`) sourced from `scenes.yaml`, never verified against the live instance | Needs verification |

### Deferred
- Den configuration
- Bedroom per-room override switches
- "Refresh Lights" Apple Home button (capstone after rooms are complete)
- Bare-metal HA migration (off VirtualBox)
- Camera integration with Frigate NVR
- Locks, thermostats, additional sensors

---

## Working style and preferences

### How Leo likes to work
- **Spec first, YAML second.** Confirm full requirements before writing YAML.
- **YAML over visual UI** for complex automations.
- **Single-purpose automations.** No overengineering, no unnecessary toggle helpers or kill switches.
- **Manual light control is the intended override** — tap dimmer or voice. Not software kill switches.
- **Human-centered solutions over technically elegant but friction-heavy ones** (e.g., bedroom motion gate for sleeping-in, not alarm webhooks).

### Communication preferences
- **Don't re-explain established architecture.** It's all in this file.
- **Don't re-check already-confirmed settings.** Move to diagnostic steps.
- **Don't suggest steps already taken.** Search past chats first.
- **Tables > prose** for blueprint settings.
- **Brief responses, focused answers.** No long preambles.

### Debugging method
- Trace files are the primary tool. Leo uploads JSON traces.
- Parse with inline scripts on `data['trace']['trace']` — extract `result` booleans, `params.service_data.brightness_pct`, `changed_variables`.
- Filter out `context` to reduce noise.
- Diagnostic info typically at the end of trace files.

---

## Backup hygiene

- **Git protects YAML.** This repo.
- **Git does NOT protect Matter commissioning.** That's in `.storage/`, which is excluded from the repo (correctly — it has secrets).
- **Google Drive Backup add-on** is the recommended off-machine backup for the `.storage/` directory. Not yet installed.
- **VirtualBox snapshots** are an additional recovery layer.

---

## Tools and resources reference

- **Blacky's Sensor Light blueprint:** `Blackshome/sensor-light.yaml`
- **Stock HA motion_light blueprint** (gym only): `homeassistant/motion_light.yaml`
- **Repo:** `https://github.com/hleorubin-blip/171-lelawood`
- **Remote access (evaluated):** Tailscale, RustDesk, Chrome Remote Desktop, RDP

---

## Roadmap (in rough order)

1. Create `input_boolean.vacation_mode` helper and verify exterior entity IDs, then test vacation mode
2. Verify `automation.main_lamps_dynamic_n` still resolves; rename if needed
3. Den configuration (last main room)
4. Second Aqara FP2 for kitchen
5. Bedroom per-room override switches
6. "Refresh Lights" Apple Home button
7. Bare-metal HA migration
8. Cameras / Frigate
9. Locks, thermostats, additional sensors

---

*Last updated: September 4, 2026 — vacation mode built (5 automations); Bedroom Wake Up, Sitting Room Lamps and Hallway Overhead gated on away/vacation; exterior lights documented.*