# Register map – Waterkotte WWPR via pCOWeb

Determined on a Waterkotte Ai1QLi 5011.5 A (WWPR, firmware 01.05.04, BIOS 6.31), with the controller set to
**Modbus slave**, address 32, 19200 8N1. Starting point was Waterkotte's "Software Technische Information
Modbus/TCP" (2017, for firmware 01.07.xx); the map matches the older firmware for everything checked so far.
"Verified" = confirmed against the display, the maintenance protocol or live behaviour.

## Addressing

| Waterkotte | Modbus | Notes |
|---|---|---|
| Analog `An` | holding register `n` | value × 10, signed 16 bit |
| Integer `In` | holding register `128 + n` | only up to HR 249 is readable |
| Digital `Dn` | coil `n` | coils 0–199 readable |

The controller rejects reads beyond HR 249 / coil 199 (exception 02) and more than 125 registers per request.

## Analog values

| Reg | Name | Verified | Notes |
|---|---|---|---|
| A1 | Außentemperatur | ✅ display | |
| A2 | Außentemperatur Ø 1 h | | |
| A3 | Außentemperatur Ø 24 h | | the controller chooses heating / neutral / cooling from this value |
| A10 | Temperatur Rücklauf Soll | ✅ | heating setpoint, switches to the hot water setpoint during a charge |
| A11 | Rücklauftemperatur | ✅ display | only meaningful while water flows |
| A12 | Vorlauftemperatur | ✅ display | can drift up at standstill (sensor near a heat source) |
| A17/A18 | Raumtemperatur | | −40.0 = no sensor connected |
| A19 | Warmwassertemperatur | ✅ display | |
| A25–A29 | P el / P th / COP | | always 0 on the reference unit |
| A30 | Heizkreistemperatur aktuell | ✅ display | |
| A31 | Heizkreistemperatur gefordert | ✅ display | follows the heating curve incl. the display's curve adjustment |
| A33/A34 | Kühlkreis Ist/Soll | ✅ A34 | A34 follows A109; A33 ≈ return temperature |
| A37 | Warmwasser gefordert | ✅ display | |
| A38 | Warmwasser Solltemperatur (BMS) | ✅ writable | display and A37 follow. Writes through the stock pcod/tcpmodbus are acknowledged but often not forwarded – use mbproxy |
| A50 | Geforderte Verdichterleistung | | writable per documentation – do not write |
| A51 | Heizungsumwälzpumpe % | ✅ | 75 % during the cyclic measurement, 100 % for hot water |
| A56 | Soll des aktiven Betriebs | ✅ | heating: = A31, hot water charge: = A37; never 0 |
| A57 | Warmwassertemperatur (Ladung) | | tracks A19 during a charge |
| A58 | Verdichterleistung % | ✅ | 100 % during hot water |
| A61 | Schaltdifferenz Heizen | ✅ display, ✅ writable | |
| A91 | Norm-Außentemperatur | ✅ display | design point of the heating curve (e.g. −12.0). **Do not change** |
| A92 | Heizkreis Sollwert Norm | ✅ display, ✅ writable | return setpoint at A91 |
| A93 | Heizgrenze (Ø 24 h outside) | ✅ display, ✅ writable | heating up to this 24 h outside temperature; above it only hot water (or cooling above A108) |
| A94 | Heizgrenze Soll | ✅ display, ✅ writable | return setpoint at A93 |
| A95 | max. Rücklauf | ✅ display | do not change |
| A96 | aktueller Rücklauf-Soll | ✅ | computed from the heating curve, without the display's curve adjustment |
| A103 | ? | | 2.0 on the reference unit |
| A107 | Schaltdifferenz Kühlen | ✅ display, ✅ writable | cooling starts at return ≥ A109 + A107 |
| A108 | Kühlgrenze (Ø 24 h outside) | ✅ display, ✅ writable | above it the heat pump switches to cooling |
| A109 | Kühlkreis Soll (return) | ✅ display, ✅ writable | A34 follows |

Not exposed via Modbus (checked by changing them on the display while watching HR 1–249 and coils 0–199):
the hot water hysteresis (Schaltdifferenz Warmwasser), the display's heating curve adjustment
("Temperaturanpassung", one step = +0.5 K on A31, so adjustment = A31 − A96) and the holiday function.

