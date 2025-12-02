# Node-RED – BYD Aux-Akku 2 Regelung

Dieses Repository enthält die Node-RED-Logik zur Steuerung eines zweiten BYD-Akkus (Aux-Akku) an einem separaten GEN24.

Die Regelung besteht aus zwei Funktionsknoten:

1. **BYD2 Hauptlogik**  
   Arbeitet auf Watt-Basis (`P_set_aux`) und entscheidet, ob und wie stark Akku 2 laden oder entladen soll.

2. **BYD2 Ausgangslogik**  
   Wandelt `P_set_aux` in die beiden BYD-Register **InWRte** und **OutWRte** um, inklusive richtiger Vorzeichen, Reihenfolge und Delay.

---

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

### 3.1 Trigger

- `inject`-Node „Alle 3 Sekunden“ triggert die Hauptfunktion.
- `msg.payload` wird nur als Takt verwendet und in der Funktion ignoriert.

### 3.2 Output 0 – P_set_aux (W)

- `msg.payload` = gewünschte Leistung für Akku 2:

  - `> 0` → Akku 2 **laden**
  - `< 0` → Akku 2 **entladen**
  - `= 0` → neutral

- send-by-change mit Deadband `P_SET_DEADBAND_W` (z. B. 20 W):
  - kleine Änderungen werden unterdrückt
  - bei Wechsel von Laden ↔ Entladen oder auf 0 wird immer gesendet

### 3.3 Output 1 – Debug-Objekt

- `msg.payload` = Objekt mit u. a.:

  - `state`, `stateBase`, `disMode`
  - `P_grid`, `P_house`
  - `SoC_main`, `SoC_aux`, `socAuxMinDischarge`
  - `P_main_chg`, `P_main_dis`, `P_aux_chg`, `P_aux_dis`
  - `auxChargeEnable`, `auxDischargeEnable`
  - `tGridImportHigh`, `tGridExportHigh`, `tImportStop`, `tStateHold`
  - `P_set_aux`, `lastSupportTarget`
  - `chargeLimitActive`, `auxLimitFullEnable`, `auxCellMaxV`, `effectiveMaxChargeW`
  - `failsafeReason`

Das Debug-Objekt geht weiter in **„Visu Status Text“**, wo es in einen kompakten String
für die Gira-Visu umgebaut wird.

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

Diese Keys werden über kleine Helper-Funktionen aus ioBroker-Werten befüllt.

### 4.2 Flow-Variablen (`flow.get`)

| Key                              | Bedeutung                                           |
|----------------------------------|-----------------------------------------------------|
| `OstWest_Akku_min_SoC`           | Minimaler SoC für Entladen Akku 2 (%)              |
| `OstWest_Akku_Limit_Charge_Full` | Limit bei fast voll aktiv (Zellspannung) (bool)    |
| `byd_1-olli_mVoltMax`           | maximal gemessene Zellspannung Akku 2 (V, 3 Stellen) |

## 5. State-Machine

### 5.1 Zustände

- `IDLE`  
  - kein aktives Laden/Entladen  
  - `P_set_aux` wird langsam auf 0 zurückgeführt

- `CHG_SURPLUS`  
  - PV-Überschuss laden  
  - `P_set_aux > 0`

- `DIS_BASE`  
  - Entladen zur Grundlastdeckung / Hauptakku-Support  
  - `P_set_aux < 0`  
  - Untermodi:
    - `GRID`: Netzbezug reduzieren
    - `SUPPORT`: Entladeleistung nach Kapazitätsverhältnis teilen

- `FREEZE`  
  - Failsafe bei ungültigen Werten (z. B. `NaN` bei SoC/Leistung)  
  - `P_set_aux → 0`, bis die Werte wieder plausibel sind

### 5.2 Beispiel-Übergänge

- **IDLE → CHG_SURPLUS**, wenn:
  - Auto-Laden freigegeben (`OstWest_Freigabe_Akku_Autom_Laden`)
  - SoC Hauptakku ≥ `SOC_MAIN_MIN_FOR_AUX_CHARGE`
  - Hauptakku entlädt nicht stark (`P_main_dis` unter Schwellwert)
  - signifikanter Export (`P_grid` deutlich negativ) über `CHG_START_DELAY_S`

- **IDLE → DIS_BASE (GRID)**, wenn:
  - Auto-Entladen freigegeben
  - SoC Akku 2 > Minimal-SoC
  - signifikanter Netzbezug über `DIS_START_DELAY_S`

- **IDLE → DIS_BASE (SUPPORT)**, wenn:
  - Auto-Entladen freigegeben
  - SoC Akku 2 > Minimal-SoC
  - Hauptakku entlädt mit mindestens `MAIN_DIS_SUPPORT_ENTRY_W`

## 6. Berechnung von P_set_aux

### 6.1 CHG_SURPLUS (Laden aus Überschuss)

`P_grid`-basierte Korrektur:

- Einspeisung (`P_grid < -GRID_TOLERANCE_W`)  
  → `P_surplus = -P_grid` (mehr laden)
