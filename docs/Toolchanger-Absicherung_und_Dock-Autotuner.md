# Toolchanger-Absicherung und Dock-Autotuner

Stand: 25.10.2026. Drucker: Voron Tallboi, Kalico, KTC (klipper-toolchanger-easy), 5 Tools (T0 bis T4) an EBB36,
Cartographer-Plugin 1.10.1, AFC.

## Überblick

| Baustein | Wo | Nutzen |
|---|---|---|
| Sicherer Error-Handler | `toolchanger/toolchanger-config.cfg` | Nach einem fehlgeschlagenen Pickup/Dropoff fährt das Shuttle entlang des Dock-Pfads aus dem Dock, statt diagonal zu fahren. Das Tool wird bei einem Fehler nicht aus dem Dock gezogen. |
| Mesh-Fade | `printer.cfg`, `[bed_mesh]` | Das Bed-Mesh verändert die Höhe der Toolwechsel-Fahrten am Dock nicht mehr. |
| `verify_change` in `tool_prepare` | ktc-gcode-preprocessor | Schlägt ein Pickup in `KTC_PREPARE_TOOLS` fehl, bricht der Druck mit einer Meldung ab, statt mit leerem Shuttle weiterzufahren. |
| Farben im `PRINT_START` | ktc-gcode-preprocessor | Die Tool-Zusammenfassung zeigt die Farbliste mit unterscheidbaren Farben. |
| Dock-Autotuner | ktc-dock-autotune | Findet die Park-Position eines Docks automatisch und setzt sie in die Mitte des Bereichs, in dem das Tool einrastet. |
| Installer | ktc-dock-autotune | Installiert das Modul, trägt Include und Update-Manager ein und startet die Dienste neu. |

Bereits früher geändert und committed: `macros.cfg` (`TC_SELECT_TOOL`-Fehlermeldung, `STRESSTEST`-Einrückung).

## 1. Sicherer Error-Handler

Die Standard-`error_gcode: PAUSE` endet in `_TOOLHEAD_PARK_PAUSE_CANCEL`: 1 mm Z hoch und eine diagonale Fahrt zur
Parkposition. Steckt das Shuttle dabei noch im Dock, bewegt sich das Tool mit. `_TOOLCHANGE_ERROR` ersetzt das:

- Es wartet, bis die Maschine steht (`M400`, 300 ms), und ruft `_TOOLCHANGE_ERROR_MOVE` auf.
- Dieses Makro liest bei jedem Fehler `params_pickup_path`, `params_dropoff_path`, `params_park_x/y/z`, `params_safe_y`,
  `params_path_speed` und `params_fast_speed` des Tools. Pfadänderungen gelten also nach einem `RESTART` sofort.
- Lock-Punkt ist der Wegpunkt mit `verify`, Release-Punkt der erste Wegpunkt mit Z mehr als 1 mm unter `park_z`.
- Es bestimmt den nächsten Wegpunkt zur aktuellen Position (Toleranz `variable_tolerance: 2.5` mm) und entscheidet:

| Situation | Aktion |
|---|---|
| Pickup, Tool nicht erkannt, vor oder am Lock-Punkt | Pickup-Pfad rückwärts, dann `params_safe_y` |
| Pickup, Tool erkannt, am oder hinter dem Lock-Punkt | Ausfahrt des Pickup-Pfads weiter |
| Dropoff, Tool noch erkannt, vor dem Release-Punkt | Dropoff-Pfad rückwärts (Tool bleibt) |
| Dropoff, Tool weg, am oder hinter dem Release-Punkt | Ausfahrt des Dropoff-Pfads weiter |
| Position weiter als 2,5 mm von jedem Wegpunkt, Zustand unklar | keine Bewegung, nur Meldung |
| Fehler außerhalb eines Toolwechsels | normales `PAUSE` |

- Danach pausiert es ohne weiteren Park-Move (`PAUSE X= Y= Z_MIN=`, `custom_park_dz` für diesen Aufruf auf 0).
- Während eines `DOCK_AUTOTUNE`-Scans (`printer.dock_autotune.active`) fährt es genauso aus dem Dock, meldet aber nichts
  und pausiert nicht. Der Scan macht mit dem nächsten Versuch weiter.

