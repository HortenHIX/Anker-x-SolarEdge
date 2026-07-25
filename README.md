# Anker MAX AC – Optimalsteuerung neben SolarEdge + LG Chem (Home Assistant)

Bindet einen **Anker MAX AC** Speicher optimal in eine bestehende PV-Anlage mit
**SolarEdge**-Wechselrichter, SolarEdge-Stromzähler und **LG Chem**-Speicher ein.

Es gibt zwei gleichwertige Umsetzungen – beide implementieren dieselbe Steuerungslogik,
wähle die für dich passende:

| Variante | Datei(en) | Wann sinnvoll |
|---|---|---|
| **Node-RED** | `node-red/anker_max_optimal_control.json` | Node-RED ist bereits im Einsatz; visuelle Flow-Bearbeitung/Debugging bevorzugt |
| **Native Home-Assistant-YAML** (ohne Node-RED) | `homeassistant/packages/anker_max_control.yaml` | Keine zusätzliche Abhängigkeit gewünscht; alles direkt in Home Assistant |

Die Logik, Grenzwerte und Sicherheitsmechanismen sind in beiden Varianten identisch (siehe
unten); es unterscheidet sich nur die technische Umsetzung.

## Ausgangslage

- SolarEdge (mit LG Chem Speicher, per StorEdge) regelt Laden/Entladen des LG-Chem-Speichers
  bereits selbständig auf maximalen Eigenverbrauch.
- Der Anker MAX AC kann **nicht** mit SolarEdge/LG Chem kommunizieren und reagiert **nicht**
  automatisch auf PV-Überschuss oder Netzbezug.
- Beide Systeme sind aber in Home Assistant eingebunden. Home Assistant (bzw. dieser
  Node-RED-Flow) ist damit die einzige Stelle, an der eine Koordination möglich ist.

## Funktionsprinzip

```
SolarEdge Zähler (Netzleistung) ──┐
SolarEdge/LG Chem SOC ────────────┼──►   Entscheidungslogik   ──►  Anker MAX AC
Anker SOC ─────────────────────── ┘   (alle 30 Sekunden geprüft)   (laden/entladen/Modus)
```

Die Logik **liest nur** die SolarEdge-/LG-Chem-Sensoren aus – sie greift nie in deren
Regelung ein, deren eigene Eigenverbrauchsoptimierung hat immer Vorrang. Der
Anker-Speicher wird erst aktiv gesteuert, wenn LG Chem seine Aufgabe nicht mehr abdecken kann:

1. **Laden des Anker:** Es fließt PV-Überschuss ins Netz (Einspeisung) **und** LG Chem ist
   bereits (nahezu) voll (SOC ≥ `lgFullSoc`, Standard 95 %) → der übrige Überschuss wird zum
   Laden des Anker verwendet (begrenzt auf dessen SOC-Obergrenze und maximale Ladeleistung).
2. **Entladen des Anker:** Es wird Strom aus dem Netz bezogen **und** LG Chem ist bereits
   (nahezu) leer (SOC ≤ `lgEmptySoc`, Standard 10 %) → der Anker deckt den restlichen Bedarf
   (begrenzt auf dessen SOC-Untergrenze und maximale Entladeleistung).
3. **Sonst (idle):** LG Chem deckt die Situation selbst ab → der Anker bleibt im
   Eigenverbrauchs-/Ruhemodus und wird nicht angesteuert.

Zusätzliche Schutzmechanismen:

- **SOC-Grenzen** (`ankerMinSoc`/`ankerMaxSoc`): Der Anker wird nie unter 10 % entladen bzw.
  nie über 95 % geladen (Zellschonung), unabhängig von allen anderen Bedingungen.
- **Totband** (`deadbandW`, Standard 60 W): Sehr kleine Überschüsse/Defizite lösen keine
  Aktion aus, damit die Steuerung nicht ständig hin- und herschaltet.
- **Hysterese/Mindestabstand** (`minCommandIntervalMs`, Standard 90 s): Befehle an den Anker
  werden nur bei relevanter Änderung und mit Mindestabstand gesendet, um die Cloud-API des
  Anker-Geräts nicht zu überlasten (Anker wird nicht lokal, sondern über die Cloud
  angesteuert).
- **Sicherheits-Override:** Wird die SOC-Grenze erreicht, wird sofort (ohne auf den
  Mindestabstand zu warten) auf 0 W gesetzt.
- **Auffrischung** (`forceResendMs`, Standard 10 min): Der aktuelle Befehl wird auch ohne
  Änderung regelmäßig erneut gesendet, falls z. B. die Cloud-Verbindung des Anker-Geräts
  zwischenzeitlich unterbrochen war.
