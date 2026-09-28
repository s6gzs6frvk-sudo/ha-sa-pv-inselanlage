# PV surplus heating, battery protection and per-phase grid emergency heating for a zero-export off-grid house (Home Assistant, YAML only)

**[English](#english) · [Deutsch](#deutsch)**

---

<a name="english"></a>
## English

### What this is

A holiday house in Kosovo that stands empty most of the time. The PV system must not export (zero export), the
inverters run in SBU mode (solar, then battery, grid only below 49 V). Home Assistant decides when the three heating
rods charge the 1000 L buffer tank from PV surplus, when the tank is heated from battery or grid in an emergency, when
the floor-heating pumps run and which setpoints the thermostats get. Everything is YAML automations and template
sensors, no Python, no Node-RED, so it stays maintainable without programming skills.

Priorities, in this order: safety, then "the house must never go dark", then comfort, then maximum self-consumption.

**Hardware**

| Part | Details |
|---|---|
| PV | 35 × 435 Wp, ~15.2 kWp, south roof |
| Inverters | 3 × Anenji ANJ-11KW-48V in parallel, 3-phase (Voltronic type, no BMS bus) |
| Battery | 2 × 16S 320 Ah LiFePO4, each with JBD BMS DP24S002 plus a 5 A active balancer |
| Tank | 1000 L hygienic buffer tank, 3 heating rods of ~2.7 kW on Shelly 1PM Gen4 relays (5-minute auto-off as hardware fallback) |
| Heating | Floor heating in screed over 4 floors, mixer, 2 pump circuits |
| Control | Home Assistant on Raspberry Pi 4, data via Solar Assistant (official HA integration) |
| Grid | Village grid with voltage dips (185 V happens), frequency 49.8 to 50.1 Hz |

### Files

| File | Role |
|---|---|
| `00_template_sensoren.yaml` | complete `configuration.yaml`: 32 template sensors (safe sensors with sentinel values, permissions, sun signal, phase protection, status texts) |
| `01_helfer.yaml` | reference for the helpers (create them in the HA UI) |
| `02_automation_heizstab_sicherheit.yaml` | safety: lock and target 0 on danger, watchdog for Solar Assistant, self-healing |
| `03_automation_heizstab_entscheidung.yaml` | decision: computes only the target level 0 to 3 |
| `04_automation_heizstab_aktorik.yaml` | actuation: relays, rotation by running hours, auto-off refresh |
| `05_automation_heizungspumpen.yaml` | pumps, dumping a full tank into the house, thermostat setpoints |
| `08_automation_netz_notlauf_perphase.yaml` | grid emergency heating per phase: a weak phase drops out on its own |
| `09_automation_akku_notladung.yaml` | the only exception to "no grid power into the battery": emergency charge via the Solar Assistant `select` entity |
| `10_automation_klima_wohnzimmer.yaml` | air conditioner as heat pump from PV surplus |
| `11_automation_alarm.yaml` | phone alerts (battery critical, grid refused, grid outage, frost, lock stuck) |
| `PRUEF_TEMPLATE_entitaeten.txt` | template for Developer Tools: lists missing entities |

Comments inside the YAML are German. Ask in an issue if you want a specific part translated.

### The ideas worth discussing

1. **Sun signal from MPPT voltage instead of PV power.** With zero export and a full battery the inverter curtails
   PV, so measured power is ~0 even in full sun. The MPPT voltage rises towards open circuit (~400 V by day, ~25 V at
   night). Above 200 V means "sun is there", whatever the power reading says.
2. **Battery floor by voltage, not SOC.** The JBD SOC drifts badly at the bottom (17 % at 48.8 V). Heating from the
   battery stops at 51.2 V and only restarts at 52.5 V and 40 %. The inverter switches to grid at 49.0 V, the house
   cuts off at 47.0 V, the BMS at 41.5 V. Every layer catches the next one.
3. **Grid emergency heating per phase.** If the tank gets too cold and the grid is there, up to three rods heat from
   the grid, but each rod hangs on exactly one inverter and therefore one phase. If a phase dips below 185 V only its
   rod drops out, the others keep heating. Rod-to-inverter mapping uses the serial number because Solar Assistant
   numbers inverters by detection order.
4. **Sentinel values for dead sensors.** Every Solar Assistant sensor has a safe twin: if the value is missing it
   returns −99999 or 99999, and the logic treats that as "unknown = do not heat", never as 0.
5. **Auto-off as hardware fallback.** The Shelly relays switch off by themselves after 5 minutes. HA re-confirms
   running rods every 2 minutes. If HA dies, the rods are off within 5 minutes.
6. **Adaptive hold against hunting at the PV limit.** If a rod was on for less than 4 minutes before it had to go off
   again, HA waits 20 instead of 5 minutes before the next attempt.
7. **One shared "on grid" sensor** built from inverter mode and grid power with hysteresis, so every automation sees
   the same truth and actuation and grid emergency heating never fight over the relays.

### Installation (short)

1. Create the helpers from `01` in the UI.
2. Use `00` as `configuration.yaml` (the file is complete, including `default_config:` and the `!include` lines),
   check configuration, reload template entities.
3. Add automations `02` to `11` as new automations via the YAML editor.
4. Paste `PRUEF_TEMPLATE_entitaeten.txt` into Developer Tools → Template: it shows which entities are missing.

### Adapting to another system

- All entity IDs (Shelly, thermostats, Solar Assistant sensors) belong to my house. The typo `bolier_3` is real on my
  side and stays.
- In `00` under "Phase 1/2/3 Stabil" replace the placeholders `SERIAL_WR_...` with your own inverter serial numbers.
- The Solar Assistant strings (`device_mode` = "Grid", `charger_source_priority` = "Solar only (OSO)") are for
  Voltronic types; check them for other inverters.
- Thresholds (185/190 V phase protection, 500 W grid, temperatures, voltages) each carry their reasoning in a comment.

### Feedback wanted

- Is the voltage floor (51.2 / 52.5 V at 16S) a sensible choice, or would you rather pull cell voltages via a BMS
  Bluetooth integration (BMS_BLE-HA) and gate on the lowest cell?
- Grid emergency heating per phase: any reason to detect "on grid" differently than "inverter mode or grid power"?
- The decision logic in `03` is one large Jinja template with priorities. Who has solved this more elegantly without
  Python?
- Inverter self-consumption at night on grid with "only solar" charging: experiences with Voltronic types?
- Anything that looks risky to you while reading.

### Status and safety note

As of 28 Sep 2026. Running since June 2026, two blackouts in September whose cause (inverters refused the grid) is
fixed. The first winter is ahead. This is my house and my responsibility. Whoever takes something from here controls
8 kW of heating rods and a 32 kWh battery: electrical work by a professional, fuses and mechanical temperature limiters
are mandatory, the software is only the second layer. No warranty.

### License

MIT, see `LICENSE`.

---

<a name="deutsch"></a>
## Deutsch

### Was das ist

Ein Ferienhaus im Kosovo, das die meiste Zeit leer steht. Die PV darf nicht einspeisen (Zero-Export), die
Wechselrichter laufen im SBU-Modus (Solar, dann Akku, Netz nur unter 49 V). Home Assistant entscheidet, wann die drei
Heizstäbe den 1000-L-Speicher aus PV-Überschuss laden, wann der Speicher im Notfall aus Akku oder Netz geheizt wird,
wann die Pumpen laufen und welche Sollwerte die Thermostate bekommen. Alles in YAML-Automationen und Template-Sensoren,
kein Python, kein Node-RED, damit es ohne Programmierkenntnisse wartbar bleibt.

Prioritäten in dieser Reihenfolge: Sicherheit, dann „das Haus darf nie dunkel werden", dann Komfort, dann maximale
Autarkie.

**Hardware**

| Teil | Details |
|---|---|
| PV | 35 × 435 Wp, ~15,2 kWp, Süddach |
| Wechselrichter | 3 × Anenji ANJ-11KW-48V parallel, 3-phasig (Voltronic-Typ, kein BMS-Bus) |
| Akku | 2 × 16S 320 Ah LiFePO4, je JBD-BMS DP24S002 plus 5-A-Aktivbalancer |
| Speicher | 1000 L Hygienespeicher, 3 Heizstäbe je ~2,7 kW auf Shelly 1PM Gen4 (5-min-Auto-Off als Hardware-Rückfallebene) |
| Heizung | Fußbodenheizung in Estrich über 4 Stockwerke, Mischer, 2 Pumpenkreise |
| Steuerung | Home Assistant auf Raspberry Pi 4, Daten über Solar Assistant (offizielle HA-Integration) |
| Netz | Dorfnetz mit Spannungseinbrüchen (185 V kommen vor), Frequenz 49,8 bis 50,1 Hz |

### Dateien

| Datei | Rolle |
|---|---|
| `00_template_sensoren.yaml` | komplette `configuration.yaml`: 32 Template-Sensoren (abgesicherte Sensoren mit Sentinel-Werten, Freigaben, Sonnen-Signal, Phasenschutz, Status-Texte) |
| `01_helfer.yaml` | Referenz der Helfer (in HA über die Oberfläche anlegen) |
| `02_automation_heizstab_sicherheit.yaml` | Sicherheit: Sperre und Soll 0 bei Gefahr, Watchdog für Solar Assistant, Selbstheilung |
| `03_automation_heizstab_entscheidung.yaml` | Entscheidung: berechnet nur die Soll-Stufe 0 bis 3 |
| `04_automation_heizstab_aktorik.yaml` | Aktorik: Relais, Rotation nach Betriebsstunden, Auto-Off-Refresh |
| `05_automation_heizungspumpen.yaml` | Pumpen, Abfuhr des vollen Speichers ins Haus, Thermostat-Sollwerte |
| `08_automation_netz_notlauf_perphase.yaml` | Netz-Notlauf pro Phase: schwache Phase fällt einzeln raus |
| `09_automation_akku_notladung.yaml` | einzige Ausnahme von „kein Netzstrom in den Akku": Notladung über die Solar-Assistant-`select`-Entität |
| `10_automation_klima_wohnzimmer.yaml` | Klimaanlage als Wärmepumpe aus PV-Überschuss |
| `11_automation_alarm.yaml` | Alarme aufs Handy (Akku kritisch, Netz verweigert, Netzausfall, Frost, Sperre) |
| `PRUEF_TEMPLATE_entitaeten.txt` | Template für die Entwicklerwerkzeuge: listet fehlende Entitäten |

### Die Ideen, um die es geht

1. **Sonnen-Signal über die MPPT-Spannung statt über die PV-Leistung.** Bei Zero-Export und vollem Akku regelt der
   Wechselrichter die PV ab, die gemessene Leistung liegt bei null, obwohl die Sonne scheint. Die MPPT-Spannung steigt
   dabei Richtung Leerlauf (~400 V gegen ~25 V nachts). Über 200 V heißt „Sonne da", egal was die Leistung sagt.
2. **Akkuboden nach Spannung, nicht nach SOC.** Der SOC des JBD driftet unten stark (17 % bei 48,8 V). Heizen aus dem
   Akku stoppt bei 51,2 V, startet erst wieder ab 52,5 V und 40 %. Der Wechselrichter geht bei 49,0 V ans Netz,
   das Haus schaltet bei 47,0 V ab, das BMS bei 41,5 V. Jede Schicht fängt die nächste.
3. **Netz-Notlauf pro Phase.** Wird der Speicher zu kalt und das Netz ist da, heizen bis zu drei Stäbe aus dem Netz,
   aber jeder Stab hängt an genau einem Wechselrichter und damit an einer Phase. Bricht eine Phase unter 185 V ein,
   fällt nur ihr Stab raus, die anderen heizen weiter. Die Zuordnung Stab zu Wechselrichter läuft über die
   Seriennummer, weil Solar Assistant die Wechselrichter nach Erkennungsreihenfolge nummeriert.
4. **Sentinel-Werte für tote Sensoren.** Jeder Sensor aus Solar Assistant hat einen abgesicherten Zwilling: fehlt der
   Wert, liefert er −99999 bzw. 99999, und die Logik behandelt das als „unbekannt = nicht heizen", nie als 0.
5. **Auto-Off als Hardware-Rückfallebene.** Die Shelly-Relais schalten nach 5 Minuten selbst ab. HA bestätigt laufende
   Stäbe alle 2 Minuten. Stirbt HA, sind die Stäbe nach spätestens 5 Minuten aus.
6. **Adaptiver Hold gegen das Pendeln am PV-Limit.** War ein Stab kürzer als 4 Minuten an, bevor er wieder runter
   musste, wartet HA 20 statt 5 Minuten bis zum nächsten Versuch.
7. **Ein gemeinsamer „am Netz"-Sensor** aus Wechselrichter-Modus und Netzbezug mit Hysterese, damit alle Automationen
   dieselbe Wahrheit sehen und sich Aktorik und Netz-Notlauf nicht um die Relais streiten.

### Einbau (kurz)

1. Helfer aus `01` in der Oberfläche anlegen.
2. `00` als `configuration.yaml` einsetzen (die Datei ist vollständig, inklusive `default_config:` und den
   `!include`-Zeilen), Konfiguration prüfen, Template-Entitäten neu laden.
3. Automationen `02` bis `11` je als neue Automation im YAML-Editor einfügen.
4. `PRUEF_TEMPLATE_entitaeten.txt` in Entwicklerwerkzeuge → Template einfügen: zeigt, welche Entitäten fehlen.

### Anpassen auf eine andere Anlage

- Alle Entity-IDs (Shelly, Thermostate, Solar-Assistant-Sensoren) sind auf mein Haus bezogen. Der Tippfehler
  `bolier_3` ist bei mir echt und bleibt so.
- In `00` bei „Phase 1/2/3 Stabil" die Platzhalter `SERIAL_WR_...` durch die eigenen Seriennummern ersetzen.
- Die Strings der Solar-Assistant-Integration (`device_mode` = „Grid", `charger_source_priority` = „Solar only (OSO)")
  gelten für Voltronic-Typen; bei anderen Wechselrichtern prüfen.
- Schwellen (185/190 V Phasenschutz, 500 W Netz, Temperaturen, Spannungen) stehen jeweils mit Begründung im Kommentar.

### Feedback gesucht

- Ist der Spannungs-Akkuboden (51,2 / 52,5 V bei 16S) sinnvoll gewählt, oder würdet ihr den SOC über eine
  BMS-Bluetooth-Integration (BMS_BLE-HA) bevorzugen und nach der niedrigsten Zelle steuern?
- Netz-Notlauf pro Phase: gibt es einen Grund, „am Netz" anders zu erkennen als über „Wechselrichter-Modus oder Netzbezug"?
- Die Entscheidungslogik in `03` ist ein großes Jinja-Template mit Prioritäten. Wer hat das schon eleganter gelöst,
  ohne Python?
- Wechselrichter-Eigenbedarf nachts am Netz bei „nur Solar laden": Erfahrungen mit Voltronic-Typen?
- Alles, was euch beim Lesen als riskant auffällt.

### Status und Sicherheitshinweis

Stand 28.09.2026. Läuft seit Juni 2026, im September zwei Blackouts, deren Ursache (Wechselrichter nahmen das Netz
nicht an) behoben ist. Der erste Winter steht bevor. Das ist mein Haus und meine Verantwortung. Wer davon etwas
übernimmt, steuert damit Heizstäbe mit 8 kW und einen 32-kWh-Akku: Elektrik vom Fachmann, Sicherungen und
mechanische Temperaturbegrenzer sind Pflicht, die Software ist nur die zweite Ebene. Keine Gewähr.

### Lizenz

MIT, siehe `LICENSE`.