Voraussetzungen: genau ein `verify`-Wegpunkt im Pickup-Pfad, der Dropoff-Pfad fährt nach unten. Die Makros `PAUSE`
(mit `X=`, `Y=`, `Z_MIN=`) und `_CLIENT_VARIABLE` (mit `custom_park_dz`) kommen aus dem Mainsail/Fluidd-Client.

## 2. Mesh-Fade

`[bed_mesh]` hat jetzt `fade_start: 1`, `fade_end: 10`, `fade_target: 0`. Ohne `fade_end` wirkt das Mesh auf jeder Höhe,
also auch bei den Fahrten am Dock (Z ca. 266) und verschiebt dort die Z-Position. Mit Fade entspricht die Höhe am Dock
genau dem befohlenen Z.

## 3. `verify_change` in `tool_prepare`

Neue Option `verify_change` (Standard `True`, in `config/gcode-preprocessor.cfg`). Nach jedem `T<n>` in
`KTC_PREPARE_TOOLS` prüft der Prozessor:

- Toolchanger-Status `ready` und aktives Tool gleich dem angeforderten,
- kein pausierter Druck,
- kein AFC-Fehlerstatus.

Trifft eines davon nicht zu, bricht `KTC_PREPARE_TOOLS` mit der Meldung ab, dass das Tool nicht aufgenommen wurde und der
Druck abgebrochen wird. Ein Wiederaufnehmen ergibt keinen Sinn, solange der Fehler nicht behoben ist. 83 Tests OK.
Auf dem Pi: `git pull` im Preprocessor-Klon, dann `RESTART`.

## 4. Farben im `PRINT_START`

`token_replacer.py` und `tool_summary.py` wählen die Farbliste, die verschiedene Farben enthält (Extruder-Farben vor
Filament-Farben). Sind beide einfarbig, wird die Ausgabe übersprungen. Zusätzlich ist der Patch
`tool_summary_single_tool.patch` per `git am` eingebaut.

## 5. Dock-Autotuner (ktc-dock-autotune)

https://github.com/ChrisFo8390/ktc-dock-autotune (öffentlich, GPLv3, Branch `main`).
Klipper-Modul `dock_autotune.py` mit `DOCK_AUTOTUNE` und `DOCK_AUTOTUNE_INTERRUPT`.

### Messprinzip

Ein Versuch ist Pickup (`SELECT_TOOL`, die Detection am Lock-Punkt muss das Tool sehen) plus Dropoff (`UNSELECT_TOOL`),
`repeats`-mal. Ab dem aktuellen Wert wird je Achse in `step`-Schritten (0,1 mm) nach beiden Seiten verschoben, bis ein
Versuch scheitert. Ein Halbschritt verfeinert die Kante. Die Fenstermitte wird nochmals geprüft und dann übernommen. Ist
eine Seite bis zur Range nicht gescheitert, bleibt die Achse unverändert. Die Achsen laufen nacheinander (x, y, z).

- Gescannt wird im Koordinatensystem eines echten Toolwechsels (der Toolchanger setzt die Tool-Offsets während des
  Wechsels zurück), die Werte gelten also im Druck.
- Messsignal ist der Detection-Pin. Ein ADXL kann zusätzlich mitgeschrieben werden (`ADXL=1`, `abort_g`), entscheidet aber
  nie.
- Ergebnis: Die neuen Werte gelten bis zum Neustart und werden als `params_park_*`-Zeilen ausgegeben, die in
  `toolchanger/tools/T<n>.cfg` gehören. Jeder Versuch steht in `~/printer_data/logs/dock_autotune.jsonl`.

### Sicherheitsnetz

- Geschwindigkeit ×0,5 und `ACCEL` höchstens 2000 während des Scans. Danach sind Geschwindigkeit, Beschleunigung und
  Park-Werte wiederhergestellt, auch nach einem Fehler oder Abbruch.
