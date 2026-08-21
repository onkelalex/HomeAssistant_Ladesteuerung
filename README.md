# Hausbatterie-Preisautomatik für Home Assistant

Dokumentation zum Blueprint **🔋 Hausbatterie Steuerung mit Wallbox & Strompreis**, Version **0.10.0**.

## Zweck

Der Blueprint steuert die beiden Hoymiles MS-A2 anhand des aktuellen Tibber-Strompreises. Er kann den Akku entweder dynamisch in den günstigsten verfügbaren Zeitfenstern oder unterhalb eines festen Preiswertes laden. Die gespeicherte Energie wird nur bei einer ausreichend hohen Preisdifferenz wieder freigegeben.

Zusätzlich berücksichtigt die Steuerung:

- manuelles Sofortladen,
- einen einstellbaren Ziel-SOC,
- das Laden des Autos an der Wallbox,
- eine separate Entladesperre,
- die Tibber-Preise in 15-Minuten-Intervallen,
- eine Mindestpreisdifferenz von 10 ct/kWh.

## Grundkonfiguration

| Einstellung | Empfohlener Wert |
|---|---:|
| Gesamtkapazität | 4,48 kWh |
| Ladeleistung | -1.200 W |
| Neutralleistung | 0 W |
| Planungshorizont | 24 Stunden |
| Entlade-Preisaufschlag | 0,10 €/kWh |
| Steuerungsmodus | `mqtt_ctrl` |
| Normalmodus | `general` |
| Wallboxstatus für Laden | `charging` |

Ein negativer Leistungswert lädt den Akku. `0 W` hält ihn neutral.

## Verwendete Geräteentitäten

| Funktion | Entität |
|---|---|
| Batteriemodus | `select.msa_xxxxxxxxxxxx_mqtt_select` |
| Batterieleistung | `number.msa_xxxxxxxxxxxx` |
| Gemeinsamer SOC | `sensor.msa_xxxxxxxxxxxx_state_of_charge_system` |
| Wallboxstatus | `sensor.easee_charge_up_status` |
| Aktueller Tibber-Preis | `sensor.xxxxxxxxxxxx_strompreis` |
| Tibber-Diagrammdaten | `sensor.xxxxxxxxxxxx_diagramm_datenexport` |

Der Diagramm-Datenexport muss im Attribut `data` Einträge mit diesem Aufbau bereitstellen:

```yaml
data:
  - start_time: "2026-08-21T08:00:00+02:00"
    price_per_kwh: 0.4300
```

Alle Preise werden in **€/kWh** verarbeitet. `0.16` entspricht 16 ct/kWh.

## Benötigte Helfer

### Preisautomatik

```yaml
input_boolean.akku_automatisches_preisladen
```

Das ist der Hauptschalter der gesamten Preissteuerung.

- **Aus:** keine preisabhängige Lade- oder Entladesteuerung.
- **An:** der dynamische oder feste Preismodus ist aktiv.

Wenn der Hauptschalter ausgeschaltet ist, geht der Akku grundsätzlich in den Normalmodus `general`. Wallbox und Entladesperre können ihn weiterhin neutral halten.

### Fixen Ladewert verwenden

```yaml
input_boolean.akku_fester_ladewert
```

Dieser Schalter wählt den Preismodus aus, hat aber nur bei eingeschalteter Preisautomatik eine Wirkung.

- **Aus:** dynamischer Modus.
- **An:** fester Preismodus.

### Sofortladen

```yaml
input_boolean.akku_sofort_laden
```

Startet unabhängig vom Strompreis sofort das Laden mit der eingestellten Ladeleistung. Sobald der Ziel-SOC erreicht ist, schaltet der Blueprint diesen Helfer automatisch aus.

Sofortladen besitzt die höchste Priorität und wird auch bei aktiver Wallbox oder Entladesperre ausgeführt.

### Entladung sperren

```yaml
input_boolean.akku_entladung_sperren
```

- **Aus:** Entladung kann abhängig von Preisautomatik und Wallbox freigegeben werden.
- **An:** der Akku wird auf `mqtt_ctrl` und `0 W` gesetzt.

Die Sperre verhindert nur das Entladen. Sofortladen und zulässiges Preisladen bleiben möglich.

### Ziel-Ladestand

```yaml
input_number.akku_ziel_ladestand
```

Legt fest, bis zu welchem SOC der Akku beim Sofortladen und beim automatischen Preisladen geladen wird.

### Fester Ladewert

Der vorhandene `input_number` für die Lade-Preisgrenze enthält den festen Preis in €/kWh.

