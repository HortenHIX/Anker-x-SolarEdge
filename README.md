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
   bereits (nahezu) voll (SOC ≥ `lgFullSoc`, Standard 95 %) **und** lädt aktuell nicht mehr
   nennenswert (Batterieleistung ≤ `lgIdlePowerW`, Standard 50 W) → der übrige Überschuss
   wird zum Laden des Anker verwendet (begrenzt auf dessen SOC-Obergrenze und maximale
   Ladeleistung).
2. **Entladen des Anker:** Es wird Strom aus dem Netz bezogen **und** LG Chem ist bereits
   (nahezu) leer (SOC ≤ `lgEmptySoc`, Standard 10 %) **und** entlädt aktuell nicht mehr
   nennenswert (Batterieleistung ≥ `-lgIdlePowerW`) → der Anker deckt den restlichen Bedarf
   (begrenzt auf dessen SOC-Untergrenze und maximale Entladeleistung).
3. **Sonst (idle):** LG Chem deckt die Situation selbst ab → der Anker bleibt im
   Eigenverbrauchs-/Ruhemodus und wird nicht angesteuert.

Die zusätzliche Prüfung der tatsächlichen LG-Chem-Lade-/Entladeleistung (nicht nur des SOC)
verhindert, dass der Anker zu früh eingreift, während SolarEdge/LG Chem den Speicher noch
aktiv lädt oder entlädt – sonst würden beide Systeme um denselben Überschuss/Bedarf
konkurrieren.

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
4. Die drei Service-Call-Knoten **„Anker: Betriebsmodus setzen“**, **„Anker: Richtung
   setzen“** und **„Anker: Zielleistung setzen“** öffnen und jeweils `entityId` (und bei
   Bedarf `domain`/`service`) auf die tatsächlichen Entities deiner Anker-Integration setzen.
5. Grenzwerte im Codeblock `LIMITS` an dein Anker-MAX-AC-Modell (Datenblatt: max.
   Lade-/Entladeleistung) sowie an deine Präferenzen anpassen.
6. Deployen. Der Flow prüft danach automatisch alle 30 Sekunden den Zustand der Anlage
   (Inject-Knoten „Alle 30s prüfen“); zum Testen steht zusätzlich „Manuell auslösen“ bereit.

## Steuerkonzept

