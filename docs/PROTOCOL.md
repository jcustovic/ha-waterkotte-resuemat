# Resumat CD 4 — Wire Protocol

Everything below is reconstructed from the working Node-RED flow and verified against a live
read on 2026-11-14. Reference implementation:
https://github.com/mwllgr/fhem-waterkotte-resuemat-cd4

Transport: raw TCP to `<gateway-ip>:3333` (serial-to-TCP gateway). All multi-byte numeric
values are **little-endian** unless noted. Framing uses DLE (`0x10`) / STX (`0x02`) / ETX (`0x03`).

---

## 1. Frame structure

```
10 02 <cmd> <args...> 10 03 <crcHi> <crcLo>
└DLE STX┘            └DLE ETX┘ └─ CRC-16/BUYPASS ─┘
```

- `0x10 0x02` = start of frame (DLE + STX)
- `0x10 0x03` = end of frame (DLE + ETX)
- The trailing 2 bytes are a **CRC-16/BUYPASS** (big-endian) over the payload.

### Byte-stuffing (important)
Because `0x10` is the DLE control byte, any literal `0x10` inside the data is **doubled**
(`0x10 0x10`) on the wire. On receive, the flow collapses doubled `0x10 0x10` back to a single
`0x10` **before** parsing (`doppel 10 durch eine ersetzen` function). Always de-stuff before
decoding offsets, and account for stuffing when hand-building commands containing `0x10` data bytes.

---

## 2. Read commands

Command byte `0x15` = "read a block". Args: `<startHi> <startLo> <lenHi> <lenLo>` — read `len`
bytes starting at absolute address `start`.

The flow issues two reads every 2 minutes:

| Name | Bytes | start | len | Response (de-stuffed) |
|---|---|---|---|---|
| **Read Part 1** | `10 02 01 15 00 00 00 F3 10 03 7C 32` | `0x0000` | `0xF3` (243) | ~253 bytes → **parser #1** (len > 250) |
| **Read Part 2** | `10 02 01 15 00 F3 00 83 10 03 F1 ED` | `0x00F3` | `0x83` (131) | **141 bytes** → **parser #2** (len == 141) |

> Note the `01` after `10 02` (seen in both reads and writes) — a fixed sub-address/sequence byte.

### Response routing by length
After de-stuffing, a `switch` on `msg.payload.length` routes the buffer:
- `> 250`  → parser #1 ("Offset=Adresse plus 5", the big 0x0000-block)
- `== 141` → parser #2 (the 0x00F3-block — contains the 04.xx warm-water fields)
- `< 70`   → discarded (ack/short frames)

### Offset rule (how address → buffer index)
Each parser's field `offset` = **(absolute address − block start address) + 5**.
The `+5` is the 5-byte response header (`FF 10 02 00 17` observed at the start of responses).

- Parser #1 block start = `0x0000` → `offset = addr + 5`
- Parser #2 block start = `0x00F3` → `offset = (addr − 0x00F3) + 5`

