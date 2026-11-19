# Status & Warning Bit Decoding (Menü 9: 09.00/09.01/09.02 + Menü 8: 8.00–8.04)

Three single-byte status registers (Menu 9 = **Statusmelder**), each a bitfield, decoded into Home
Assistant binary_sensors. Added + corrected 2026-11-14.

> **Firmware:** this controller is a **Resümat CD 4, version 8126**. Its address/menu layout differs
> from the public reference implementation (mwllgr FHEM module), which was written for a firmware
> where the buffers live under **Menu 8** at addresses `0x097/0x098`. On **8126** they are **Menu 9**
> (`09.01` DO-Buffer `0x00CF`, `09.02` DI-Buffer `0x00D0`, `09.00` Betriebszustände `0x00CE`).
> Menu 8 on 8126 is Warn/Unterbrechung/Ausfall (a *different* register). Trust the **bit-order
> convention and name tables** from the reference, but the **addresses are firmware-specific**.

## Source bytes (Node-RED)

Read in **Part 1** (parser #1 `d2f10b835ce942c0`, base 0x0000, `offset = addr + 5`). Exposed as plain
numeric sensors (`entity_category: diagnostic`):

| Register | Address | Parser offset | HA sensor |
|---|---|---|---|
| 09.00 Betriebszustände | 0x00CE | 211 | `sensor.waterkotte_betriebszustaende_byte` |
| 09.01 DO-Buffer (Ausgänge, Ein/Aus) | 0x00CF | 212 | `sensor.waterkotte_do_buffer_byte` |
| 09.02 DI-Buffer (Eingänge, OK/NOK & Ein/Aus) | 0x00D0 | 213 | `sensor.waterkotte_di_buffer_byte` |

## ⚠️ Bit order is MSB-first (do not use LSB-first)

**Authoritative source: the Waterkotte manual, Menü 9 Statusmeldungen (p.23)** — photographed in
`manual_menu8-9_statusmeldungen_p23.jpg` in this folder. The manual's "Dim" column gives the exact bit mask per row.
(The public FHEM reference's name arrays turned out to be WRONG for this firmware — it had an "Alarm"
and "Kurbelwannenhz" bit that don't exist here; always prefer the manual.)

The masks below are taken **directly from the manual**. `mask` = the single set bit for that signal.

### DO-Buffer (09.01, 0x00CF) — "Do-Buffer", Ein/Aus. From manual:
| name | mask (manual "Dim") | binary_sensor | device_class |
|---|---|---|---|
| Magnetventil | `00000001` = 1 | `binary_sensor.do_magnetventil` | opening |
| Kompressor 1 | `00000010` = 2 | `binary_sensor.do_kompressor_1` | running |
| Umschaltventil Kühlung | `00001000` = 8 | `binary_sensor.do_umschaltventil_kuehlung` | opening |
| Stufe 2 | `00010000` = 16 | `binary_sensor.do_stufe_2` | running |
| Pumpe-Heizung | `00100000` = 32 | `binary_sensor.do_pumpe_heizung` | running |
| Pumpe-Warmwasser / 3-Wege Brauchwasser | `01000000` = 64 | `binary_sensor.do_pumpe_warmwasser` | running |
| Pumpe-Quelle | `10000000` = 128 | `binary_sensor.do_pumpe_quelle` | running |

> Mask `00000100` (=4) is **not listed in the manual** → unused. (An earlier version wrongly created a
> "DO Alarm" there and a "Kurbelwannenheizung" at mask 8 — both removed 2026-11-14.)
> **Note:** `Pumpe-Heizung` (mask 32) is genuinely reported ON during a WW cycle on this install
> (observed live DO=0xE3). That is the controller's real output state, not a decode error.

### DI-Buffer (09.02, 0x00D0) — "Di-Buffer", OK/NOK & Aus/Ein. From manual:
| name | mask (manual "Dim") | binary_sensor | device_class | polarity |
|---|---|---|---|---|
| Öldruck-Pressostat / Motorschutzkompressor | `00000001` = 1 | `binary_sensor.di_oeldruck_kompressor_stoerung` | problem | on = bit set = NOK |
| Niederdruck-Pressostat | `00000010` = 2 | `binary_sensor.di_niederdruck_pressostat_stoerung` | problem | on = NOK |
| Hochdruck-Pressostat | `00000100` = 4 | `binary_sensor.di_hochdruck_pressostat_stoerung` | problem | on = NOK |
| Pumpe-Quelle | `00001000` = 8 | `binary_sensor.di_pumpe_quelle` | running | on = Ein |
| Sole Minimum | `00010000` = 16 | `binary_sensor.di_sole_minimum` | — | on = set |
| Externe Sollwertbeeinflussung | `01000000` = 64 | `binary_sensor.di_ext_sollwertbeeinflussung` | — | on = Ein |
| Externe Abschaltung | `10000000` = 128 | `binary_sensor.di_ext_abschaltung` | — | on = Ein |

> Mask `00100000` (=32) is unused in the manual's DI table. Live DI = 0x20 (only that unused bit) on a
> healthy running system, so all named DI bits read 0 = healthy.
> **CORRECTION (2026-11-14):** the pressostat sensors are now **NON-inverted** (`on` = bit set = NOK/fault).
> Verified: with DI=0x20 all three read **off (OK)** during a healthy WW cycle — no false alarms.
> (The earlier inverted version wrongly showed them as problems while idle/running.)

### Betriebszustände (09.00, 0x00CE) — MSB-first, on = active
| name | mask | binary_sensor | device_class |
|---|---|---|---|
| Ext. Steuerung | 128 | `binary_sensor.bz_ext_steuerung` | — |
| Unterbrechung | 16 | `binary_sensor.bz_unterbrechung` | problem |
| Hand-Betrieb | 8 | `binary_sensor.bz_hand_betrieb` | — |
| St2-Betrieb | 4 | `binary_sensor.bz_stufe_2_betrieb` | running |
| Hz-Betrieb | 2 | `binary_sensor.bz_heizbetrieb` | running |
| Ww-Betrieb | 1 | `binary_sensor.bz_warmwasserbetrieb` | running |

> Betriebszustände masks are not individually listed in the manual page; taken from the FHEM name array
> (reversed/MSB-first) and **validated live** (BZ=0x01 = Ww-Betrieb during a WW cycle ✓).

Template form: `{{ (states('sensor.waterkotte_<byte>')|int(0)) | bitwise_and(<mask>) > 0 }}`.

## Live validation (2026-11-14)

Captured raw bytes idle vs during a real **WW cycle** (WW Ist climbing toward Soll 50 °C):

| byte | idle | WW cycle | manual decode of WW cycle |
|---|---|---|---|
| DO | 0x00 | **0xE3** | Pumpe-Quelle + Pumpe-Ww + Pumpe-Heizung + Kompressor 1 + Magnetventil on ✓ |
| BZ | 0x00 | **0x01** | **Ww-Betrieb on** ✓ (Heizbetrieb off) |
| DI | 0x20 | 0x20 | only the unused mask-32 bit; all named DI bits = 0 → pressostats OK ✓ |

All decodes match physical reality and the manual. Pressostats read **off (OK)**; DO shows the real
running outputs incl. Pumpe-Heizung; BZ shows Warmwasserbetrieb.

## Open item (minor)
`Pumpe-Heizung` reporting ON during a WW-only cycle is the controller's genuine output state on this
install. If a future heating-cycle capture shows it behaves differently, revisit — but per the manual
the bit/label is correct.

## Creating / fixing the sensors (no restart)

All 21 are **template binary_sensors** created live via `ha_config_set_helper helper_type=template`,
`config.next_step_id=binary_sensor`, `config.state=<jinja>`, optional `config.device_class`.
To change a mask: delete via `ha_remove_helpers_integrations` (`target=<entry_id>`, `confirm=true`) and
recreate — no HA restart needed. (Entry-ids are listed via `ha_config_list_helpers helper_type=template`.)

## Dashboard
Section **"Status (Buffer)"** (last section of `lovelace-waterkotte`), three read-only `entities` cards:
DI-Buffer (Eingänge), DO-Buffer (Ausgänge), Betriebszustände. State + click-for-history.

---

# Menü 8 — Warnungen / Unterbrechungen / Ausfall (warning & fault bits)

Added 2026-11-14. Source: manual section **2.12 "Unterdrückung von Warnmeldungen"**
(photo `manual_2.12_warnmeldungen_bits.jpg`) for 8.01/8.02/8.03 bit masks, cross-checked with the FHEM
reference arrays (which also provide 8.00 Unterbrechungen). Both agree → **MSB-first**, masks below
are the literal set bit per signal. All are `on = active` (problem), i.e. a `1` means the
warning/interruption is present.

## Source bytes (Node-RED, parser #1, `offset = addr + 5`)
| Register | Address | offset | HA sensor |
|---|---|---|---|
| 8.00 Unterbrechungen | 0x00C3 | 200 | `sensor.waterkotte_unterbrechungen_byte` |
| 8.01 Warnung Eingangsseite | 0x00C4 | 201 | `sensor.waterkotte_warnung_eingang_byte` |
| 8.02 Warnung Ausgangsseite | 0x00C5 | 202 | `sensor.waterkotte_warnung_ausgang_byte` |
| 8.03 Warnung Sonstige | 0x00C6 | 203 | `sensor.waterkotte_warnung_sonstige_byte` |
| 8.04 Ausfall | 0x00C7 | 204 | `sensor.waterkotte_ausfall_byte` |

All read **0** on a healthy system (verified live 2026-11-14).

## 8.00 Unterbrechungen — I-codes (RESOLVED 2026-11-14 via manual §3.12)

**Authoritative source: manual section 3.12 "Displayanzeige"** (photo `manual_3.12_I-F-codes.jpg`),
which lists the controller's display codes: **I = Interruption (Unterbrechung)** and
**F = Failure (Ausfall)**. The manual lists only **5 interruptions (I01–I05)** and **5 failures
(F01–F05)** — far fewer than the 8-bit FHEM "Unterbrechungen" array that was used before, so the
earlier FHEM-named rows were wrong for this firmware and have been **replaced** by the I-codes below.

### I-code legend (manual §3.12)
| Code | Beschreibung |
|---|---|
| I01 | Unterbr. Temp QuelleAus < OK (Temperatur Quelle-Austritt < OK) |
| I02 | Unterbr. Begr. Hzg > OK (Hz-VorlaufTemp oder Kondensationsdruck > OK) |
| I03 | Externe Abschaltung |
| I04 | Unterbr. Schalthäufigkeit |
| I05 | Unterbr. Fühler QuelleAus NOK (Temperatur Quelle-Austritt nicht OK) |

### Deployed 8.00 binary_sensors (bit masks ascending, same convention as W-codes)
| mask | entity | note |
|---|---|---|
| 1 | `binary_sensor.i01_unterbr_temp_quelleaus_unter_ok` | I01 (provisional bit) |
| 2 | `binary_sensor.i02_unterbr_begr_hzg_ueber_ok` | I02 (provisional bit) |
| 4 | `binary_sensor.i03_externe_abschaltung` | I03 (provisional bit) |
| 8 | `binary_sensor.i04_unterbr_schalthaeufigkeit` | I04 (provisional bit) |
| 16 | `binary_sensor.i05_unterbr_fuehler_quelleaus_nok` | I05 (provisional bit) |

All `device_class: problem`, `on = active`. Byte currently **0** (no interruption). The exact
bit↔code mapping is **provisional** (assigned I01→mask1 … I05→mask16, same approach as the W-codes)
and should be confirmed as each I-code first fires live.

> The old FHEM-named rows (`binary_sensor.unterbr_*`, 8 masks) are now **orphans** — superseded by
> I01–I05. Safe to delete: `unterbr_temp_hz_vorlauf_zu_hoch`, `unterbr_ww_fuehler_defekt`,
> `unterbr_hz_fuehler_defekt`, `unterbr_wpumpe_fuehler_defekt`, `unterbr_schalthaeufigkeit`,
> `unterbr_ext_abschaltung`, `unterbr_temp_qaus_zu_niedrig`, `unterbr_ungueltiger_betriebs_mode`.

<details><summary>Previous FHEM-array decode (superseded, kept for reference)</summary>

| mask | signal | binary_sensor |
|---|---|---|
| 128 | Temp. Hz-Vorlauf zu hoch | `binary_sensor.unterbr_temp_hz_vorlauf_zu_hoch` |
| 64 | Ww-Fühler defekt | `binary_sensor.unterbr_ww_fuehler_defekt` |
| 32 | Hz-Fühler defekt | `binary_sensor.unterbr_hz_fuehler_defekt` |
| 16 | WPumpe-Fühler defekt | `binary_sensor.unterbr_wpumpe_fuehler_defekt` |
| 8 | Schalthäufigkeit | `binary_sensor.unterbr_schalthaeufigkeit` |
| 4 | Ext. Abschaltung | `binary_sensor.unterbr_ext_abschaltung` (no device_class) |
| 2 | Temp. QAus zu niedrig | `binary_sensor.unterbr_temp_qaus_zu_niedrig` |
| 1 | Ungültiger Betriebs-Mode | `binary_sensor.unterbr_ungueltiger_betriebs_mode` |

</details>

## 8.01 Warnung Eingangsseite — W-codes (RESOLVED 2026-11-14 via manual W-code table)

**The full W01–W17 warning-code legend is now documented** — photo `manual_warncodes_W01-W17.jpg`.
This is the authoritative source for the controller's "Wxx" display codes (the §2.12 page only had
suppression bit masks, no W-numbers). **W04 confirmed = "TDiff. QuelleAustritt − Verdampfung > OK"**,
which matches the live evidence (Quelle-Aus 6.0 − Verdampfer −2.6 = 8.6 K → active).

### Full W-code reference (manual)
| Code | Beschreibung |
|---|---|
| W01 | Verdampfungstemperatur < OK |
| W02 | Temperatur Quelle-Austritt < OK |
| W03 | TDiff. QuelleEintritt − QuelleAustritt > OK |
| **W04** | **TDiff. QuelleAustritt − Verdampfung > OK** |
| W05 | — (unused) |
| W06 | — (unused) |
| W07 | TDiff. HeizungsAustritt − Heizungseintritt < OK |
| W08 | TDiff. HeizungsAustritt − Heizungseintritt > OK |
| W09 | TDiff. Kondensation − HeizungsAustritt > OK |
| W10 | Fühler Temp. Kondensation NOK |
| W11 | Fühler Temp. Heizungsrücklauf NOK |
| W12 | Fühler Temp. Heizungsvorlauf NOK |
| W13 | Fühler Außenwandtemperatur NOK |
| W14 | Digital Output Handschaltung |
| W15 | Systemdruck NOK |
| W16 | Fühler Quelle-Eintritt NOK |
| W17 | Fühler Verdampfungstemperatur NOK |

> **Note on numbering:** the W-codes are a flat 1–17 list spanning all three §2.12 byte groups
> (8.01/8.02/8.03), with **W05/W06 unused**. So the controller's W-number does NOT map 1:1 to a
> single byte's §2.12 bit-mask — that's why the live W04 appeared at the 8.01 byte's **mask 16**, not
> mask 8. Only **8.01 mask 16 = W04** is live-confirmed; the exact bit↔W mapping for the other codes
> is inferred and should be confirmed as each warning is observed live.

### Deployed 8.01 binary_sensors
| mask | entity | note |
|---|---|---|
| 1 | `binary_sensor.w01_verdampfungstemp_zu_niedrig` | W01 (provisional bit) |
| 2 | `binary_sensor.w02_temp_quelle_austritt_zu_niedrig` | W02 (provisional bit) |
| 4 | `binary_sensor.w03_tdiff_quelleein_quelleaus_zu_hoch` | W03 (provisional bit) |
| 8 | `binary_sensor.8_01_diff_quelleaus_verdampfung_zu_hoch_maske_8` | §2.12 4th bit (currently off) |
| **16** | `binary_sensor.w04_tdiff_quelleaustritt_verdampfung_zu_hoch` | **W04 — live-confirmed, active** |

### Status & monitoring (2026-11-14)
- **W04 is currently ACTIVE** and is a *benign, expected* input-side warning: source-outlet vs
  evaporation spread (~8.6 K) is high on a cold day during heating. Not a fault — informational.
  It shows on the dashboard ("Warnungen & Störungen" → 8.01) as a `problem`-class entity.
- **Left as-is per user decision (2026-11-14); user will monitor.** No further action taken.
- **What's firm vs provisional:**
  - FIRM: the W01–W17 descriptions (manual legend `manual_warncodes_W01-W17.jpg`); 8.01 mask 16 = W04 (live).
  - PROVISIONAL: the exact bit↔W mapping for W01/W02/W03 (masks 1/2/4) — inferred, not yet seen
    active. Confirm each when it first fires live.
- **If W04 should ever be hidden/suppressed:** the controller supports suppressing warnings via
  Service-Kanal 8.10/8.11/8.12 (see manual §2.12) — not done here.

### 8.02 / 8.03 vs W-codes
The W07–W09 (Ausgangsseite TDiff) and W10–W17 (Fühler/sensor faults, Systemdruck, Do-Handschaltung)
codes correspond to the 8.02/8.03 byte groups documented below. Those bytes are currently 0 (no
warnings). Their individual binary_sensors (`warnaus_*`, `warnsonst_*`) use the §2.12 mask names;
cross-reference the W-code table above for the controller's display code if one ever activates.

## 8.02 Warnung Ausgangsseite (manual 2.12) — output/heating side
The W-code tags below are the cross-reference to the manual W-legend (provisional — not individually
live-confirmed, since this byte has never fired; the §2.12 **descriptions** are authoritative).
| mask | signal | binary_sensor | W-code |
|---|---|---|---|
| 32 | Diff. Kondensation-Vorlauf zu hoch | `binary_sensor.warnaus_diff_kondensation_vorlauf_zu_hoch` | W09 |
| 8 | Diff. HzgVorlauf-Rücklauf zu hoch | `binary_sensor.warnaus_diff_hzgvorlauf_ruecklauf_zu_hoch` | W08 |
| 4 | Diff. HzgVorlauf-Rücklauf zu niedrig | `binary_sensor.warnaus_diff_hzgvorlauf_ruecklauf_zu_niedrig` | W07 |
| 2 | Kondensationstemperatur zu hoch | `binary_sensor.warnaus_kondensationstemp_zu_hoch` | — |

## 8.03 Warnung Sonstige (manual 2.12 + FHEM) — sensor faults & special conditions
W-code tags are the provisional cross-reference (descriptions authoritative).
| mask | signal | binary_sensor | W-code |
|---|---|---|---|
| 16 | Solestand Minimum | `binary_sensor.warnsonst_solestand_minimum` | W15 (Systemdruck NOK) |
| 8 | Do-Buffer in Handstellung | `binary_sensor.warnsonst_do_buffer_in_handstellung` (no device_class) | W14 (Digital Output Handschaltung) |
| 4 | Außenwandfühler defekt | `binary_sensor.warnsonst_aussenfuehler_defekt` | W13 |
| 2 | Hz-Vorlauffühler defekt | `binary_sensor.warnsonst_hz_vorlauffuehler_defekt` | W12 |
| 1 | Hz-Rücklauffühler defekt | `binary_sensor.warnsonst_hz_ruecklauffuehler_defekt` | W11 |

> The dashboard row **labels were left unchanged** (not prefixed with Wxx) per user decision
> 2026-11-14 — this table is the lookup if an 8.02/8.03 bit ever activates.

## 8.04 Ausfall — F-codes (RESOLVED 2026-11-14 via manual §3.12)

The manual §3.12 "Displayanzeige" lists **5 failures (F01–F05)** (photo `manual_3.12_I-F-codes.jpg`).
Decoded into per-bit binary_sensors (same ascending-mask convention as the W-/I-codes), plus the
existing summary sensor retained as a header indicator.

### F-code legend (manual §3.12)
| Code | Beschreibung |
|---|---|
| F01 | Ausfall HD-Pressostat (Hochdruckpressostat) |
| F02 | Ausfall ND-Pressostat (Niederdruckpressostat) |
| F03 | Ausfall Motorschutz des Kompressors |
| F04 | Ausfall Pumpe Quelle (Pumpe der Wärmequelle) |
| F05 | Ausfall Nasslauf (Kältekreislauf; Nasslauf) |

### Deployed 8.04 binary_sensors
| mask | entity | note |
|---|---|---|
| — | `binary_sensor.ausfall_vorhanden` | summary: `{{ ausfall_byte > 0 }}` (kept as header) |
| 1 | `binary_sensor.f01_ausfall_hd_pressostat` | F01 (provisional bit) |
| 2 | `binary_sensor.f02_ausfall_nd_pressostat` | F02 (provisional bit) |
| 4 | `binary_sensor.f03_ausfall_motorschutz_kompressor` | F03 (provisional bit) |
| 8 | `binary_sensor.f04_ausfall_pumpe_quelle` | F04 (provisional bit) |
| 16 | `binary_sensor.f05_ausfall_nasslauf` | F05 (provisional bit) |

All `device_class: problem`, `on = active`. Byte currently **0** (no failure). Bit↔code mapping is
**provisional** (F01→mask1 … F05→mask16); confirm as each F-code first fires live.

## Dashboard
Section **"Warnungen & Störungen"** (last section of `lovelace-waterkotte`), read-only `entities`
cards: Unterbrechungen (I01–I05, 8.00), Ausfälle (F01–F05, 8.04), Warnungen Eingangsseite (8.01),
Ausgangsseite (8.02), Sonstige (8.03).

---

# Menü 6 — Hand-Ein/Ausschaltung von Relaisausgängen (6.18 / 6.19)

Added 2026-11-14 (**read-only** so far). Source: manual section **2.11 "Hand-Ein/Ausschaltung von
Relaisausgängen"** (photo `manual_2.11_handkanal.jpg`). Lets a single relay output be forced on/off
by hand via Menü 6 "Handschaltung": **6.18 DO-Handkanal** selects *which* relay (1–8), **6.19
DO-Handkanal-Ein** sets its state (0=Aus, 1=Ein).

## Source bytes (Node-RED, parser #2, `offset = (addr − 0x00F3) + 5`)
| Register | Address | offset | HA sensor |
|---|---|---|---|
| 6.18 DO-Handkanal | 0x016E | 128 | `sensor.waterkotte_do_handkanal` (number 0–8) |
| 6.19 DO-Handkanal-Ein | 0x016F | 129 | `sensor.waterkotte_do_handkanal_ein` (0/1) |

Live at install: both **0** (no manual override active).

## Handkanal-Nummer → Relaisausgang (from the manual table)
| Nr | Relaisausgang |
|---|---|
| 0 | — (keine Handschaltung) |
| 1 | Magnetventil Kältekreislauf |
| 2 | Kompressor |
| 3 | Sammelstörmeldung |
| 4 | Motorventil Kühlen |
| 5 | Stufe 2 |
| 6 | Pumpe-Heizung |
| 7 | Pumpe + Motorventil Warmwasser |
| 8 | Pumpe-Quelle |

> Note the Handkanal numbering (1–8) is a different order than the 09.01 DO-Buffer bit order.

## Helper entities (display)
- `sensor.do_handkanal_auswahl` — template sensor mapping the 6.18 number → relay name (shows "—" when 0).
  The mapping logic lives **in this HA template helper's `state`** (not in Node-RED):
  ```jinja
  {% set m = {'0':'—','1':'Magnetventil Kältekreislauf','2':'Kompressor','3':'Sammelstörmeldung',
              '4':'Motorventil Kühlen','5':'Stufe 2','6':'Pumpe-Heizung',
              '7':'Pumpe + Motorventil Warmwasser','8':'Pumpe-Quelle'} %}
  {{ m.get(states('sensor.waterkotte_do_handkanal')|int(0)|string, 'unbekannt') }}
  ```
  Edit via HA → Settings → Devices & Services → Helpers → "DO-Handkanal Auswahl" (no restart).
- `binary_sensor.do_handkanal_ein` — template binary_sensor, `on` = 6.19 > 0 (Hand-Ein aktiv), device_class running.

## Dashboard
Section **"Hand-Ein/Ausschaltung von Relaisausgängen"** (last section of `lovelace-waterkotte`):
Ausgewählter Relaisausgang (name), Hand-Ein aktiv, plus the two raw sensors.

## Writing (NOT YET implemented)
Only reading is done. To later add manual relay control from HA, writing 6.18 (`0x016E`) and 6.19
(`0x016F`) would follow the same framed-write + CRC pattern as the temperature/time setters
(write command `0x13`, 1-byte value, `slice(2,9?)` — verify the CRC slice for a 1-byte payload).
**⚠️ Safety:** the manual warns the Sicherheitskette is bypassed on Hand-Ein — forcing a relay (esp.
compressor/Kompressor) can damage the heat pump. Do NOT implement writes here without explicit intent
and care.