- Import (`P_grid > GRID_TOLERANCE_W`)  
  → `P_surplus = -P_grid` (Ladeleistung reduzieren)
- nahe 0  
  → `P_surplus = 0`

Ziel:

```js
let P_target = P_aux_chg + P_surplus;  // aktuelle Ladeleistung + Korrektur
if (P_target < 0) P_target = 0;

P_target = Math.min(
  P_target,
  AUX_WR_AC_MAX_W,
  effectiveMaxChargeW   // Zellspannungs-/C-Rate-Limit
);

P_set_aux_new = P_target; // > 0 = Laden
let P_base_need;

if (isFinite(P_house) && P_house > 0) {
  P_base_need = Math.min(P_house, BASELOAD_TARGET_W);
} else {
  P_base_need = Math.min(Math.max(P_grid, 0), BASELOAD_TARGET_W);
}

let target = Math.min(P_base_need, BASELOAD_TARGET_W, AUX_WR_AC_MAX_W);
P_set_aux_new = -Math.max(0, Math.round(target)); // < 0 = Entladen


---

### Block 7 – Rampe & Dynamik

```markdown
## 7. Rampe & Dynamik

Es gibt getrennte Rampen für Laden und Entladen:

- **Laden (CHG)**  
  - max. Schritt: `AUX_P_DELTA_MAX_CHG_W` (z. B. 200 W)  
  - min. Zeit zwischen zwei „größer werden“-Schritten:
    `RAMP_MIN_HOLD_CHG_S` (z. B. 10 s)

- **Entladen (DIS)**  
  - max. Schritt: `AUX_P_DELTA_MAX_DIS_W` (z. B. 80 W)  
  - min. Zeit zwischen zwei „größer werden“-Schritten:
    `RAMP_MIN_HOLD_DIS_S` (z. B. 30 s)

Im SUPPORT-Modus werden Änderungen zusätzlich „gehalten“, damit der WR nicht ständig
zwischen Stufen hin- und herspringt.

Generell:

- Betrag größer → nur alle X Sekunden  
- Betrag kleiner → darf schneller folgen (innerhalb Δ-Max-Grenze)
- sehr kleine Leistungen (`|P_set_aux| < 10 W`) werden auf 0 geklemmt

## 8. BYD-Ausgangslogik (InWRte / OutWRte)

Funktion: **DIE Ausgangs Logik V2**

### 8.1 Grundprinzip

- Input: `P_set_aux` (W)
- Umrechnung in Prozent relativ zu `AUX_BAT_MAX_W`
- 1 Nachkommastelle

Modus:

- `P_set_aux > 0` → **CHG (Laden erzwingen)**  
  - `InWRte = +X`, `OutWRte = -X`

- `P_set_aux < 0` → **DIS (Entladen erzwingen)**  
  - `OutWRte = +X`, `InWRte = -X`

- `P_set_aux = 0` → Spezialfall:
  - zuerst das **negative** Register auf 0
  - dann das andere Register mit Delay (`msg.delay = NEG_DELAY_MS`)

### 8.2 Reihenfolge & Delay

- Negative Werte (oder zweiter Schritt bei 0-Stellung) bekommen ein Delay,
  damit die BYD-Firmware die Sequenz sicher verarbeiten kann.
- Umsetzung über `delay`-Node im Modus **delayv**:
  - Input: `msg.delay` (ms)
  - danach geht es zum jeweiligen `ioBroker out` (Modbus-Adapter).

### 8.3 Outputs

- **Output 0**: `InWRte` → `modbus.7.holdingRegisters.1.40366_InWRte`
- **Output 1**: `OutWRte` → `modbus.7.holdingRegisters.1.40365_OutWRte`

Beide Ausgänge sind send-by-change (SBC), d. h. es werden nur Änderungen gesendet.

## 9. Visualisierung (Gira)

### 9.1 Visu Status Text

Function-Node **„Visu Status Text“** nimmt das Debug-Objekt (2. Ausgang der Hauptlogik)
und erzeugt einen String, z. B.:

```text
Mode=CHG_SURPLUS | Akku=1200W


---

### Block 10 – Tuning

```markdown
## 10. Tuning

Typische Stellschrauben in der Hauptfunktion:

- **Laden zu träge?**
  - `AUX_P_DELTA_MAX_CHG_W` erhöhen (größere Schritte)
  - `RAMP_MIN_HOLD_CHG_S` verkleinern (häufiger nachregeln)

- **Entladen zu nervös?**
  - `AUX_P_DELTA_MAX_DIS_W` verkleinern
  - `RAMP_MIN_HOLD_DIS_S` vergrößern
  - ggf. `BASELOAD_TARGET_W` etwas erhöhen oder `GRID_TOLERANCE_W` anpassen

- **SUPPORT-Modus „zittert“?**
  - `SUPPORT_STEP_W` erhöhen (gröbere Stufen)
  - `SUPPORT_TARGET_HYST_W` erhöhen (größere Hysterese)

- **Akku 2 bei hoher Zellspannung zu aggressiv?**
  - `AUX_MAX_CELL_V` etwas niedriger wählen
  - `AUX_MAX_CHARGE_FULL_W` kleiner wählen