- Die Baseline (aktueller Wert) muss rasten, sonst Abbruch ohne Änderung.
- Nach einem fehlgeschlagenen Pickup: `INITIALIZE_TOOLCHANGER`, dann Prüfung (Shuttle leer, kein Tool erkannt, Shuttle bei
  `params_safe_y`). Sonst Abbruch.
- Pausiert ein Error-Handler, der den Scan-Modus nicht kennt, bricht der Scan ab.
- Nicht während eines laufenden oder pausierten Drucks. `max_cycles` (Standard 400) begrenzt die Pickups.
- `DOCK_AUTOTUNE_INTERRUPT` stoppt nach dem aktuellen Versuch, `M112` sofort.
- Ein Scan dauert je Achse höchstens `2 * range / step + 4` Versuche (`DRY_RUN=1` zeigt die Zahl), geschätzt 15 bis
  30 Minuten pro Dock (nicht gemessen).

### Vorbereitung wie `PRINT_START` (per Prompt)

Ohne `PREPARE` erkennt das Modul die Probe und zeigt in Mainsail/Fluidd/KlipperScreen einen Prompt:

- Erkannt werden Cartographer (`CARTOGRAPHER_TOUCH_HOME`), Beacon (`home_method: contact`, Touch ist `G28 Z`), Eddy,
  normale Probe oder Z-Endstop, sowie QGL (`QUAD_GANTRY_LEVEL`) oder Z-Tilt.
- Der Prompt zeigt Probe, Touch-Befehl und alle Schritte. Buttons: "Prepare and start (bed 70 C)", -10/-5/+5/+10 für die
  Bett-Temperatur (30 °C bis `max_temp`), "The machine is already prepared", "Cancel". Alle übrigen Parameter bleiben
  erhalten.

Schritte der Vorbereitung:

1. `INITIALIZE_TOOLCHANGER`. Ein Tool auf dem Shuttle wird abgelegt (bei nichts Gehomtem zuerst `G28`).
2. `M190` auf die Bett-Temperatur (Standard 70 °C, `SOAK=<s>` wartet länger), `G28`.
3. `QUAD_GANTRY_LEVEL` (oder `Z_TILT_ADJUST`), `BED_MESH_CLEAR`.
4. Touch-Homing mit T0 (Tool per `SELECT_TOOL`, nicht per `T0`, also ohne AFC): Düse auf `touch_temp`, `before_touch`
   (z. B. `BRUSH`), `touch_repeats`-mal der Touch-Befehl, Düse aus, T0 zurück ins Dock.
   Ohne Touch-Homing: `G28 Z` nach der QGL.
5. Dann beginnt der Scan. Ein Fehler in einem Schritt stoppt alles mit einem Report.

Das Touch-Homing läuft immer mit T0, auch wenn ein anderes Dock getunt wird. Die Referenz hängt vom Homing-Verfahren ab
(Touch mit der Düse oder Scan mit der Probe, gemessen rund 0,25 mm Unterschied), deshalb wird im selben Zustand getunt wie
im Druck.

Geht Homing ohne Tool nicht (`tool_probe`, Z-Endstop `probe:` ohne Probe-Sektion, altes `[scanner]`-Modul), zeigt der
Prompt die Meldung und bietet nur "weiter, Maschine ist vorbereitet" oder "Schließen". `PREPARE=1` wird dann abgelehnt.

### Parameter

| Parameter | Bedeutung |
|---|---|
| `T=<n>` | Tool, Pflicht |
| `AXES=xyz` | zu tunende Achsen |
| `STEP`, `RANGE`, `RANGE_X/Y/Z`, `REPEATS`, `SPEED` | Schrittweite, Bereich, Pickups je Kandidat, Geschwindigkeitsfaktor |
| `PREPARE=1\|0` | vorbereiten und scannen / Maschine ist vorbereitet, ohne Prompt |
| `BED=<°C>`, `SOAK=<s>`, `TOUCH=0` | Bett-Temperatur, Wartezeit, Touch-Homing aus |
| `ADXL=1` | Beschleunigungssensor des Tools aufzeichnen (`[adxl345 T<n>]`) |
| `DRY_RUN=1` | Erkennung und Plan anzeigen, nichts bewegen |

