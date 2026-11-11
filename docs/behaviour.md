# Behaviour observed on the heat pump

Observations on the reference unit (Waterkotte Ai1QLi with a Mitsubishi Zubadan outdoor unit, WWPR firmware
01.05.04). They explain several of the derived sensors in the package.

- **Cyclic measurement:** every ~12 min the circulation pump runs ~2 min at 75 % (I51 = 2) so the sensors see
  real water. At standstill the flow sensor drifts up (local heat source), so flow and return are only
  meaningful while the pump runs. Hence `sensor.warmepumpe_vorlauf_durchfluss` / `_rucklauf_durchfluss`, which
  only show values while water flows, and `binary_sensor.warmepumpe_zyklische_messung`. The operating state
  shows it as "Zyklische Messung"; the pump overrun after compressor operation shows as "Pumpenlauf".
- **Hot water charge:** request (D21, I51 bit 8, I106 = 3) → valve and pump at 100 % → compressor ~1.5 min later
  → charge → 2 min pump overrun. Switch-on point = setpoint − hysteresis (5 K on the reference unit).
- **EVU lock:** the grid operator's time switch shows up as I53 bit 2 ("I013 Ext. Abschaltung"). A cyclic
  measurement starts right after the lock ends. If a lock lasts much longer than the time switch allows, the
  heat pump may have missed its end: `binary_sensor.warmepumpe_evu_sperre_hangt` (75 min, adjust to your
  schedule).
- **Defrost:** a Mitsubishi outdoor unit defrosts on its own (~2 min, flow drops several K *below* return). The
  WWPR does not report it (I51 bit 12 stays 0); `binary_sensor.warmepumpe_abtauung_erkannt` detects it from the
  temperatures, and the spread ignores it.
- **Spread:** only meaningful while actually heating (3–4 K at 1.5–2 kW on the reference unit). Large values at
  standstill mean nothing; `sensor.warmepumpe_spreizung` is only available while the compressor heats.
- **Heating curve:** a straight line through (A91, A92) and (A93, A94), capped at A95, flat above the heating
  limit. The controller regulates on the return temperature. Moving A92 and A94 by the same amount shifts the
  curve in parallel, which is what "Kennlinie wärmer / kälter" and the holiday function do.
- **Operating hours:** I10 / I18 stop at 16384 h. The package counts compressor and pump hours itself, plus the
  runtime per year for heating, hot water and cooling.
- **Writes:** reading is harmless. Writing parameters wears the controller's non-volatile memory, so the package
  only writes when a value really changes. Once the controller acknowledged a write without applying it; the
  same write a few minutes later worked, which is why `script.wp_register_schreiben` checks the read-back and
  tries once more.
- **BMS switches:** I30 / I31 / I32 = 0 switch heating / cooling / hot water off ("Man. Betrieb aus" on the
  display); 1 switches them back on. The Steuerung tab has a switch for each, e.g. to stop a function in an
  emergency. They are locked while a holiday is planned or running, because the holiday function sets I31 / I32
  itself and restores them at the end.