Worked example (parser #2):
- 04.00 Abschaltung `0x0134` → `(0x0134−0x00F3)+5 = 65+5 = 70`
- 04.01 Zeit ein    `0x0135` → `(0x0135−0x00F3)+5 = 66+5 = 71`
- 04.02 Zeit aus    `0x0138` → `(0x0138−0x00F3)+5 = 69+5 = 74`

---

## 3. Write commands

Command byte `0x13` = "write". Args: `<addrHi> <addrLo> <value...>` then end-frame + CRC.
CRC is computed over `command.slice(2, N)` (from after `10 02` up to before `10 03`).

### 3a. Set a target temperature (float, 4 bytes LE)
```
10 02 01 13 <addrHi> <addrLo> <float32 LE (4 bytes)> 10 03 <crcHi> <crcLo>
```
CRC over bytes [2..10). Address per function:

| Function node | Address | Meaning |
|---|---|---|
| `Heizung Temperatur und CRC zu Befehl` | `0x00F8` | Heating return setpoint (02.02) |
| `WW Temperatur und CRC zu Befehl` | `0x013B` | Warmwasser Soll (04.04) |
| `Kuehlung Temperatur und CRC zu Befehl` | `0x0128` | Cooling setpoint (03.02) |

```js
// value encoding
let buffer = Buffer.alloc(4);
buffer.writeFloatLE(temperature, 0);
```

### 3b. Turn a function on/off (1 byte: 0x00 = on, 0x01 = off)
These are literal pre-computed frames (`change` nodes). Note the inverted logic:
**00 = ein (on), 01 = aus (off / Abschaltung)**.

| Node | Bytes | Address | Effect |
|---|---|---|---|
| `WW ein`  | `10 02 01 13 01 34 00 10 03 44 69` | `0x0134` ← 04.00 | WW Abschaltung = 0 → **on** |
| `WW aus`  | `10 02 01 13 01 34 01 10 03 C4 6C` | `0x0134` ← 04.00 | WW Abschaltung = 1 → **off** |
| `Heizung ein/aus` | (addr `0x0123` ← 03.00/02.00 family) | | heating on/off |
| `Kuehlung ein/aus` | (addr per flow) | | cooling on/off |

### 3b-time. Set a 3-byte time value (04.01 / 04.02 WW Zeit ein/aus) — VERIFIED
```
10 02 01 13 <addrHi> <addrLo> <ss> <mm> <hh> 10 03 <crcHi> <crcLo>
```
- Addresses: `0x0135` (04.01 Zeit ein), `0x0138` (04.02 Zeit aus)
- Value: 3 bytes **`[ss, mm, hh]`** (same order as the datetime encoding, time part only)
- **CRC over `slice(2, 9)`** (payload `01 13 addrHi addrLo ss mm hh`) — note this is a shorter
  slice than the 4-byte-float temperature writes (`slice(2, 10)`).
- **Byte-stuffing matters here:** a float value rarely contains `0x10`, but a time byte can
  (ss/mm = 16, or hh in 16:00–16:59). Any value byte `0x10` must be **doubled** on the wire.
  The CRC is computed over the **logical (unstuffed)** payload; stuffing is applied only for transport.

Verified end-to-end 2026-11-14: wrote `06:50:01` to 0x0135 via the flow → controller read back
`06:50:01`; restored to `06:50:00` → read back `06:50:00`. Example frames:
```
ein 06:50:00 -> 10 02 01 13 01 35 00 32 06 10 03 A1 67
aus 21:45:00 -> 10 02 01 13 01 38 00 2D 15 10 03 87 0E
```

### 3c. Sync the clock (6-byte datetime)
Command byte `0x14`, address `0x0000`, then 6 datetime bytes:
```
10 02 01 14 00 00 <ss mm hh dd MM yy> 10 03 <crcHi> <crcLo>
```
CRC over bytes [2..12).

---

## 4. Value encodings

| Encoding | Bytes | Notes |
|---|---|---|
| **float LE** (`floatle`) | 4 | IEEE-754 little-endian. Temps, pressures, hours. |
| **8bit / byte** | 1 | flags, small ints |
| **int16be** | 2 | e.g. Version |
| **3-byte time** | 3 | `[ss, mm, hh]` → format as `HH:MM:SS`. Used by 04.01/04.02 „Zeit ein/aus", 05.00 Uhrzeit. |
| **6-byte datetime** | 6 | `[ss, mm, hh, dd, MM, yy]`; year is `2000 + yy`. Used by 05.00/05.01 & clock sync. |

### Time / datetime byte order (verified)
From `parseDateTimeFromBytes` (read) and `convertDateTimeToBytes` (write):
```
byte[0] = seconds
byte[1] = minutes
byte[2] = hours
byte[3] = day        (datetime only)
byte[4] = month      (datetime only)
byte[5] = year-2000  (datetime only)
```
`parseDateTimeFromBytes`:
```js
new Date(2000 + b[5], b[4]-1, b[3], b[2], b[1], b[0]);  // (yr, monIdx, day, hh, mm, ss)
```
For a **3-byte** time field, only `b[0..2]` exist → `HH:MM:SS = pad(b[2]):pad(b[1]):pad(b[0])`.

> ⚠️ Do **not** reuse `parseDateTimeFromBytes` for a 3-byte time — it reads b[3..5] (day/month/year)
> which are out of range and yield a garbage Date. Use a dedicated 3-byte formatter.

---

## 5. CRC-16/BUYPASS

Polynomial `0x8005`, init `0x0000`, no reflection, no final XOR; output big-endian.

```js
function crc16Bypass(data) {
    let crc = 0x0000;
    const poly = 0x8005;
    for (let i = 0; i < data.length; i++) {
        crc ^= (data[i] << 8);
        for (let j = 0; j < 8; j++) {
            crc = (crc & 0x8000) ? ((crc << 1) ^ poly) : (crc << 1);
        }
    }
    return crc & 0xFFFF;
}
// writeUInt16BE(crc) appended as the last 2 frame bytes.
```
CRC input range: for reads/writes the flow computes over `command.slice(2, N)` where `N` is the
index of the closing `10 03` (i.e. the payload between STX and ETX, excluding the frame markers).

---

## 6. Bit-field decode tables (from the flow's "Notes")

Use these to decode the byte flags into human-readable status (not yet all surfaced as entities):

```
Mode-Heizung    => [UnterbrFuehlerfehler, KeinBedarf, Unterdrueckt, Zeitprog, Sommer, SchnellAufhz, ExtAnheb, Normal]
Mode-Wasser     => ["", "", "", Unterdrueckt, UnterbrFuehlerfehler, KeinBedarf, Zeitprog, Normal]
Betriebszustaende => [Ext. Steuerung, "", "", Unterbrechung, Hand-Betrieb, St2-Betrieb, Hz-Betrieb, Ww-Betrieb]
Do-Buffer       => [Pumpe-Quelle, Pumpe-Ww, Pumpe-Hz, St2, Kurbelwannenhz, Alarm, Kompr-1, Magnetventil]
Di-Buffer       => [Ext-Abschaltung, Ext-Sollwertbeeinflussung, "", Sole-Minimum, Pumpe-Quelle, HD-Pressostat, ND-Pressostat, Oeldruck/Kompr-Störung]
```

> **⚠️ Bit order = MSB-first.** The reference decodes these with `unpack 'B8'` and indexes the name
> array reversed, so `name[0]` = high bit `0x80` and `name[7]` = low bit `0x01`; i.e.
> `mask(name[i]) = 1 << (7-i)`. Verified live: DO=0xE3 during a WW cycle → Pumpe-Quelle/Ww/Hz +
> Kompr-1 + Magnetventil on, Alarm off; BZ=0x01 → Ww-Betrieb. See `STATUS-BUFFERS.md` for the full
> per-bit entity map, the live validation, and the DI safety-bit caveat.

> **⚠️ Firmware-specific addresses.** This controller is **Resümat CD 4 v8126**. The reference
> implementation targets a different firmware where these buffers are under **Menu 8** at
> `0x097/0x098`; on 8126 they are **Menu 9 Statusmelder** (`0x00CE/0x00CF/0x00D0`). The bit NAMES and
> ORDER carry over, the ADDRESSES do not.

---

## 7. Replaying a read (debug/dev)

A read-only probe (connect, send the 12-byte read command, de-stuff, inspect offsets) can be run
from the HA host, which can reach the gateway. Pattern used on 2026-11-14:

```python
import socket
PART2 = bytes([0x10,0x02,0x01,0x15,0x00,0xF3,0x00,0x83,0x10,0x03,0xF1,0xED])
s = socket.socket(); s.settimeout(5); s.connect(("<gateway-ip>", 3333))
s.sendall(PART2); buf = s.recv(4096); s.close()
# de-stuff 0x10 0x10 -> 0x10, then read offsets 70 (04.00), 71-73 (04.01), 74-76 (04.02)
```
Observed PART2 response: exactly **141 bytes**, offset70=0, offset71-73 = `00 32 06` → 06:50:00,
offset74-76 = `00 2D 15` → 21:45:00.

> The gateway may accept one connection at a time and the flow polls every 2 min — keep probes
> short and close the socket. This is **read-only**; never send `0x13`/`0x14` write frames from a
> probe unless you intend to change the controller.
