# Home Assistant Entities created by the flow

Current entity output nodes (`ha-sensor` / `ha-binary-sensor`) in the Waterkotte flow, 2026-11-14.

## Sensors (ha-sensor)
- Temperatur Aussen
- Temperatur Aussen 24h
- Temperatur WW Soll
- Temperatur WW Ist
- Temperatur WQuelle Ein
- Temperatur WQuelle Aus
- Temperatur Vorlauf Ist
- Temperatur Rücklauf Ist
- Temperatur Rücklauf Soll
- Temp Verdampfung
- Temp Kondensation
- Temp Saugleitung
- Druck Verdampfer
- Druck Kondensator
- Betriebsstunden Kompressor
- Betriebsstunden Heizung
- Betriebsstunden Warmwasser
- Betriebsstunden Stufe 2
- HeizungSollTemperatur
- HeizungEinsatzTemp
- WarmwasserSoll
- KlgSollTemperatur
- KlgEinsatzTemperatur
- DatumUhrzeit

## Binary sensors (ha-binary-sensor)
- WW läuft
- Heizung läuft
- Kuehlung läuft
- WWAbschaltung
- HeizungAbschaltung
- KlgAbschaltung

## Write / control (api-call-service + command builders)
- Warmwasser Button On / Off, Warmwasser Soll Set
- Heizung Button On / Off, Heizung Soll Set
- Kuehlung Button On / Off, Kuehlung Soll Set
- Zeit synchronisieren (clock sync)
- **WW Zeit ein / aus (04.01 / 04.02)** — writable via `input_datetime.ww_zeit_ein` /
  `input_datetime.ww_zeit_aus` (time pickers). Full write + read-back + 15s-guard parity. ✅ done 2026-11-14

## Read sensors (done)
- **WW Zeit ein** (04.01) → `sensor.waterkotte_ww_zeit_ein`
- **WW Zeit aus** (04.02) → `sensor.waterkotte_ww_zeit_aus`

## HA-side helpers & automations (not Node-RED — see AUTOMATIONS.md)
- `input_datetime.ww_zeit_ein` / `ww_zeit_aus` — writable release-window controls (feed Node-RED write chain)
- `input_boolean.ww_boost_aktiv` — WW Boost in-progress flag (internal)
- `input_datetime.ww_zeit_ein_gespeichert` / `ww_zeit_aus_gespeichert` — saved originals during a boost
- `input_datetime.ww_boost_gestartet` — boost start timestamp
- `binary_sensor.ww_boost_lauft` — read-only template mirror of the boost flag (dashboard/history)
- `automation.ww_boost_start` / `automation.ww_boost_ende` — the WW Boost logic

## Status / warning decode (see STATUS-BUFFERS.md)
- **Menü 9:** `sensor.waterkotte_{betriebszustaende,do_buffer,di_buffer}_byte` + 20 `binary_sensor.{bz_,do_,di_}*`
- **Menü 8:** `sensor.waterkotte_{unterbrechungen,warnung_eingang,warnung_ausgang,warnung_sonstige,ausfall}_byte`
  + `binary_sensor.{w01..w04, warnaus_, warnsonst_}*`
  + **I-codes** `binary_sensor.i0{1..5}_*` (8.00 interruptions, manual §3.12)
  + **F-codes** `binary_sensor.f0{1..5}_*` (8.04 failures, manual §3.12) and summary `binary_sensor.ausfall_vorhanden`
  - *Orphans (old FHEM 8.00 rows, superseded by I01–I05, safe to delete):*
    `binary_sensor.unterbr_{temp_hz_vorlauf_zu_hoch,ww_fuehler_defekt,hz_fuehler_defekt,wpumpe_fuehler_defekt,schalthaeufigkeit,ext_abschaltung,temp_qaus_zu_niedrig,ungueltiger_betriebs_mode}`

## Dashboard
`lovelace-waterkotte` (YAML mode, file `/config/dashboards/waterkotte.yaml`). Sections:
Status & Steuerung · Betriebsstatus · Temperaturen · Temperaturverlauf · Energie ·
Kältekreis (Diagnose) · Betriebsstunden · **Erweitert** (Warmwasser-Erweitert: Soll + WW Freigabe
ein/aus; Wartung: Zeit synchronisieren; WW Boost: `binary_sensor.ww_boost_lauft` read-only) ·
**Status (Buffer)** (DI/DO/Betriebszustände) · **Warnungen & Störungen** (5 cards: Unterbrechungen
I01–I05 · Ausfälle F01–F05 · Warnungen 8.01/8.02/8.03).
A local copy of the deployed YAML is in `flow-backup/`.
