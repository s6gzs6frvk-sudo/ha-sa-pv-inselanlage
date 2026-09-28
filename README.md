# PV-Überschuss-Heizung, Akku-Schutz und Netz-Notlauf für ein Zero-Export-Inselhaus (Home Assistant, nur YAML)

**English summary:** Home Assistant YAML (no Python, no Node-RED) that runs a 15 kWp zero-export PV system in a
holiday house in Kosovo: three parallel Voltronic-type inverters (Anenji 11 kW, 3-phase), 32 kWh LiFePO4 with JBD BMS
(no BMS-to-inverter communication), a 1000 L buffer tank with three 2.7 kW heating rods on Shelly relays, floor-heating
pumps and thermostats. Data comes from Solar Assistant. Priorities: safety, then "the house must never go dark", then
comfort, then maximum self-consumption. I am looking for feedback on the control logic. Comments inside the YAML are
German; the questions I have are listed under "Feedback gesucht" below.

## Was das ist

Ein Ferienhaus, das die meiste Zeit leer steht. Die PV darf nicht einspeisen (Zero-Export), die Wechselrichter laufen
im SBU-Modus (Solar, dann Akku, Netz nur unter 49 V). Home Assistant entscheidet, wann die drei Heizstäbe den
1000-L-Speicher aus PV-Überschuss laden, wann der Speicher im Notfall aus Akku oder Netz geheizt wird, wann die
Pumpen laufen und welche Sollwerte die Thermostate bekommen. Alles in YAML-Automationen und Template-Sensoren,
damit es ohne Programmierkenntnisse wartbar bleibt.

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

## Dateien

| Datei | Rolle |
|---|---|
| `00_template_sensoren.yaml` | komplette `configuration.yaml`: 32 Template-Sensoren (abgesicherte Sensoren mit Sentinel-Werten, Freigaben, Sonnen-Signal, Phasenschutz, Status-Texte) |
| `01_helfer.yaml` | Referenz der Helfer (in HA über die Oberfläche anlegen) |
| `02_automation_heizstab_sicherheit.yaml` | Sicherheit: Sperre und Soll 0 bei Gefahr, Watchdog für Solar Assistant, Selbstheilung |
| `03_automation_heizstab_entscheidung.yaml` | Entscheidung: berechnet nur die Soll-Stufe 0 bis 3 |
| `04_automation_heizstab_aktorik.yaml` | Aktorik: Relais, Rotation nach Betriebsstunden, Auto-Off-Refresh |
| `05_automation_heizungspumpen.yaml` | Pumpen, Abfuhr des vollen Speichers ins Haus, Thermostat-Sollwerte |
| `08_automation_netz_notlauf_perphase.yaml` | Netz-Notlauf pro Phase: schwache Phase fällt einzeln raus |
| `09_automation_akku_notladung.yaml` | Einzige Ausnahme von „kein Netzstrom in den Akku": Notladung über die Solar-Assistant-`select`-Entität |
| `10_automation_klima_wohnzimmer.yaml` | Klimaanlage als Wärmepumpe aus PV-Überschuss |
| `11_automation_alarm.yaml` | Alarme aufs Handy (Akku kritisch, Netz verweigert, Netzausfall, Frost, Sperre) |
| `PRUEF_TEMPLATE_entitaeten.txt` | Template für die Entwicklerwerkzeuge: listet fehlende Entitäten |

## Die Ideen, um die es geht

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

## Einbau (kurz)

1. Helfer aus `01` in der Oberfläche anlegen.
2. `00` als `configuration.yaml` einsetzen (die Datei ist vollständig, inklusive `default_config:` und den
   `!include`-Zeilen), Konfiguration prüfen, Template-Entitäten neu laden.
3. Automationen `02` bis `11` je als neue Automation im YAML-Editor einfügen.
4. `PRUEF_TEMPLATE_entitaeten.txt` in Entwicklerwerkzeuge → Template einfügen: zeigt, welche Entitäten fehlen.

## Anpassen auf eine andere Anlage

- Alle Entity-IDs (Shelly, Thermostate, Solar-Assistant-Sensoren) sind auf mein Haus bezogen. Die Tippfehler
  `bolier_3` sind bei mir echt und bleiben so.
- In `00` bei „Phase 1/2/3 Stabil" die Platzhalter `SERIAL_WR_...` durch die eigenen Seriennummern ersetzen.
- Die Strings der Solar-Assistant-Integration (`device_mode` = „Grid", `charger_source_priority` = „Solar only (OSO)")
  gelten für Voltronic-Typen; bei anderen Wechselrichtern prüfen.
- Schwellen (185/190 V Phasenschutz, 500 W Netz, Temperaturen, Spannungen) stehen jeweils mit Begründung im Kommentar.

## Feedback gesucht

- Ist der Spannungs-Akkuboden (51,2 / 52,5 V bei 16S) sinnvoll gewählt, oder würdet ihr den SOC über eine
  BMS-Bluetooth-Integration (BMS_BLE-HA) bevorzugen?
- Netz-Notlauf pro Phase: gibt es einen Grund, das anders zu lösen als über „Wechselrichter-Modus oder Netzbezug"?
- Die Entscheidungslogik in `03` ist ein großes Jinja-Template mit Prioritäten. Wer hat das schon eleganter gelöst,
  ohne Python?
- Wechselrichter-Eigenbedarf nachts am Netz bei „nur Solar laden": Erfahrungen mit Voltronic-Typen?
- Alles, was euch beim Lesen als riskant auffällt.

## Status und Sicherheitshinweis

Stand 28.09.2026. Läuft seit Juni 2026, im September zwei Blackouts, deren Ursache (Wechselrichter nahmen das
Netz nicht an) behoben ist. Der erste Winter steht bevor. Das ist mein Haus und meine Verantwortung. Wer davon etwas
übernimmt, steuert damit Heizstäbe mit 8 kW und einen 32-kWh-Akku: Elektrik vom Fachmann, Sicherungen und
mechanische Temperaturbegrenzer sind Pflicht, die Software ist nur die zweite Ebene. Keine Gewähr.

## Lizenz

MIT, siehe `LICENSE`.