Viele Anker-SOLIX-Geräte (z. B. Solarbank Max AC über die Integration
[`ha-anker-solix`](https://github.com/thomluther/ha-anker-solix)) bieten einen eigenen
**„Drittanbieter-Steuerung“-Modus** (Rohwert: `third_party_control`) statt separater
Lade-/Entladeleistungs-Limits: Man wählt eine **Richtung** (`charge`/`discharge`) und gibt
eine **Ziel-Watt-Zahl** vor. Genau darauf ist dieser Flow ausgelegt. Falls deine
Anker-Integration ein anderes Steuerkonzept hat, müssen die drei Service-Call-Knoten
entsprechend umgebaut werden (siehe Kommentar im Funktionsknoten).

**Wichtig:** Die in der App/HA-UI angezeigten deutschen Texte (z. B. „Drittanbieter-Steuerung“,
„Netzbezug“) sind nur Anzeigenamen – an `select.select_option` müssen die zugrunde liegenden
Rohwerte gesendet werden. Diese findest du unter *Entwicklerwerkzeuge → Zustände* im
Attribut `options:` der jeweiligen Entity.

## Zu konfigurierende Entities

| Konfigurationsschlüssel | Bedeutung | Vorkommen im Flow |
|---|---|---|
| `ENTITIES.gridPower` | Netzleistung (SolarEdge-Zähler), + = Bezug / − = Einspeisung | Funktionsknoten |
| `ENTITIES.gridPowerSign` | `1` oder `-1`, falls dein Sensor umgekehrtes Vorzeichen liefert | Funktionsknoten |
| `ENTITIES.lgSoc` | Ladestand (%) des LG-Chem-Speichers | Funktionsknoten |
| `ENTITIES.lgPower` | Lade-/Entladeleistung (W) des LG-Chem-Speichers, + = lädt / − = entlädt | Funktionsknoten |
| `ENTITIES.ankerSoc` | Ladestand (%) des Anker MAX AC | Funktionsknoten |
| `select.anker_solarbank_usage_mode` (Platzhalter) | Betriebsmodus-Auswahl, Rohwert `third_party_control` | Knoten „Anker: Betriebsmodus setzen“ |
| `select.anker_solarbank_power_flow` (Platzhalter) | Richtung, Rohwerte `charge`/`discharge` | Knoten „Anker: Richtung setzen“ |
| `number.anker_solarbank_grid_power` (Platzhalter) | Ziel-Leistung in Watt | Knoten „Anker: Zielleistung setzen“ |

> Die genauen Entity- und Rohwert-Namen hängen von der jeweils verwendeten Anker-Integration
> und Firmware-Version ab. Prüfe in Home Assistant unter *Entwicklerwerkzeuge → Zustände*,
> wie deine Entities tatsächlich heißen und welche `options:` sie akzeptieren.

## Wichtige Parameter (im Funktionsknoten, Block `LIMITS`)

| Parameter | Standard | Bedeutung |
|---|---|---|
| `ankerMinSoc` / `ankerMaxSoc` | 10 % / 95 % | Zellschutz-Grenzen des Anker |
| `ankerMaxChargePower` / `ankerMaxDischargePower` | 1200 W / 1800 W | Leistungsgrenzen laut Datenblatt |
| `lgFullSoc` / `lgEmptySoc` | 95 % / 20 % | Großzügige SOC-Obergrenze für „voll“/„leer genug“ – die eigentliche Bestätigung liefert `lgIdlePowerW`, da LG Chem oft schon vorher (eigene Reserve-Einstellung) aufhört zu laden/entladen |
| `lgIdlePowerW` | 50 W | LG Chem gilt erst als gesättigt, wenn Lade-/Entladeleistung darunter liegt |
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
– bereits mit den realen Entity-IDs dieser Installation befüllt (SolarEdge Modbus Multi +
Anker SOLIX Solarbank Max AC über `ha-anker-solix`).

### Steuerkonzept des Anker SOLIX Solarbank Max AC

Diese Anker-Integration bietet einen eigenen **„Drittanbieter-Steuerung“-Modus**
(`third_party_control`), der genau für externe Automatisierung gedacht ist. Die Steuerung
läuft über drei native Felder statt über separate Lade-/Entlade-Leistungslimits:

| Feld (Anzeige in der App/HA) | Entity | Rohwerte |
|---|---|---|
| Betriebsmodus | `select.anker_solix_solarbank_max_ac_betriebsmodus_gerat_lauft_im_drittanbieter_steuermodus` | `third_party_control` / `custom_mode` |
| Leistungsfluss | `select.anker_solix_solarbank_max_ac_leistungsfluss` | `charge` / `discharge` |
| Netzleistung | `number.anker_solix_solarbank_max_ac_netzleistung` | Ziel-Watt |
| Entladegrenze | `number.anker_solix_solarbank_max_ac_entladegrenze` | % (Zellschutz, geräteseitig) |
| Ladeobergrenze | `number.anker_solix_solarbank_max_ac_ladeobergrenze` | % (Zellschutz, geräteseitig) |

**Wichtig:** Die in der App/HA-UI angezeigten deutschen Texte („Drittanbieter-Steuerung“,
„Netzbezug“ …) sind nur übersetzte Anzeigenamen – die Automation muss die englischen
Rohwerte (`third_party_control`, `charge`, `discharge`) an `select.select_option` senden,
sonst schlägt der Befehl fehl. Das ist in der Datei bereits korrekt hinterlegt.

## Installation

1. Falls noch nicht vorhanden, in `configuration.yaml` **Packages** aktivieren:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
2. Die Datei `homeassistant/packages/anker_max_control.yaml` aus diesem Repository nach
   `<dein-ha-config-verzeichnis>/packages/anker_max_control.yaml` kopieren.
3. **Einmalig** im Anker-Gerät (App oder HA-Steuerelemente) die SOC-Schutzgrenzen setzen:
   **Entladegrenze = 10 %**, **Ladeobergrenze = 95 %** (passend zu `min_soc`/`max_soc`
   unten in der Datei – bei dir standen sie zuletzt auf 5 %/100 %).
4. **Home Assistant neu starten** (Packages werden nur beim Neustart eingelesen, nicht
   durch „YAML neu laden“).
5. `grid_power_sign` ist für diese Installation bereits korrekt auf `-1` gesetzt (per
   Praxistest am 30.07.2026 bestätigt: `sensor.solaredge_i1_m1_ac_power` liefert negativ =
   Bezug, positiv = Einspeisung). Überträgst du die Datei auf eine andere Installation,
   ggf. erneut prüfen (siehe Kommentar in der Datei).

Nach dem Neustart erscheinen automatisch folgende neue Entities:

- `sensor.anker_control_aktion` (`charge`/`discharge`/`idle`/`unavailable`, mit Attributen
  `grid_power_w`, `lg_soc`, `anker_soc`, `can_charge`, `can_discharge`)
- `sensor.anker_control_ladeleistung_soll` / `sensor.anker_control_entladeleistung_soll` (W)
- Helper `input_datetime.anker_control_last_command`, `input_text.anker_control_last_action`,
  `input_number.anker_control_last_power` (interne Zustandsspeicherung für
  Hysterese/Rate-Limiting, nicht manuell verändern)
- Automation **„Anker Control: Entscheidung anwenden“**

## Zu konfigurierende Entities

| Schlüssel/Bereich | Bedeutung | Wert in dieser Installation |
|---|---|---|
| `grid_power_entity` | Netzleistung (SolarEdge-Zähler) | `sensor.solaredge_i1_m1_ac_power` |
| `lg_soc_entity` | Ladestand (%) des LG-Chem-Speichers | `sensor.solaredge_i1_b1_state_of_energy` |
| `lg_power_entity` | Lade-/Entladeleistung (W) des LG-Chem-Speichers, + = lädt / − = entlädt | `sensor.solaredge_i1_b1_dc_power` |
| `anker_soc_entity` | Ladestand (%) des Anker MAX AC | `sensor.anker_solix_solarbank_max_ac_soc` |
| Betriebsmodus (Automation) | s. Tabelle oben | fest auf `third_party_control` |
| Leistungsfluss (Automation) | s. Tabelle oben | `charge`/`discharge` je nach Aktion |
| Netzleistung (Automation) | s. Tabelle oben | berechnete Ziel-Watt |

> Falls du die Datei auf ein anderes Anker-Gerät/eine andere Installation überträgst,
> sind die Entity-IDs vermutlich anders benannt (Zahlen-Suffixe, Gerätename) – über
> *Entwicklerwerkzeuge → Zustände* mit Suchbegriff „anker“ bzw. „solaredge“ prüfen und in
> der Datei anpassen.

## Wichtige Parameter (im `template:`-Block, Abschnitt „Grenzwerte“)

| Parameter | Standard | Bedeutung |
|---|---|---|
| `min_soc` / `max_soc` | 10 % / 95 % | Zellschutz-Grenzen des Anker |
| `max_charge_power` / `max_discharge_power` | 1200 W / 1800 W | Leistungsgrenzen laut Datenblatt |
| `lg_full_soc` / `lg_empty_soc` | 95 % / 20 % | Großzügige SOC-Obergrenze für „voll“/„leer genug“ – die eigentliche Bestätigung liefert `lg_idle_power_w`, da LG Chem oft schon vorher (eigene Reserve-Einstellung) aufhört zu laden/entladen |
| `lg_idle_power_w` | 50 W | LG Chem gilt erst als gesättigt, wenn Lade-/Entladeleistung darunter liegt |
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
- **`sensor.anker_control_aktion` zeigt `unavailable`:** Eine der vier Pflicht-Entities
  (`grid_power_entity`, `lg_soc_entity`, `lg_power_entity`, `anker_soc_entity`) liefert
  `unavailable`/`unknown` oder die Entity-ID stimmt nicht – es erscheint zusätzlich eine
  persistente Benachrichtigung in Home Assistant.
- **Anker springt trotz `lg_full_soc`/`lg_empty_soc` nicht an:** Prüfen, ob
  `sensor.solaredge_i1_b1_dc_power` gerade über/unter `lg_idle_power_w` liegt – solange
  LG Chem noch aktiv lädt/entlädt, bleibt der Anker absichtlich inaktiv.
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