Config `dock_autotune.cfg`: `step`, `range_x/y/z`, `repeats`, `refine`, `speed_factor`, `accel`, `settle`, `max_cycles`,
`bed_temp` (70), `bed_soak`, `touch_command` (`auto`, `none` oder eigene Zeile), `touch_tool` (0), `touch_temp` (150,
bei dir 148), `touch_repeats` (2), `before_touch` (bei dir `BRUSH`), `accelerometer`, `abort_g`, `log_file`.

Beispiele:

```
DOCK_AUTOTUNE T=1 DRY_RUN=1              # Plan anzeigen
DOCK_AUTOTUNE T=1 AXES=z RANGE=0.4       # erster Lauf: eine Achse, kleiner Bereich
DOCK_AUTOTUNE T=1                        # x, y, z mit den konfigurierten Bereichen
DOCK_AUTOTUNE T=1 BED=100                # Prompt überspringen, Bett 100 °C
DOCK_AUTOTUNE_INTERRUPT                  # stoppt nach dem aktuellen Versuch
```

Ob die Dock-Z-Position von der Betttemperatur abhängt, lässt sich prüfen: dasselbe Dock einmal mit `BED=60` und einmal
mit `BED=100` scannen und die `params_park_z`-Werte vergleichen.

## 6. Installer

`install.sh` im Repo:

- verlinkt `klipper/extras/dock_autotune.py` in Klipper und kopiert `dock_autotune.cfg` (eine vorhandene Datei bleibt),
- trägt `[include dock_autotune.cfg]` in die `printer.cfg` ein (über dem `SAVE_CONFIG`-Block, Zeilenenden bleiben),
- trägt `[update_manager ktc-dock-autotune]` in die `moonraker.conf` ein,
- startet Klipper und, falls die `moonraker.conf` geändert wurde, Moonraker neu (Moonraker liest seine Config nur beim Start),
- legt vor jeder Änderung ein Backup an. Ein zweiter Lauf ändert nichts Vorhandenes.

Optionen: `--no-include`, `--no-update-manager`, `--no-restart`, `--uninstall` (entfernt Link und Include-Zeile).
`KLIPPER_PATH=…` und `CONFIG_PATH=…` ändern die Pfade.

```bash
cd ~/ktc-dock-autotune && git pull && ./install.sh
# oder frisch:
git clone https://github.com/ChrisFo8390/ktc-dock-autotune.git ~/ktc-dock-autotune && ~/ktc-dock-autotune/install.sh
```

Manuelle Kopien liegen in `C:\Users\Admin\Downloads\dock_autotune_for_printer` (mit `touch_temp: 148`,
`before_touch: BRUSH`). Fehlt der Update-Manager-Eintrag in Mainsail: Moonraker neu starten, sonst
`~/printer_data/logs/moonraker.log` nach `update_manager` durchsuchen.

## 7. Auf den Drucker bringen

| Was | Wie |
|---|---|
| `toolchanger-config.cfg`, `printer.cfg` | aus `C:\Users\Admin\Downloads` bzw. dem Tallboi-Repo, dann `RESTART`. Die Datei in `Downloads` enthält schon den Scan-Modus. |
| Preprocessor (`tool_prepare`) | `git pull` im Klon, `RESTART` |
| Autotuner | `git pull && ./install.sh`, danach `DOCK_AUTOTUNE T=1 DRY_RUN=1` |

## 8. Inbetriebnahme

1. Park-Position des Docks von Hand prüfen (`DOCK_TUNER`), das Tool muss sauber einrasten. Werte in `T<n>.cfg`
   übernehmen, `RESTART`.
2. Pickup- und Dropoff-Pfad prüfen (ein `verify`-Wegpunkt, Dropoff fährt nach unten). Nach jeder Pfadänderung neu tunen.
3. `DOCK_AUTOTUNE T=<n> DRY_RUN=1`, dann `DOCK_AUTOTUNE T=<n> AXES=z RANGE=0.4`. Am Drucker stehen, `M112` in Reichweite.
   Erst Z, dann X und Y. Mit T0 anfangen, weil dessen Dock die Z-Referenz bestimmt.