Beispiel:

```text
0,16 €/kWh = 16 ct/kWh
```

Dieser Wert wird nur im festen Preismodus für die Ladeentscheidung verwendet.

## Zusammenspiel der beiden Preisschalter

| Preisautomatik | Fester Ladewert | Verhalten |
|---|---|---|
| Aus | Aus | Keine Preissteuerung; Normalmodus, sofern keine Sperre greift |
| Aus | An | Ebenfalls keine Preissteuerung; der Modusschalter allein startet nichts |
| An | Aus | Dynamische Auswahl der günstigsten Zeitfenster |
| An | An | Laden unterhalb des festen Ladewertes |

## Dynamischer Preismodus

Der dynamische Modus ist aktiv, wenn:

```text
Preisautomatik = An
Fester Ladewert = Aus
```

### Berechnung der benötigten Ladezeit

Der Blueprint berechnet zunächst die bis zum Ziel-SOC fehlende Energie:

```text
fehlende Energie = 4,48 kWh × (Ziel-SOC − aktueller SOC) / 100
```

Bei 1,2 kW Ladeleistung liefert ein 15-Minuten-Block rechnerisch:

```text
1,2 kW × 0,25 h = 0,30 kWh
```

Aus der fehlenden Energie ergibt sich die Anzahl der benötigten Viertelstunden. Der Wert wird immer auf den nächsten vollständigen Block aufgerundet.

Die tatsächliche Ladezeit kann wegen Ladeverlusten, Leistungsregelung und SOC-Auflösung etwas länger sein.

### Auswahl der Ladezeitfenster

1. Der Blueprint untersucht die verfügbaren Preise innerhalb des Planungshorizonts.
2. Er sucht die nächste Preisphase, die mindestens um den eingestellten Preisaufschlag über einem vorherigen günstigen Preis liegt.
3. Existiert eine solche Kostenspitze, wählt er die benötigten günstigsten Viertelstunden vor dieser Spitze.
4. Reicht die verbleibende Zeit nicht für eine vollständige Ladung, werden alle noch vorhandenen Blöcke vor der Spitze genutzt.
5. Gibt es keine ausreichend hohe Kostenspitze, wählt er trotzdem die benötigten günstigsten Viertelstunden des Planungshorizonts.
6. Sobald der Ziel-SOC erreicht ist, endet die Ladung.

Damit verhindert die 10-ct-Regel nicht das günstige Laden. Sie entscheidet hauptsächlich darüber, wann die gespeicherte Energie wieder freigegeben wird.

## Fester Preismodus

Der feste Modus ist aktiv, wenn:

```text
Preisautomatik = An
Fester Ladewert = An
```

Der Akku lädt, wenn alle folgenden Bedingungen erfüllt sind:

- aktueller Strompreis ist kleiner oder gleich dem eingestellten festen Ladewert,
- aktueller SOC liegt unter dem Ziel-SOC,
- die Wallbox verhindert das parallele Laden nicht.

Beispiel bei einem festen Ladewert von `0,16 €/kWh`:

| Strompreis | Verhalten |
|---:|---|
| 0,14 €/kWh | Laden |
| 0,16 €/kWh | Laden |
| 0,17 €/kWh | Nicht laden |

## Entladelogik

Der Akku wird nicht mit einer festen positiven Leistung entladen. Stattdessen wechselt der Blueprint bei ausreichend hohem Preis in den Normalmodus `general`. Die interne MS-A2-Regelung kann dann den Hausverbrauch aus dem Akku versorgen.

### Dynamischer Modus

Als Vergleichspreis verwendet der Blueprint den niedrigsten bekannten Preis der vergangenen 24 Stunden:

```text
Entladeschwelle = niedrigster Preis der letzten 24 Stunden + Preisaufschlag
```

### Fester Modus

Als Vergleichspreis verwendet der Blueprint den eingestellten festen Ladewert:

```text
Entladeschwelle = fester Ladewert + Preisaufschlag
```

### Beispiel

```text
Vergleichspreis: 0,24 €/kWh
Preisaufschlag:  0,10 €/kWh
Entladen ab:     0,34 €/kWh
```

Unterhalb der Entladeschwelle bleibt der Akku bei aktiver Preisautomatik in `mqtt_ctrl` mit `0 W`. Ab Erreichen der Schwelle wird `general` aktiviert.

Der Blueprint erlaubt keinen Preisaufschlag unter `0,10 €/kWh`.

## Wallbox-Verhalten

