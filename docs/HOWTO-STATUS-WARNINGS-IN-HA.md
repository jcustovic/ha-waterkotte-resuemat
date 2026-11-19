# How the values, warnings and errors get into Home Assistant

End-to-end description of the chain for the **status / warning / interruption / failure** values and
the **hot water time window** (04.01 / 04.02). Everything below matches `resuemat_8126_flows.json`.

```
Controller ──TCP──> Node-RED ──buffer-parser──> ha-sensor (raw byte) ──> HA template binary_sensors ──> dashboard
 (Resümat CD4)     read every 2 min   one output per field   numeric 0..255        one bit = one sensor
```

The controller only delivers **whole bytes**. Node-RED publishes each byte as a plain numeric sensor;
Home Assistant splits the byte into individual on/off sensors with a template (`bitwise_and`).
No bit logic lives in Node-RED.

---

## 1. Node-RED: where the values are read

Two read requests run every 2 minutes (see `PROTOCOL.md`). The answer is de-stuffed and routed by length
to one of two `buffer-parser` nodes. Offset = (address − block start) + 5.

### Parser 1 "Offset=Adresse plus 5" (block start 0x0000) – new outputs

| Out | Field | Menu | Address | Offset | Type | ha-sensor node → entity |
|---|---|---|---|---|---|---|
| 24 | Betriebszustaende | 09.00 | 0x00CE | 211 | byte | `Betriebszustaende Byte` → `sensor.waterkotte_betriebszustaende_byte` |
| 25 | DO-Buffer | 09.01 | 0x00CF | 212 | byte | `DO-Buffer Byte` → `sensor.waterkotte_do_buffer_byte` |
| 26 | DI-Buffer | 09.02 | 0x00D0 | 213 | byte | `DI-Buffer Byte` → `sensor.waterkotte_di_buffer_byte` |
| 27 | Unterbrechungen | 08.00 | 0x00C3 | 200 | byte | `Unterbrechungen Byte` → `sensor.waterkotte_unterbrechungen_byte` |
| 28 | Warnung-Eingang | 08.01 | 0x00C4 | 201 | byte | `Warnung-Eingang Byte` → `sensor.waterkotte_warnung_eingang_byte` |
| 29 | Warnung-Ausgang | 08.02 | 0x00C5 | 202 | byte | `Warnung-Ausgang Byte` → `sensor.waterkotte_warnung_ausgang_byte` |
| 30 | Warnung-Sonstige | 08.03 | 0x00C6 | 203 | byte | `Warnung-Sonstige Byte` → `sensor.waterkotte_warnung_sonstige_byte` |
| 31 | Ausfall | 08.04 | 0x00C7 | 204 | byte | `Ausfall Byte` → `sensor.waterkotte_ausfall_byte` |

### Parser 2 (block start 0x00F3, response length 141) – new outputs

| Out | Field | Menu | Address | Offset | Len | Type | Target |
|---|---|---|---|---|---|---|---|
| 6 | WWZeitEin | 04.01 | 0x0135 | 71 | **3** | byte `[ss,mm,hh]` | function `Zeit ein -> HH:MM:SS` → `sensor.waterkotte_ww_zeit_ein` |
| 7 | WWZeitAus | 04.02 | 0x0138 | 74 | **3** | byte `[ss,mm,hh]` | function `Zeit aus -> HH:MM:SS` → `sensor.waterkotte_ww_zeit_aus` |
| 10 | DO-Handkanal | 06.18 | 0x016E | 128 | 1 | byte | `DO-Handkanal Byte` → `sensor.waterkotte_do_handkanal` |
| 11 | DO-Handkanal-Ein | 06.19 | 0x016F | 129 | 1 | byte | `DO-Handkanal-Ein Byte` → `sensor.waterkotte_do_handkanal_ein` |

Important details:
* The time fields must have **length 3**. The formatter turns `[ss,mm,hh]` into `HH:MM:SS`
  (do not reuse the 6-byte date/time parser, it would read garbage).
* The sensor entity ids come from the **entity config node name** (e.g. "Waterkotte DO-Buffer Byte"
  → `sensor.waterkotte_do_buffer_byte`). The Home Assistant templates below depend on exactly these ids.
  If you rename a config node, adjust the template file.
* A healthy system reports `0` for all Menü 8 bytes.

### Adding another field
1. Find address and length in `Adressliste.txt` / `docs/ADDRESSES.md`.
2. Part 1 block (0x0000–0x00F2) → parser 1, Part 2 block (0x00F3–0x0175) → parser 2.
3. `offset = (address − block start) + 5`; add a spec item, increase the parser's `outputs`.
4. Wire the new output to a new `ha-sensor` node with its own entity config node.
5. Deploy and compare with the controller display.

---

## 2. Home Assistant: turning bytes into warning/error sensors

File: `homeassistant/waterkotte_status_templates.yaml`

Include it in `configuration.yaml`:

```yaml
template: !include waterkotte_status_templates.yaml
```

(Or paste the entries below an existing `template:` section; restart HA or reload template entities.)

Each sensor is one bit of one byte:

```yaml
- name: "DO Kompressor 1"
  state: "{{ (states('sensor.waterkotte_do_buffer_byte') | int(0)) | bitwise_and(2) > 0 }}"
  device_class: running
```

`bitwise_and(mask) > 0` is true when the bit is set. Masks come from the controller manual
(photos in `docs/manual_*.jpg`); see `STATUS-BUFFERS.md` for every mask.

| Byte | Meaning | Sensors created | Polarity |
|---|---|---|---|
| 09.00 Betriebszustände | operating state | `bz_*` (6) | on = active |
| 09.01 DO-Buffer | relay outputs | `do_*` (7) | on = output on |
| 09.02 DI-Buffer | inputs / pressostats | `di_*` (7) | pressostats: **on = NOK (fault)**; others on = set |
| 08.00 Unterbrechungen | interruptions I01–I05 | `i01_*` … `i05_*` | on = active (problem) |
| 08.01 Warnung Eingang | warnings (W01–W04) | `w01_*` … `w04_*` | on = active (problem) |
| 08.02 Warnung Ausgang | warnings (W07–W09 …) | `warnaus_*` | on = active (problem) |
| 08.03 Warnung Sonstige | sensor faults etc. | `warnsonst_*` | on = active (problem) |
| 08.04 Ausfall | failures F01–F05 | `f01_*` … `f05_*`, `ausfall_vorhanden` | on = active (problem) |
| 06.18/06.19 Handkanal | manual relay override | `sensor.do_handkanal_auswahl`, `binary_sensor.do_handkanal_ein` | read only |

The **code legend** (what W04, I03, F02 … mean) is in `STATUS-BUFFERS.md` and the manual photos
(`manual_warncodes_W01-W17.jpg`, `manual_3.12_I-F-codes.jpg`).

### Confidence of the bit → code mapping
* Confirmed on the live system: DO / Betriebszustände bits during a hot water cycle, DI idle state,
  and **8.01 mask 16 = W04**.
* **Provisional**: the other W bits and all I and F bits (assigned in ascending order 1, 2, 4, 8, 16).
  Verify each one when it fires for the first time and correct the mask in the YAML if needed.

### Changing a mask
Edit the number in `bitwise_and(<mask>)` and reload template entities
(Developer tools → YAML → *Template entities*). No Node-RED change needed.

---

## 3. Dashboard

File: `homeassistant/dashboard_waterkotte.yaml` (YAML-mode dashboard).
Relevant sections:

* **Status (Buffer)** – DI-Buffer, DO-Buffer, Betriebszustände
* **Warnungen & Störungen** – Unterbrechungen (I01–I05), Ausfälle (F01–F05), Warnungen 8.01 / 8.02 / 8.03
* **Erweitert** – Warmwasser release window (Zeit ein / aus), Zeit synchronisieren, WW Boost status

Some entities in this file are named `sensor.nodered_<id>`; these ids are install specific, replace
them with your own entity ids.

Example for loading it (configuration.yaml):

```yaml
lovelace:
  dashboards:
    lovelace-waterkotte:
      mode: yaml
      title: Waterkotte
      icon: mdi:heat-pump
      filename: dashboards/waterkotte.yaml
```

---

## 4. Hot water release window (04.01 / 04.02)

Read: `sensor.waterkotte_ww_zeit_ein` / `sensor.waterkotte_ww_zeit_aus` (text `HH:MM:SS`).

Write: two `input_datetime` helpers (time only) are the controls:

* `input_datetime.ww_zeit_ein`, `input_datetime.ww_zeit_aus`
* The flow listens for changes (10 s), builds the frame (command `0x13`, address `0x0135` / `0x0138`,
  3 value bytes `[ss,mm,hh]`, byte-stuffing for `0x10`, CRC over the unstuffed payload) and sends it.
* The next read writes the controller value back into the helper, but only if the helper was **not edited
  in the last 15 s**, so a read never overwrites a fresh user edit.

The optional **WW Boost** automations (`homeassistant/automations/`) use these helpers; see `AUTOMATIONS.md`
for the helpers they need.

---

## 5. Quick verification checklist

1. Node-RED debug / HA developer tools: `sensor.waterkotte_*_byte` exist and are numbers (0 when healthy).
2. `sensor.waterkotte_ww_zeit_ein` / `_aus` show times that match the controller display (04.01 / 04.02).
3. With the pump heating hot water: `binary_sensor.do_kompressor_1`, `binary_sensor.do_pumpe_warmwasser`,
   `binary_sensor.bz_warmwasserbetrieb` are on.
4. All pressostat sensors (`di_*_stoerung`) are **off** on a healthy system.
5. Active controller warnings show up as `problem` entities under "Warnungen & Störungen".