## Integer values (HR = 128 + n)

| Reg | HR | Name | Verified | Notes |
|---|---|---|---|---|
| I5–I9 | 133–137 | Tag / Monat / Jahr / Stunde / Minute | ✅ | controller clock |
| I10 | 138 | Betriebsstunden Verdichter 1 | ✅ | **stops at 16384 (2^14)** – also on the display and in the yearly statistics |
| I18 | 146 | Betriebsstunden Heizungsumwälzpumpe | ✅ | stops at 16384 as well |
| I30 | 158 | Handabschaltung Heizbetrieb durch BMS | | 1 = on. **Never write 0: it switches heating off** |
| I31 | 159 | Handabschaltung Kühlbetrieb durch BMS | ✅ writable | 1 = on, 0 = off (display: "Man. Betrieb aus") |
| I32 | 160 | Handabschaltung Warmwasser durch BMS | ✅ writable | 1 = on, 0 = off (display: "Man. Betrieb aus") |
| I33 | 161 | Handabschaltung Pool durch BMS | | never write |
| I30–I42 | 158–170 | flags | | always 1 on the reference unit (except HR 164 = 0); the package uses them to detect pcod's all-zero dropouts |
| I51 | 179 | Schaltzustand | ✅ | bits below |
| I52 | 180 | Ausfälle F-Codes | ✅ | bits below |
| I53 | 181 | Unterbrechungen I-Codes | ✅ | bits below |
| I106 | 234 | Betriebsart? | | 3 during hot water |

### I51 / HR 179 – Schaltzustand

| Bit | Value | Meaning | Verified |
|---|---|---|---|
| 0 | 1 | Quellenpumpe | |
| 1 | 2 | Heizungsumwälzpumpe | ✅ |
| 2 | 4 | Freigabe Regelung EVD / Magnetventil | |
| 3 | 8 | Verdichter 1 | ✅ |
| 4 | 16 | Verdichter 2 | |
| 5 | 32 | ext. Wärmeerzeuger | |
| 6 | 64 | Alarmausgang | |
| 7 | 128 | Motorventil Kühlbetrieb | |
| 8 | 256 | Motorventil Warmwasser | ✅ |
| 9 | 512 | Motorventil Pool | |
| 12 | 4096 | 4-Wege-Ventil im Kältekreis | stays 0 when a Mitsubishi outdoor unit defrosts on its own |
| 13–15 | | Mischerkreis 1 Pumpe / AUF / ZU | |

Observed: 2 = cyclic measurement (pump only); 256 → 258 → 266 = hot water request → pump → compressor.

### I52 / HR 180 – Ausfälle (F-Codes)

Bit 0 F100 Motorschutz 1 · 1 F101 Motorschutz 2 · 2 F102 Phasenfehler · 3 F103 Primärseite ·
4 F110 HD-Pressostat · 5 F120 ND-Pressostat · 6 F121 Drucküberwachung Verdampfer ·
7 F122 Temperaturüberwachung Verdampfer · 8 F123 Nasslauf · 9 F111 Verflüssigungstemperatur zu niedrig ·
10 F104 Störung Sekundärseite · 11 F105 Motorschutz 3 · 12 F130 4-Wege-Ventil ·
**14 communication trigger (toggles constantly, ignore)** · 15 Sammelmeldung

### I53 / HR 181 – Unterbrechungen (I-Codes)

Bit 0 I011 T Quelle aus · 1 I012 p Kondensator · **2 I013 Ext. Abschaltung = EVU lock** ·
3–5 I014 Schalthäufigkeit · 6 I015 T Quelle ERR · 7 I501 Vorlauf-Grenzwert Kühlkreis ·
8 I502 Rücklauf-Grenzwert Kühlkreis · 9–12 I019 Stillstandszeit Verdichter · 13/14 I014 · 15 Sammelmeldung

## Coils

| Coil | Meaning | Verified |
|---|---|---|
| D1 | Störung Verdichter 1 | documentation |
| D2 | Störung Verdichter 2 | documentation |
| D21 | Warmwasser-Anforderung | ✅ |
| D38, D71, D81 | Heizungsumwälzpumpe läuft | ✅ |
| D120–D122 | Umschaltventil Warmwasser | ✅ (D123 inverse) |
