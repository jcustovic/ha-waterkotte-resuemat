
Integration to read values from Waterkotte Resümat CD4 (Version 8126) and have them available in Home Assistant (HA).

I am using this on Waterkotte AI1 5006.4 and it is connected via USB-Serial-Port Converter.

The logic is implemented via Node-Red so you will need to install the following modules on your HA installation:
* node-red-contrib-home-assistant-websocket
* node-red-contrib-buffer-parser
* node-red-node-serialport
* node-red-show-value


The Node-Red flow reads all the values every 2 min (but you can adjust) and you can control the following:
* Heating on/off (02.00 Handabschaltung)
* Cooling on/off (03.00 Abschaltung)
* Hot watter on/off (04.00 Abschaltung)
* Heating temperature (02.02 Hzg Ruecklauf Sollwert am Einsatzp.)
* Cooling temperature (03.02 Ruecklauftemp. Sollwert am Einsatzzeitp.)
* Hot watter temperature (04.04 Warmwasser Soll)
* Synchronize time and date
* Hot water release window: **04.01 Zeit ein / 04.02 Zeit aus** (read + write, via `input_datetime.ww_zeit_ein` / `ww_zeit_aus`)

Read-only diagnostics (decoded in Home Assistant, see below):
* Menü 9 status bytes: 09.00 Betriebszustände, 09.01 DO-Buffer, 09.02 DI-Buffer
* Menü 8 warnings / interruptions / failures: 8.00 (I01–I05), 8.01–8.03 (W01–W17), 8.04 (F01–F05)
* Menü 6 manual relay channel: 6.18 DO-Handkanal, 6.19 DO-Handkanal-Ein

How it looks like:

Node-Red:

<img width="2576" height="1158" alt="image" src="https://github.com/user-attachments/assets/e60b6fa0-83c5-4555-8a8c-e245453ca4e4" />

Home Assistant:

<img width="454" height="1150" alt="image" src="https://github.com/user-attachments/assets/dfe4b228-b4cf-4e25-abcf-7a442aedcf9e" />
<img width="451" height="384" alt="image" src="https://github.com/user-attachments/assets/a5154e80-98a4-440b-be5b-3f688fd51c08" />

--

## Repository layout

| Path | Contents |
|---|---|
| `resuemat_8126_flows.json` | The complete Node-RED flow export (import via Node-RED menu → Import) |
| `Adressliste.txt` | Controller address list |
| `homeassistant/waterkotte_status_templates.yaml` | Template `binary_sensor`s / `sensor` that decode the status, warning, failure and Handkanal bytes (`template: !include ...`) |
| `homeassistant/dashboard_waterkotte.yaml` | Example Lovelace dashboard (entity ids of the `nodered_<id>` sensors are install specific – adjust to yours) |
| `homeassistant/automations/` | WW Boost automations (widen the hot water window when WW Ist is too low outside the normal window) |
| `docs/HOWTO-STATUS-WARNINGS-IN-HA.md` | **Start here**: how the status / warning / error bytes and the hot water times travel from the controller through Node-RED into Home Assistant |
| `docs/` | Protocol, address map, status/warning bit decoding, Node-RED flow notes, entity list, automations, and photos of the manual pages the bit masks come from |

After importing the flow, set the TCP request node ("query data TCP") to the host/port of your serial gateway (port 3333) and select your Home Assistant server in the entity nodes.

NOTE: You are using this at your own risk.