Wenn der Wallboxsensor den konfigurierten Status `charging` meldet:

- wird eine Entladung des Hausakkus verhindert,
- wird der Akku auf `mqtt_ctrl` mit `0 W` gesetzt,
- darf Preisladen nur stattfinden, wenn **Preisladen auch während Wallbox-Ladung erlauben** aktiviert ist,
- bleibt Sofortladen aufgrund seiner höheren Priorität aktiv.

Standardmäßig ist paralleles Preisladen während des Autoladens ausgeschaltet.

## Prioritäten

Die erste passende Regel gewinnt:

1. **Sofortladen:** bis zum Ziel-SOC mit der eingestellten Ladeleistung.
2. **Automatisches Preisladen:** dynamisch oder nach festem Ladewert.
3. **Wallbox lädt:** Hausakku neutral halten.
4. **Entladung gesperrt oder Preis nicht teuer genug:** Hausakku neutral halten.
5. **Keine Sperre beziehungsweise Preis teuer genug:** Normalmodus `general`.

Wichtig: Weil Ladeaktionen vor der Entladesperre ausgewertet werden, verhindert die Entladesperre keine Ladung.

## Reaktionszeiten und Auslöser

Der Blueprint wertet die Logik erneut aus bei:

- Änderung des Wallboxstatus,
- Änderung des Sofortladen-Schalters,
- Änderung der Preisautomatik,
- Wechsel zwischen dynamischem und festem Modus,
- Änderung der Entladesperre,
- Änderung des Ziel-SOC,
- Änderung des festen Ladewertes,
- Änderung des Akku-SOC,
- Änderung des aktuellen Strompreises,
- Aktualisierung der Tibber-Diagrammdaten,
- zusätzlich alle 30 Sekunden.

Das wiederholte Senden alle 30 Sekunden stellt sicher, dass der gewünschte Batteriemodus und Leistungswert erhalten bleiben.

## Fehler- und Sonderfälle

- Bei ungültigem SOC oder Ziel-SOC wird keine automatische Ladung gestartet.
- Bei ungültigem aktuellem Strompreis wird weder Preisladen noch preisabhängiges Entladen freigegeben.
- Im festen Modus muss auch der feste Preishelfer einen gültigen Zahlenwert liefern.
- Fehlen im dynamischen Modus verwertbare Diagrammdaten, kann kein dynamisches Ladefenster ausgewählt werden.
- Sind noch keine morgigen Preise vorhanden, plant der Blueprint nur mit den bereits verfügbaren Daten.
- Der Modusschalter allein aktiviert die Preisautomatik nicht.
- Die Entladesperre blockiert keine Ladeaktion.

## Empfohlene Dashboard-Karten

```yaml
type: grid
columns: 2
square: false
cards:
  - type: tile
    entity: input_boolean.akku_automatisches_preisladen
    name: Akku-Preisautomatik
    icon: mdi:battery-clock
    color: green

  - type: tile
    entity: input_boolean.akku_fester_ladewert
    name: Fester Ladewert
    icon: mdi:tune-vertical
    color: orange

  - type: tile
    entity: input_boolean.akku_sofort_laden
    name: Sofort laden
    icon: mdi:battery-charging
    color: blue

  - type: tile
    entity: input_boolean.akku_entladung_sperren
    name: Entladung sperren
    icon: mdi:battery-lock
    color: red

  - type: tile
    entity: input_number.akku_ziel_ladestand
    name: Ziel-Ladestand
    icon: mdi:battery-high

  - type: tile
    entity: input_number.DEINE_PREISGRENZE
    name: Fester Ladewert
    icon: mdi:currency-eur
```

`input_number.DEINE_PREISGRENZE` muss durch die tatsächliche Entitäts-ID deines vorhandenen Preisgrenzen-Helfers ersetzt werden.

## Empfohlene Bedienung

### Normaler automatischer Betrieb

```text
Preisautomatik: An
Fester Ladewert: Aus
Entladung sperren: Aus
```

### Laden nach fester Preisgrenze

```text
Preisautomatik: An
Fester Ladewert: An
Fester Preis: gewünschter Wert in €/kWh
Entladung sperren: Aus
```

### Akku vorübergehend nicht entladen

```text
Entladung sperren: An
```

### Unabhängig vom Preis sofort laden

```text
Sofortladen: An
Ziel-Ladestand: gewünschter SOC
```

## Blueprint-Datei

Die zu dieser Dokumentation gehörende Datei ist:

```text
Hausbatterie_Preisautomatik_v0.10.0.yaml
```
