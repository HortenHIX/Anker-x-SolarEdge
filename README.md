# Anker MAX AC – Optimalsteuerung neben SolarEdge + LG Chem (Node-RED / Home Assistant)

Dieser Flow bindet einen **Anker MAX AC** Speicher optimal in eine bestehende PV-Anlage mit
**SolarEdge**-Wechselrichter, SolarEdge-Stromzähler und **LG Chem**-Speicher ein.

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
SolarEdge/LG Chem SOC ────────────┼──►  Node-RED Entscheidungslogik  ──►  Anker MAX AC
Anker SOC ─────────────────────── ┘        (alle 30 Sekunden)             (laden/entladen/Modus)
```

Der Flow **liest nur** die SolarEdge-/LG-Chem-Sensoren aus – er greift nie in deren Regelung
ein, deren eigene Eigenverbrauchsoptimierung hat immer Vorrang. Der Anker-Speicher wird erst
aktiv gesteuert, wenn LG Chem seine Aufgabe nicht mehr abdecken kann:

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

## Voraussetzungen

- Home Assistant mit:
  - **SolarEdge**-Integration (Wechselrichter + Zähler), z. B. die offizielle
    SolarEdge-Cloud-Integration oder `solaredge_modbus`/`solaredge_modbus_multi`.
  - LG-Chem-Speicherdaten, die über SolarEdge/StorEdge in Home Assistant landen
    (SOC-Sensor, ggf. Lade-/Entladeleistung).
  - Einer Anker-Integration, die den Anker MAX AC in Home Assistant als Entities bereitstellt
    (SOC-Sensor sowie steuerbare Entities für Ladeleistung/Ausgangsleistung/Betriebsmodus),
    z. B. die Community-Integration [`ha-anker-solix`](https://github.com/thomluther/ha-anker-solix)
    oder eine gleichwertige Integration.
- Node-RED mit der Palette **`node-red-contrib-home-assistant-websocket`**, verbunden mit
  deiner Home-Assistant-Instanz (Long-Lived Access Token bzw. Add-on-Modus).

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

## Erweiterungsideen

- Dynamische Stromtarife (z. B. Tibber/aWATTar) einbeziehen: Anker gezielt bei günstigen
  Preisen laden bzw. bei hohen Preisen entladen, statt nur auf PV-Überschuss zu reagieren.
- Eigene HA-Sensoren (z. B. `input_text`/Template-Sensor) aus dem `Status`-Ausgang befüllen,
  um ein Dashboard/Verlauf in Home Assistant/Grafana aufzubauen.
- Wetterprognose einbeziehen, um den Anker vorausschauend zu entladen, bevor am nächsten Tag
  wieder PV-Überschuss erwartet wird.

## Fehlersuche

- **„Sensor fehlt“ im Node-Status / Warnung im Debug-Fenster:** Eine der drei Pflicht-Entities
  (`gridPower`, `lgSoc`, `ankerSoc`) liefert `unavailable`/`unknown` oder existiert nicht –
  Entity-ID in Home Assistant prüfen.
- **Befehle kommen nicht an:** `domain`/`service`/`entityId` in den drei Service-Call-Knoten
  unter *Entwicklerwerkzeuge → Dienste* in Home Assistant gegenprüfen und testweise dort
  manuell aufrufen.
- **Anker schaltet zu oft um:** `deadbandW`, `minPowerDeltaToResend` und
  `minCommandIntervalMs` erhöhen.
