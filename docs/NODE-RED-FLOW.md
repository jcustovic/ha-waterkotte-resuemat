# Node-RED Flow — "Waterkotte Wärmepumpe"

- Instance: `http://<ha-host>:1880/` (Node-RED **5.0.8**)
- Flow tab: **"Waterkotte Wärmepumpe"**, id **`f6f2187d.f17ca8`**, ~88 nodes, enabled
- Admin API is open on the host loopback (no auth) → reachable via SSH: `ssh ha` then `curl http://localhost:1880/...`
  (see your HA infrastructure notes for the SSH key gotcha; use `-i <ssh-key>`).

Backups of the flow JSON are in `flow-backup/`.

---

## Data paths

### READ (every 2 min)
```
inject "Every 2min" (repeat 120s)
  └─> change "Read Part 1"  (payload = 12-byte read cmd, bin)   ─┐
  └─> change "Read Part 2"  (payload = 12-byte read cmd, bin)   ─┤
                                                                 ▼
                                             delay ─> tcp request "query data TCP"
                                                       (<gateway-ip>:3333, ret=buffer, out=time, splitc=5000)
                                                                 ▼
                                        function "doppel 10 durch eine ersetzen" (de-stuff 0x10 0x10)
                                                                 ▼
                                        switch on msg.payload.length:
                                          • > 250  ─> buffer-parser "Offset=Adresse plus 5"  (parser #1, base 0x0000)
                                          • == 141 ─> buffer-parser (unnamed)                (parser #2, base 0x00F3)
                                          • < 70   ─> (drop)
                                                                 ▼
                                        each parser output ─> ha-sensor / ha-binary-sensor entity
```

### WRITE (user action in HA → command to controller)
```
HA button/number ─> server-state-changed ─> (switch/change) ─> function "<X> und CRC zu Befehl"
   (builds 10 02 01 13 <addr> <value> 10 03 <crc>) ─> tcp request ─> controller
```
Write builders:
- `WW Temperatur und CRC zu Befehl` → addr `0x013B` (Warmwasser Soll), float32 LE
- `Heizung Temperatur und CRC zu Befehl` → addr `0x00F8`, float32 LE
- `Kuehlung Temperatur und CRC zu Befehl` → addr `0x0128`, float32 LE
- `WW ein`/`WW aus` (literal frames) → addr `0x0134` (04.00), 1 byte, **00=on / 01=off**
- `Ziet synchronisieren und CRC zu Befehl` → clock sync (cmd `0x14`, 6-byte datetime)

Debounce: `Last <X> Temp NOT set within 15s` switches + `Set global variable if <X> Temp state changed`
server-state-changed nodes guard against spamming the controller while a slider is moving.

### Full write/read-back parity pattern (studied from the temperature setters)

Each writable value uses **two** directions plus an anti-clobber guard. Using WW Soll as the model:

**Write (HA → pump):**
```
input_number.warmwasser_temp
  → server-state-changed (for: 10s, outputOnlyOnStateChange, payload=entityState)
  → "WW Temperatur und CRC zu Befehl"  (build frame + CRC)
  → shared delay → tcp request
```

**Read-back (pump → HA), with 15s write-priority guard:**
```
tcp → de-stuff → switch(len) → parser → ha-sensor "WarmwasserSoll"
  → switch "<X> ts set?" (global.<x>_last_timestamp  nnull? / null?)
        • null  → api-call-service set_value   (push immediately, first time)
        • nnull → switch "Last <X> NOT set within 15s"
                     (passes only if  global.<x>_last_timestamp < now-15s)
                     → api-call-service set_value
```
A separate `Set global variable if <X> state changed` (server-state-changed, for: 1s) stamps
`global.<x>_last_timestamp` whenever the user edits the control, so a poll can't overwrite a
fresh user edit for 15 s.

**WW Zeit ein/aus write (04.01/04.02) — implemented 2026-11-14 with full parity:**
- Helpers: `input_datetime.ww_zeit_ein` / `ww_zeit_aus` (`has_time:true, has_date:false`).
- Write builders: `WW Zeit ein -> Befehl (0x0135)` / `aus (0x0138)` — parse `HH:MM:SS` → `[ss,mm,hh]`,
  stuff `0x10`, frame, CRC `slice(2,9)` → shared delay → tcp.
- Stamp nodes → globals `ww_zeit_ein_last_timestamp` / `ww_zeit_aus_last_timestamp`.
- Read-back taps the formatter output (`HH:MM:SS`) → `ts set?` switch → `15s` switch →
  `api-call-service input_datetime.set_datetime {"time": msg.payload}`.