4. Ausgegebene `params_park_*`-Zeilen in `T<n>.cfg` eintragen. Zum Schluss einen `PRINT_START` mit zwei Tools fahren und den
   Abbruch bei einem absichtlich fehlenden Tool testen.

## 9. Geschwindigkeiten beim Toolwechsel

- `params_fast_speed` (13200 = 220 mm/s) und `params_path_speed` (2100 = 35 mm/s, der Kommentar "20 mm/s" ist falsch) sind
  nur die Wunschgeschwindigkeit `F`. Sie überschreiben die Limits aus der `printer.cfg` nicht. Klipper begrenzt jede Fahrt
  mit `max_velocity` (950) und `max_accel` (12000), bei Z-Anteil zusätzlich mit `max_z_velocity` (200) und `max_z_accel`
  (350), skaliert mit dem Z-Anteil.
- Bei kurzen Z-Legs ist `max_z_accel: 350` der Engpass: Spitzengeschwindigkeit etwa √(350 mm/s² · Strecke), also bei 5 mm
  rund 42 mm/s. Mehr `path_speed` bringt dort kaum etwas. Die 90°-Ecken im Pfad bremsen zusätzlich.
- Testen: `SET_VELOCITY_LIMIT Z_ACCEL=… Z_VELOCITY=…` (gilt bis zum Neustart). Dauerhaft in `[printer]`
  (`max_z_accel`, `max_z_velocity`), das wirkt dann auch im Druck.
- Wege zwischen den Docks: `params_fast_speed` (bis `max_velocity`, KTC-Standard 30000) und `max_accel`. Mit leerem
  Shuttle und auf Schrittverluste achten (sensorloses X/Y).
- Reihenfolge: erst Positionen tunen (langsam), dann Z-Geschwindigkeit, dann die Wege zwischen den Docks, jeweils mit
  `STRESSTEST`. Rastet ein Dock bei höherer Geschwindigkeit nicht mehr zuverlässig, mit `DOCK_AUTOTUNE T=<n> SPEED=1`
  nachstellen. Der Error-Handler skaliert mit, weil er dieselben Parameter liest.
- Geprüft gegen den Kalico-Code und die Standard-Makros von KTC. Die `toolchanger.cfg` auf dem Pi liegt außerhalb des
  Repos und könnte abweichen.

## 10. Teststand und offene Punkte

- Das Modul wurde nur in einer Simulation (echter Kalico-`gcode.py`/`gcode_move.py`, echter `toolchanger.py`, Dock-Modell)
  und mit 59 Unit-Tests geprüft. Auf einem echten Drucker lief es noch nie. Der Error-Handler wurde ebenfalls nur
  simuliert.
- `install.sh` wurde in Git Bash mit Test-Verzeichnissen geprüft (LF und CRLF, zweiter Lauf, Uninstall), nicht auf dem Pi.
  Den echten `systemctl`-Neustart und den Symlink-Zweig habe ich unter Windows nicht ausführen können.
- Dass `CARTOGRAPHER_TOUCH_HOME` erkannt wird, folgt aus der Plugin-Quelle, nicht aus deinem Drucker (`DRY_RUN=1` zeigt es).
- Ob dein HelixScreen die Prompts anzeigt, ist offen (Mainsail sicher). Sonst `PREPARE=1` bzw. `PREPARE=0` verwenden.
- ADXL: Ungenannte `[adxl345]`-Sektionen in den fünf Tool-Configs werden zu einem Chip zusammengeführt, wirksam ist nur
  EBB4. Für `ADXL=1` müssen sie `[adxl345 T0]` bis `[adxl345 T4]` heißen (und `resonance_tester` entsprechend).
- Der Scan nutzt `SELECT_TOOL` und `UNSELECT_TOOL` direkt. AFC sieht nur die Extruder-Aktivierungen und kann das gescannte
  Tool als aktive Lane übernehmen. Vor dem nächsten Druck AFC- und KTC-Zustand prüfen.