- **Fail-Safe:** Sind Pflicht-Sensoren `unavailable`/`unknown` (z. B. Home Assistant startet
  gerade neu), wird **keine** Aktion ausgeführt und stattdessen eine Warnung/Benachrichtigung
  erzeugt.

## Voraussetzungen (beide Varianten)

- Home Assistant mit:
  - **SolarEdge**-Integration (Wechselrichter + Zähler), z. B. die offizielle
    SolarEdge-Cloud-Integration oder `solaredge_modbus`/`solaredge_modbus_multi`.
  - LG-Chem-Speicherdaten, die über SolarEdge/StorEdge in Home Assistant landen
    (SOC-Sensor, ggf. Lade-/Entladeleistung).
  - Einer Anker-Integration, die den Anker MAX AC in Home Assistant als Entities bereitstellt
    (SOC-Sensor sowie steuerbare Entities für Ladeleistung/Ausgangsleistung/Betriebsmodus),
    z. B. die Community-Integration [`ha-anker-solix`](https://github.com/thomluther/ha-anker-solix)
    oder eine gleichwertige Integration.
- Nur für die Node-RED-Variante zusätzlich: Node-RED mit der Palette
  **`node-red-contrib-home-assistant-websocket`**, verbunden mit deiner
  Home-Assistant-Instanz (Long-Lived Access Token bzw. Add-on-Modus).

---

# Variante A: Node-RED

## Installation

1. In Node-RED: Menü → *Import* → Datei `node-red/anker_max_optimal_control.json` auswählen
   und in einen neuen Tab importieren.
2. Den Config-Knoten **„Home Assistant“** doppelklicken und mit deiner HA-Instanz verbinden
   (falls noch kein Server-Knoten existiert bzw. einen vorhandenen wiederverwenden).
3. Den Funktionsknoten **„Optimale Steuerung berechnen“** öffnen und im Codeblock
   `ENTITIES` die Entity-IDs an deine Installation anpassen (siehe Tabelle unten).
4. Die drei Service-Call-Knoten **„Anker: Ladeleistung setzen“**, **„Anker: Entladeleistung
   setzen“** und **„Anker: Betriebsmodus setzen“** öffnen und jeweils `entityId` (und bei
   Bedarf `domain`/`service`) auf die tatsächlichen Entities deiner Anker-Integration setzen.
5. Grenzwerte im Codeblock `LIMITS` an dein Anker-MAX-AC-Modell (Datenblatt: max.
   Lade-/Entladeleistung) sowie an deine Präferenzen anpassen.
6. Deployen. Der Flow prüft danach automatisch alle 30 Sekunden den Zustand der Anlage
   (Inject-Knoten „Alle 30s prüfen“); zum Testen steht zusätzlich „Manuell auslösen“ bereit.

## Zu konfigurierende Entities

| Konfigurationsschlüssel | Bedeutung | Vorkommen im Flow |
|---|---|---|
| `ENTITIES.gridPower` | Netzleistung (SolarEdge-Zähler), + = Bezug / − = Einspeisung | Funktionsknoten |
| `ENTITIES.gridPowerSign` | `1` oder `-1`, falls dein Sensor umgekehrtes Vorzeichen liefert | Funktionsknoten |
| `ENTITIES.lgSoc` | Ladestand (%) des LG-Chem-Speichers | Funktionsknoten |
| `ENTITIES.ankerSoc` | Ladestand (%) des Anker MAX AC | Funktionsknoten |
| `number.anker_solarbank_charge_power_limit` (Platzhalter) | Steuerbare Ladeleistung des Anker | Knoten „Anker: Ladeleistung setzen“ |
| `number.anker_solarbank_output_power_preset` (Platzhalter) | Steuerbare Ausgangs-/Entladeleistung des Anker | Knoten „Anker: Entladeleistung setzen“ |
| `select.anker_solarbank_usage_mode` (Platzhalter) | Betriebsmodus-Auswahl des Anker (z. B. „Manual“/„Self Consumption“) | Knoten „Anker: Betriebsmodus setzen“ |

> Die genauen Entity- und Service-Namen hängen von der jeweils verwendeten Anker-Integration
> und Firmware-Version ab. Prüfe in Home Assistant unter *Entwicklerwerkzeuge → Zustände*
> bzw. *→ Dienste*, wie deine Entities tatsächlich heißen, und trage sie entsprechend ein.
> Falls deine Integration keinen eigenen Betriebsmodus-Schalter besitzt, kannst du den Knoten
> „Anker: Betriebsmodus setzen“ einfach unverbunden lassen bzw. löschen.

## Wichtige Parameter (im Funktionsknoten, Block `LIMITS`)

| Parameter | Standard | Bedeutung |
|---|---|---|
| `ankerMinSoc` / `ankerMaxSoc` | 10 % / 95 % | Zellschutz-Grenzen des Anker |
| `ankerMaxChargePower` / `ankerMaxDischargePower` | 1200 W / 1800 W | Leistungsgrenzen laut Datenblatt |
| `lgFullSoc` / `lgEmptySoc` | 95 % / 10 % | Ab wann LG Chem als „voll“/„leer“ gilt |
| `deadbandW` | 60 W | Totband gegen Flattern |
| `minCommandIntervalMs` | 90 000 ms | Mindestabstand zwischen Cloud-Befehlen |
| `minPowerDeltaToResend` | 50 W | Nötige Änderung, damit ein Befehl erneut gesendet wird |
| `forceResendMs` | 600 000 ms | Intervall für automatische Auffrischung des Befehls |

## Fehlersuche (Node-RED)

- **„Sensor fehlt“ im Node-Status / Warnung im Debug-Fenster:** Eine der drei Pflicht-Entities
  (`gridPower`, `lgSoc`, `ankerSoc`) liefert `unavailable`/`unknown` oder existiert nicht –
  Entity-ID in Home Assistant prüfen.
- **Befehle kommen nicht an:** `domain`/`service`/`entityId` in den drei Service-Call-Knoten
  unter *Entwicklerwerkzeuge → Dienste* in Home Assistant gegenprüfen und testweise dort
  manuell aufrufen.
- **Anker schaltet zu oft um:** `deadbandW`, `minPowerDeltaToResend` und
  `minCommandIntervalMs` erhöhen.

---

# Variante B: Native Home-Assistant-YAML (ohne Node-RED)

Setzt dieselbe Logik als **Home-Assistant-Package** um: Helper-Entities zur
Zustandsspeicherung, Template-Sensoren zur Berechnung und eine Automation, die die
Befehle an den Anker sendet. Keine zusätzliche Software nötig – läuft komplett in Home
Assistant.

Datei: [`homeassistant/packages/anker_max_control.yaml`](homeassistant/packages/anker_max_control.yaml)

## Installation

1. Falls noch nicht vorhanden, in `configuration.yaml` **Packages** aktivieren:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Die Datei `homeassistant/packages/anker_max_control.yaml` aus diesem Repository nach
   `<dein-ha-config-verzeichnis>/packages/anker_max_control.yaml` kopieren.
3. Im Abschnitt **„ANPASSEN“** (oben im `template:`-Block) die drei Entity-IDs
   `grid_power_entity`, `lg_soc_entity`, `anker_soc_entity` sowie ggf. `grid_power_sign`
   an deine Installation anpassen.
4. Im Abschnitt **„ANPASSEN“** der Automation (`sequence:` der drei `choose:`-Zweige) die
   `entity_id`s der Anker-Steuer-Entities (`number.anker_solarbank_charge_power_limit`,
   `number.anker_solarbank_output_power_preset`, `select.anker_solarbank_usage_mode`) auf
   die tatsächlichen Entities deiner Anker-Integration setzen.
5. Bei Bedarf die Grenzwerte (`min_soc`, `max_soc`, `lg_full_soc`, `lg_empty_soc`,
   `deadband_w`, `max_charge_power`, `max_discharge_power`) direkt im Template anpassen.
6. **Home Assistant neu starten** (Packages werden nur beim Neustart eingelesen, nicht
   durch „YAML neu laden“).

Nach dem Neustart erscheinen automatisch folgende neue Entities:

- `sensor.anker_control_aktion` (`charge`/`discharge`/`idle`/`unavailable`, mit Attributen
  `grid_power_w`, `lg_soc`, `anker_soc`, `can_charge`, `can_discharge`)
- `sensor.anker_control_ladeleistung_soll` / `sensor.anker_control_entladeleistung_soll` (W)
- Helper `input_datetime.anker_control_last_command`, `input_text.anker_control_last_action`,
  `input_number.anker_control_last_charge_target` / `..._last_discharge_target` (interne
  Zustandsspeicherung für Hysterese/Rate-Limiting, nicht manuell verändern)
- Automation **„Anker Control: Entscheidung anwenden“**

## Zu konfigurierende Entities

| Schlüssel/Bereich | Bedeutung | Ort |
|---|---|---|
| `grid_power_entity` | Netzleistung (SolarEdge-Zähler), + = Bezug / − = Einspeisung | `template:` → `variables:` |
| `grid_power_sign` | `1` oder `-1`, falls Vorzeichen umgekehrt ist | `template:` → `variables:` |
| `lg_soc_entity` | Ladestand (%) des LG-Chem-Speichers | `template:` → `variables:` |
| `anker_soc_entity` | Ladestand (%) des Anker MAX AC | `template:` → `variables:` |
| `number.anker_solarbank_charge_power_limit` (Platzhalter) | Steuerbare Ladeleistung des Anker | Automation, `choose:`-Zweig „charge“/„idle“ |
| `number.anker_solarbank_output_power_preset` (Platzhalter) | Steuerbare Ausgangs-/Entladeleistung des Anker | Automation, `choose:`-Zweig „discharge“/„idle“ |
| `select.anker_solarbank_usage_mode` (Platzhalter) | Betriebsmodus-Auswahl des Anker | Automation, alle drei `choose:`-Zweige |

> Wie bei der Node-RED-Variante: Die genauen Entity-/Service-Namen hängen von deiner
> Anker-Integration ab – unter *Entwicklerwerkzeuge → Zustände* bzw. *→ Dienste* prüfen.
> Fehlt ein Betriebsmodus-Schalter, einfach die drei `select.select_option`-Schritte aus
> der Automation entfernen.

## Wichtige Parameter (im `template:`-Block, Abschnitt „Grenzwerte“)

| Parameter | Standard | Bedeutung |
|---|---|---|
| `min_soc` / `max_soc` | 10 % / 95 % | Zellschutz-Grenzen des Anker |
| `max_charge_power` / `max_discharge_power` | 1200 W / 1800 W | Leistungsgrenzen laut Datenblatt |
| `lg_full_soc` / `lg_empty_soc` | 95 % / 10 % | Ab wann LG Chem als „voll“/„leer“ gilt |
| `deadband_w` | 60 W | Totband gegen Flattern |
| *(fest im Automation-Code)* `min_interval_ok` | 90 s | Mindestabstand zwischen Cloud-Befehlen |
| *(fest im Automation-Code)* `significant_change` | 50 W | Nötige Änderung, damit ein Befehl erneut gesendet wird |
| *(fest im Automation-Code)* `due_for_refresh` | 600 s | Intervall für automatische Auffrischung des Befehls |

Die drei fest im Automation-Code stehenden Schwellenwerte (90 s / 50 W / 600 s) lassen
sich bei Bedarf direkt in den `variables:` der Automation ändern (Suche nach `>= 50`,
`>= 90`, `>= 600`).

## Fehlersuche (YAML-Variante)

- **Package wird nicht geladen:** `homeassistant: packages: !include_dir_named packages`
  in `configuration.yaml` prüfen, Pfad/Dateiname kontrollieren, Home Assistant neu starten
  (nicht nur „YAML neu laden“) und unter *Entwicklerwerkzeuge → YAML → Konfiguration
  prüfen* auf Fehler achten.
- **`sensor.anker_control_aktion` zeigt `unavailable`:** Eine der drei Pflicht-Entities
  (`grid_power_entity`, `lg_soc_entity`, `anker_soc_entity`) liefert `unavailable`/`unknown`
  oder die Entity-ID stimmt nicht – es erscheint zusätzlich eine persistente Benachrichtigung
  in Home Assistant.
- **Befehle kommen nicht an:** Die `service`/`entity_id`-Kombinationen in der Automation unter
  *Entwicklerwerkzeuge → Dienste* testweise manuell aufrufen; Automations-Traces
  (*Einstellungen → Automatisierungen → „Anker Control: Entscheidung anwenden“ → Traces*)
  zeigen, welcher Zweig ausgeführt wurde und wie `should_send` berechnet wurde.
- **Anker schaltet zu oft um:** Schwellenwerte wie oben beschrieben erhöhen.

---

## Erweiterungsideen (beide Varianten)

- Dynamische Stromtarife (z. B. Tibber/aWATTar) einbeziehen: Anker gezielt bei günstigen
  Preisen laden bzw. bei hohen Preisen entladen, statt nur auf PV-Überschuss zu reagieren.
- Den berechneten Status (Aktion, Zielleistungen, Sensorwerte) in einem Dashboard/Verlauf
  in Home Assistant oder Grafana visualisieren – in der YAML-Variante direkt über die
  vorhandenen `sensor.anker_control_*`-Entities, in Node-RED über den `Status`-Ausgang.
- Wetterprognose einbeziehen, um den Anker vorausschauend zu entladen, bevor am nächsten Tag
  wieder PV-Überschuss erwartet wird.
