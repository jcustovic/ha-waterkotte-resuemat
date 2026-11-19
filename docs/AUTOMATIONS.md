# Home Assistant Automations & Helpers (Waterkotte)

HA-side logic built on top of the Node-RED read/write entities. These live in HA (not Node-RED)
and drive the controller indirectly by setting the `input_datetime.ww_zeit_ein/aus` helpers, which
the Node-RED write chain then transmits to the controller.

---

## WW Boost (automatic warm-water heating outside the normal window)

**Purpose:** the normal WW release window is **13:00–21:30** (evening showers). If someone showers
at an unusual time and `WW Ist` drops too low *outside* that window, temporarily widen the window so
the heat pump heats now; restore the original times once hot water recovers.

### Entities

| Entity | Role |
|---|---|
| `sensor.heizung_ww_ist_2` | WW Ist (monitored temperature) |
| `sensor.heizungsolltemperatur_2` | WW Soll (used for the dynamic restore threshold) |
| `input_boolean.warmwasseronoff` | WW enabled gate |
| `input_datetime.ww_zeit_ein` / `ww_zeit_aus` | the live release window (writes to controller via Node-RED) |
| `input_boolean.ww_boost_aktiv` | boost-in-progress flag (internal) |
| `input_datetime.ww_zeit_ein_gespeichert` / `ww_zeit_aus_gespeichert` | saved originals (time-only) |
| `input_datetime.ww_boost_gestartet` | timestamp when boost started (date+time) |
| `binary_sensor.ww_boost_lauft` | **read-only mirror** of `ww_boost_aktiv` for the dashboard (template helper) |

### `automation.ww_boost_start`
- **Trigger:** `sensor.heizung_ww_ist_2` **< 35 °C** for 2 min.
- **Conditions (all):** `ww_boost_aktiv` off; `warmwasseronoff` on; **now is OUTSIDE the window**
  (`now < ein OR now > aus`, lexicographic `HH:MM:SS` compare). If inside the window → do nothing
  (the controller already heats).
- **Actions:** save ein/aus → `*_gespeichert`; stamp `ww_boost_gestartet = now`; set `ww_boost_aktiv` on; then branch:
  - `now < ein` → `ww_zeit_ein = max(00:00:00, now − 5 min)`
  - else (`now > aus`) → `ww_zeit_aus = min(23:59:00, now + 30 min)`

### `automation.ww_boost_ende`
- **Trigger (either):** template `WW Ist ≥ (WW Soll − 3)` for 1 min; **OR** `ww_boost_aktiv` on for 2 h (safety).
- **Condition:** `ww_boost_aktiv` on.
- **Actions:** restore `ww_zeit_ein/aus` from the `*_gespeichert` helpers; set `ww_boost_aktiv` off.

### Design notes / validated edges
- Lexicographic time compare is correct for zero-padded `HH:MM:SS`. Validated cases:
  07:00 & 11:45 → outside (EIN−5 branch); 13:00/15:00/21:30 → inside (no action);
  22:30 → outside (AUS+30); 00:30 → outside, treated as morning (EIN−5).
- Caps: `ein` clamped to not wrap before 00:00; `aus` capped at 23:59 (no midnight rollover).
- Restore threshold is **dynamic** (`Soll − 3`), so changing WW Soll updates it automatically.
- The automation never talks to the controller directly — it only sets `input_datetime` helpers;
  the Node-RED write chain (+ its 15 s read-back guard) does the transmission.
- `mode: single` on both; `ww_boost_aktiv` flag prevents re-triggering while a boost runs.

### Dashboard
In the **Erweitert** section, card "WW Boost (automatisch)" shows `binary_sensor.ww_boost_lauft`
("Boost läuft") as a plain entities row — current state + click-for-history, read-only (behaves like
"Heizung läuft"). The internal `input_boolean`/`input_datetime` helpers are NOT exposed as editable
controls.

### The read-only mirror (`binary_sensor.ww_boost_lauft`)
`input_boolean` renders an editable toggle on a dashboard; to display boost state read-only with
history (like a real binary_sensor) a **template binary_sensor** mirrors it:
```
state: {{ is_state('input_boolean.ww_boost_aktiv', 'on') }}
```
Created live via the **template helper config-entry flow** (`ha_config_set_helper helper_type=template`,
`config.next_step_id=binary_sensor`) — **no configuration.yaml edit, no HA restart**.

---

## Creating helpers / template sensors live (no restart) — reference

Via the `@ha` MCP `ha_config_set_helper` (needs the BPS key, see your HA infrastructure notes):
- `input_boolean`, `input_datetime` (`config.has_date`/`has_time`), `input_number`, etc.
- **template** helper: `helper_type=template` + `config.next_step_id=<binary_sensor|sensor|...>` + `config.state=<jinja>`.
These create config-entry (UI) helpers that are live immediately — no restart, unlike
`configuration.yaml` template platforms which need a reload/restart.
