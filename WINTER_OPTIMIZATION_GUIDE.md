# Winterbetrieb (Nov-Feb)

Tarif dieser Installation: 00-05 Uhr 0,16 €/kWh, 05-00 Uhr 0,30 €/kWh. Es gibt keine Spitzenlast-Zeit.

## Ablauf

| Zeit | LG Chem (SolarEdge) | Anker (diese Automation) |
|---|---|---|
| 00-05 Uhr | lädt per SolarEdge-Zeitplan vom Netz (manuell im SolarEdge einzustellen) | lädt mit 800 W, solange SOC < 90 % |
| 05-24 Uhr | regelt wie gewohnt | entlädt bei Netzbezug (> 60 W Totband), solange SOC > Reserve; lädt bei PV-Überschuss, wenn LG >= Schwelle |

Im Nachtfenster entlädt der Anker nie (sonst würde er in den vom Netz ladenden LG entladen).

## Helper (Einstellungen > Geräte & Dienste > Helfer)

- `input_number.winter_min_soc_reserve` - Anker-SOC, unter dem tagsüber nicht entladen wird (Standard 20 %)
- `input_number.winter_lg_full_threshold` - ab diesem LG-SOC darf der Anker tagsüber PV-Überschuss laden (Standard 80 %)

Das Sensor-Attribut `current_period` am Sensor "Anker Control Aktion" zeigt das aktive Zeitfenster.

## Grenzen / Hinweise

- Der Anker lädt per AC maximal mit 800 W, also höchstens ca. 4 kWh in den 5 Nachtstunden.
- Die Nachtladung lohnt sich, solange 0,16 €/kWh geteilt durch den Round-Trip-Wirkungsgrad unter 0,30 € bleibt (bei ca. 85-90 % Wirkungsgrad: ca. 0,18-0,19 €, also ja). Der Gewinn pro kWh ist klein, der Effekt liegt eher im niedrigeren Tagesbezug.
- Wintermonate sind in `is_winter` hart auf 11, 12, 1, 2 gesetzt. Laut Ihren Monatsdaten ist die PV schon im Oktober deutlich schwächer; bei Bedarf Monate ergänzen (z. B. 10 und 3).
- Der SolarEdge-Nachtzeitplan für den LG gilt meist ganzjährig. Im Sommer lädt der LG dann ebenfalls nachts vom Netz, der Anker nicht (Sommerlogik).
- Die Automation liest den Sensor im selben 15-s-Takt, in dem er berechnet wird; Befehle laufen daher bis zu 15 s verzögert.
