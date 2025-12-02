# Node-RED: Automatische Lade-/Entlade-Steuerung für zweiten BYD-Akku

Dieses Projekt enthält eine Node-RED-Logik zur automatischen Steuerung eines zweiten BYD-Speichers
(z. B. kleiner WR an einer Ost/West-Anlage), basierend auf:

- Netzbezug / Einspeisung (Smart Meter)
- SoC von Haupt- und Nebenakku
- Lade-/Entladeleistung des Hauptakkus
- Zellspannung Akku 2
- Freigaben aus der Visualisierung (Gira)

Ziele:

- PV-Überschüsse in Akku 2 laden (CHG_SURPLUS)
- Grundlast und/oder Hauptakku beim Entladen unterstützen (DIS_BASE)
- sanfte Rampen + Hysterese
- BYD-Register **InWRte / OutWRte** sauber und ohne Modbus-Fehler beschreiben

## 1. Architektur

Der Flow besteht im Kern aus drei Function-Nodes plus ein paar ioBroker-I/O-Knoten:

1. **DIE 2. Akku Steuerung V4**  
   _Hauptlogik_
   - arbeitet auf Watt-Basis (`P_set_aux` in W)
   - State-Machine (`IDLE`, `CHG_SURPLUS`, `DIS_BASE`, `FREEZE`)
   - berücksichtigt:
     - Netzverknüpfungspunkt (`P_grid`)
     - SoC Hauptakku (`BYD_SoC`)
     - SoC Akku 2 (`BYD7.7_SoC`)
     - Lade-/Entladeleistung Haupt- und Nebenakku
     - Freigaben & Zellspannung

2. **DIE Ausgangs Logik V2**  
   _BYD-Ausgangslogik_
   - Input: `P_set_aux` (W)
   - Output: **InWRte / OutWRte** in % (1 Nachkommastelle)
   - implementiert BYD-Logik:
     - Laden erzwingen:    `InWRte > 0`, `OutWRte < 0`
     - Entladen erzwingen: `InWRte < 0`, `OutWRte > 0`
   - sorgt für korrekte Reihenfolge + Delay, damit keine „Illegal Value“-Fehler auftreten

3. **Visu Status Text** + **OstWest_Funktion_Status_txt**
   - Debug-Objekt → kompakter Status-String für Gira (CO@…)
   - send-by-change, damit die Visu nicht gespammt wird

Dazu kommen:

- `ioBroker in` für Freigaben & Parameter aus Gira / Modbus
- `ioBroker out` auf die BYD-Register `40365_OutWRte` und `40366_InWRte`
- ein `inject`-Node, der alle 3 s triggert

## 2. Flow-Import

Der komplette Flow liegt als Node-RED-JSON (Gruppe „Automatische Lade / Entlade steuerung“)
mit u. a.:

- `DIE 2. Akku Steuerung V4` (Function)
- `DIE Ausgangs Logik V2` (Function)
- Delay-Nodes für `msg.delay`
- ioBroker I/O-Knoten
- Kommentar-Node „Funktionsbeschreibung“

### 2.1 Import in Node-RED

1. JSON-Export des Flows aus diesem Repository kopieren.
2. In Node-RED: **Menü → Import → Clipboard** → JSON einfügen.
3. Ziel: **Neuer Flow** auswählen, importieren.
4. Danach:
   - ioBroker-Topics ggf. an deine Instanz anpassen (Adapter-ID, Index).
   - Deploy.

## 3. Ein-/Ausgänge der Hauptlogik

- **Trigger**
  - `inject`-Node „Alle 3 Sekunden“
  - `msg.payload` wird ignoriert, dient nur als Takt

- **Output 0 – P_set_aux (W)**
  - `msg.payload` = Leistung für Akku 2
    - `> 0` → Laden
    - `< 0` → Entladen
    - `= 0` → neutral
  - send-by-change mit Deadband `P_SET_DEADBAND_W`
    (kleine Änderungen werden unterdrückt, 0-/Vorzeichenwechsel wird immer gesendet)

- **Output 1 – Debug-Objekt**
  - enthält alle relevanten Zustände (State, SoC, Leistungen, Timer, Limits, Failsafe)
  - wird nur bei Änderungen gesendet (SBC)

## 4. Benötigte Variablen

### 4.1 Globale Variablen (`global.get`)

| Key                                   | Bedeutung                                          |
|---------------------------------------|----------------------------------------------------|
| `200.40097_W`                         | P am Netzverknüpfungspunkt (+Bezug, −Einspeisung)  |
| `BYD_SoC`                             | SoC Hauptakku (%)                                  |
| `BYD7.7_SoC`                          | SoC Akku 2 (%)                                     |
| `GEN24-8.0_Akku_Laden_W`             | Ladeleistung Hauptakku (W, positiv)                |
| `GEN24-8.0_Akku_Entladen_W`          | Entladeleistung Hauptakku (W, positiv)             |
| `GEN24-3.0_Akku_Laden_W`             | Ladeleistung Akku 2 (W, positiv)                   |
| `GEN24-3.0_Akku_Entladen_W`          | Entladeleistung Akku 2 (W, positiv)                |
| `Hausverbrauch_W` (optional)         | berechneter Hausverbrauch                          |
| `OstWest_Freigabe_Akku_Autom_Laden`  | Auto-Laden freigegeben (bool)                      |
| `OstWest_Freigabe_Akku_Autom_Entladen` | Auto-Entladen freigegeben (bool)                 |

### 4.2 Flow-Variablen (`flow.get`)

| Key                              | Bedeutung                                           |
|----------------------------------|-----------------------------------------------------|
| `OstWest_Akku_min_SoC`           | Minimaler SoC für Entladen Akku 2 (%)              |
| `OstWest_Akku_Limit_Charge_Full` | Limit bei fast voll aktiv (Zellspannung) (bool)    |
| `byd_1-olli_mVoltMax`            | maximal gemessene Zellspannung Akku 2 (V, 3 Stellen) |

## 5. State-Machine

### 5.1 Zustände

- `IDLE`  
  - kein aktives Laden/Entladen, P langsam Richtung 0

- `CHG_SURPLUS`  
  - PV-Überschuss in Akku 2 laden

- `DIS_BASE`  
  - Akku 2 entlädt
  - Modi:
    - `GRID` → Netzbezug/Grundlast reduzieren
    - `SUPPORT` → Hauptakku beim Entladen entlasten

- `FREEZE`  
  - Failsafe bei ungültigen Werten (SoC/Leistungen ungültig)  
  - P → 0, bis Werte wieder plausibel

### 5.2 Typische Übergänge (vereinfacht)

- `IDLE → CHG_SURPLUS`
  - Auto-Laden frei, SoC Hauptakku hoch genug,
  - Hauptakku entlädt nicht stark,
  - relevanter Export über `CHG_START_DELAY_S`.

- `IDLE → DIS_BASE (GRID)`
  - Auto-Entladen frei,
  - SoC Akku 2 > Min-SoC,
  - relevanter Import über `DIS_START_DELAY_S`.

- `IDLE → DIS_BASE (SUPPORT)`
  - Auto-Entladen frei,
  - SoC Akku 2 > Min-SoC,
  - Hauptakku entlädt über `MAIN_DIS_SUPPORT_ENTRY_W`.

- Zurück nach `IDLE`
  - Freigaben weg, SoC zu niedrig oder kein Import/Support-Bedarf mehr.