- Node ids: `wkWein_*` / `wkWaus_*`. Import source: `add-write-04.01-04.02-import.json`.

---

## Parser #1 — "Offset=Adresse plus 5" (response length > 250, base 0x0000)

`offset = addr + 5`. Output index → field:

| out | offset | addr | type | field | wired? |
|----|----|----|----|----|----|
| 0 | 5 | 0x0000 | int16be | Version | ✓ |
| 1 | 13 | 0x0008 | floatle | Temp-Aussen | ✓ |
| 2 | 17 | 0x000C | floatle | Temp-Aussen24h | ✓ |
| 3 | 37 | 0x0020 | floatle | Temp-WW Soll | ✓ |
| 4 | 41 | 0x0024 | floatle | Temp-WW Ist | ✓ |
| 5 | 53 | 0x0030 | floatle | Temp-WQuelle-Ein | ✓ |
| 6 | 57 | 0x0034 | floatle | Temp-WQuelle-Aus | ✓ |
| 7 | 25 | 0x0014 | floatle | Temp-Ruecklauf Soll | ✓ |
| 8 | 29 | 0x0018 | floatle | Temp-Ruecklauf Ist | ✓ |
| 9 | 111 | 0x006A | floatle | Betriebsstunden Kompressor | ✓ |
| 10 | 115 | 0x006E | floatle | Betriebsstunden Heizung | ✓ |
| 11 | 119 | 0x0072 | floatle | Betriebsstunden WW | ✓ |
| 12 | 228 | 0x00DF | 8bit | Heizungläuft (09.10 Mode Heizung) | ✓ |
| 13 | 230 | 0x00E1 | 8bit | WWläuft (09.12 Mode Warmwasser) | ✓ |
| 14 | 33 | 0x001C | floatle | Temp-Vorlauf Ist | ✓ |
| 15 | 123 | 0x0076 | floatle | Betriebsstunden Stufe 2 | ✓ |
| 16 | 229 | 0x00E0 | 8bit | Kuehlungläuft (09.11 Mode Kuehlung) | ✓ |
| 17 | 105 | 0x0064 | byte×6 | DatumUhrzeit (05.00/05.01) | ✓ |
| 18 | 61 | 0x0038 | floatle | Temp-Verdampfer | ✓ |
| 19 | 65 | 0x003C | floatle | Temp-Kondensation | ✓ |
| 20 | 69 | 0x0040 | floatle | Temp-Saugleitung | ✓ |
| 21 | 73 | 0x0044 | floatle | Temp-frei | — unwired |
| 22 | 77 | 0x0048 | floatle | Druck-Verdampfer | ✓ |
| 23 | 81 | 0x004C | floatle | Druck-Kondensator | ✓ |

## Parser #2 — unnamed (response length == 141, base 0x00F3)

`offset = (addr − 0x00F3) + 5`. Output index → field:

| out | offset | addr | type | field | wired? |
|----|----|----|----|----|----|
| 0 | 5 | 0x00F3 | byte | HZAbschaltung (02.00) | ✓ |
| 1 | 70 | 0x0134 | byte | WWAbschaltung (04.00) | ✓ |
| 2 | 10 | 0x00F8 | floatle | HzSollTemp (02.02) | ✓ |
| 3 | 6 | 0x00F4 | floatle | HzEinsatz (02.01) | ✓ |
| 4 | 77 | 0x013B | floatle | Warmwasser Soll (04.04) | ✓ |
| 5 | 53 | 0x0123 | byte | KlgAbschaltung (03.00) | ✓ |
| **6** | **71** | **0x0135** | **byte (len 2!)** | **WWZeitEin (04.01)** | **— unwired** |
| **7** | **74** | **0x0138** | **byte (len 3)** | **WWZeitAus (04.02)** | **— unwired** |
| 8 | 54 | 0x0124 | floatle | KlgEinsatz (03.01) | ✓ |
| 9 | 58 | 0x0128 | floatle | KlgSollTemp (03.02) | ✓ |

> ⚠️ As found, `WWZeitEin` (out6) has **length 2** and `WWZeitAus` (out7) **length 3** — both
> should be **length 3** (`[ss, mm, hh]`) per the Adressliste (04.01/04.02 = 0x0003). Fix the
> length when wiring them up (see below).

---

## How to add a new field (worked example: 04.01 / 04.02 WW Zeit ein/aus)

**Goal:** expose warm-water allowed-from / allowed-until times as HA sensors.

These are already in parser #2 (outputs 6 and 7) but (a) have wrong lengths and (b) go nowhere.

**Steps**
1. **Fix the parser spec items** in parser #2 (node id `7df376b8e6a732d1`):
   - `WWZeitEin` (item index 6): `length` → **3** (offset 71 unchanged)
   - `WWZeitAus` (item index 7): `length` → **3** (offset 74 unchanged)
   Each now emits a 3-byte array `[ss, mm, hh]`.

2. **Add a 3-byte time formatter** function node (do NOT reuse `parseDateTimeFromBytes`):
   ```js
   // WW Zeit (3 bytes [ss,mm,hh]) -> "HH:MM:SS"
   const b = msg.payload;
   const p = v => String(v).padStart(2,'0');
   msg.payload = `${p(b[2])}:${p(b[1])}:${p(b[0])}`;
   return msg;
   ```
   Use **two copies** (one per output) so the two values never clobber each other.

3. **Add two `ha-sensor` nodes** (e.g. `WW Zeit ein`, `WW Zeit aus`). Wire:
   - parser #2 **out6** → formatter → `WW Zeit ein`
   - parser #2 **out7** → formatter → `WW Zeit aus`

4. **Deploy** via the Admin API (`POST /flows` with the full modified flow) or via the editor's
   Import. Always snapshot `flow-backup/` first. After deploy, reload the editor and confirm the
   two new entities appear in HA.

**Verified expected values (2026-11-14 live read):** Zeit ein `06:50:00`, Zeit aus `21:45:00`,
04.00 Abschaltung `0` (WW enabled).

### General recipe for ANY new read field
1. Find the address + byte count in `ADDRESSES.md`.
2. Decide which read block it's in (Part 1 → parser #1, Part 2 → parser #2) — see the block ranges there.
3. Compute `offset = (addr − block_start) + 5`.
4. Add a spec item to that parser (`type` = `floatle` for 4-byte temps/hours, `byte` len 1 for flags,
   `byte` len 3 for a time, `byte` len 6 for a datetime).
5. Wire the new output → (optional formatter/scale) → a new `ha-sensor`/`ha-binary-sensor`.
6. **Validate with a live read** (`PROTOCOL.md` §7) before trusting the value.

### Adding a WRITE for a field
Clone one of the `… und CRC zu Befehl` functions, change the target address bytes and the value
encoding, keep the `crc16Bypass` + frame markers. Gate it behind a debounce like the existing
temperature setters. **Test carefully** — writes change the controller.

---

## Admin API cheatsheet (read-only ops are safe)

```bash
# list flows (full) / just this tab
ssh -i <ssh-key> <user>@<ha-host> "curl -s http://localhost:1880/flows"
ssh -i <ssh-key> <user>@<ha-host> "curl -s http://localhost:1880/flow/f6f2187d.f17ca8"

# Node-RED version / settings
ssh -i <ssh-key> <user>@<ha-host> "curl -s http://localhost:1880/settings"
```
Deploying a change is `POST /flows` (full) or `PUT /flow/f6f2187d.f17ca8` (single tab) with a
`Node-RED-Deployment-Type` header. Back up first.

Programmatic deploy pattern used here (single tab, safest):
```bash
# 1. backup
ssh ... "curl -s http://localhost:1880/flows" > flow-backup/nodered_full_flows_<TS>.json
# 2. edit the tab JSON locally (merge new nodes, keep all existing), then:
scp local_merged.json ha:/tmp/wk.json
ssh ... "curl -s -X PUT http://localhost:1880/flow/f6f2187d.f17ca8 \
  -H 'Content-Type: application/json' -H 'Node-RED-Deployment-Type: nodes' -d @/tmp/wk.json"
```

### GOTCHA — `api-call-service` v7 requires the `action` field (not just domain/service)
When building an `api-call-service` node by hand (e.g. the WW Zeit read-back setters), the
**HA node-red v7 node needs `"action": "<domain>.<service>"`** in addition to (or instead of)
`domain`/`service`. If `action` is missing you get a runtime
`ValidationError: "action" is required`, AND the editor renders the **entity-target field empty**
(because it derives the target picker from `action`). Fix: set
`"action": "input_datetime.set_datetime"` (and keep `entityId`, `data`, `dataType`). This bit the
WW Zeit read-back nodes on first deploy — corrected 2026-11-14.

### Byte-stuffing on writes (0x10)
Time/datetime value bytes equal to `0x10` must be **doubled** on the wire (the receiver de-stuffs).
Floats (temperatures) essentially never contain `0x10`, so the temp setters don't stuff; the
WW Zeit builders DO stuff the 3 value bytes. CRC is computed over the **logical (unstuffed)**
payload. Verified: known-good `WW ein` frame CRC `slice(2,7)` = `44 69` exact match; the JS
`crc16Bypass` (no in-loop masking, mask only at return) produces identical results to a fully-masked
reference across all 1440 time values.
