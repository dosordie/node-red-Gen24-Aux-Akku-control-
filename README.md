# Node-RED – BYD Aux-Akku 2 Regelung

Dieses Repository enthält die Node-RED-Logik zur Steuerung eines zweiten BYD-Akkus (Aux-Akku) an einem separaten GEN24.

Die Regelung besteht aus zwei Funktionsknoten:

1. **BYD2 Hauptlogik**  
   Arbeitet auf Watt-Basis (`P_set_aux`) und entscheidet, ob und wie stark Akku 2 laden oder entladen soll.

2. **BYD2 Ausgangslogik**  
   Wandelt `P_set_aux` in die beiden BYD-Register **InWRte** und **OutWRte** um, inklusive richtiger Vorzeichen, Reihenfolge und Delay.

---

## 1. Gesamtüberblick

### 1.1 BYD2 Hauptlogik

- Arbeitet mit `P_set_aux` in **Watt**.
- Zustandsautomat mit den States:
  - `IDLE`
  - `CHG_SURPLUS` (Laden aus PV-Überschuss)
  - `DIS_BASE` (Entladen zur Grundlastdeckung / Support)
  - `FREEZE` (Failsafe)
- Berücksichtigt:
  - Netzbezug / Einspeisung (`P_grid`)
  - SoC beider Akkus (`SoC_main`, `SoC_aux`)
  - Lade-/Entladeleistung des Haupt- und Aux-Akkus
  - Zellspannung des Aux-Akkus
  - Auto-Freigaben Laden/Entladen
- Liefert:
  - `P_set_aux` (Watt) als Setpoint für Akku 2
  - Debug-Objekt mit allen Zuständen und Messwerten

### 1.2 BYD2 Ausgangslogik

- Wandelt `P_set_aux` → **InWRte / OutWRte** (in %).
- Kümmert sich um:
  - richtige Vorzeichen-Kombination Laden/Entladen
  - Reihenfolge der Register-Schreibungen
  - Delays für negative Werte
  - „0-Stellen“ der Register in sicherer Reihenfolge
- Send-by-Change (SBC) pro Register.

---

## 2. BYD2 Hauptlogik

Datei: [`functions/byd2-main-logic.js`](functions/byd2-main-logic.js)  
Node-RED: Function-Node mit **2 Ausgängen**.

### 2.1 I/O

**Eingang**

- `msg`: Tick-Event (z. B. Inject alle 3 s)
- `msg.payload` wird nicht verwendet, dient nur als Trigger.

**Ausgänge**

- **Output 0:**  
  `msg.payload = P_set_aux` (Number, in W)

  - `> 0` → Akku 2 **laden**
  - `< 0` → Akku 2 **entladen**
  - `0`   → neutral / nichts tun

  Send-by-Change: es wird nur gesendet, wenn sich der Wert um mind. `P_SET_DEADBAND_W` ändert oder das Vorzeichen wechselt / auf 0 geht.

- **Output 1:**  
  `msg.payload = Debug-Objekt`  
  Enthält alle relevanten Zustände (State, SoC, Leistungen, Timer, Limits etc.).  
  Ebenfalls SBC (nur bei Änderungen).

### 2.2 Benötigte globale Variablen

Folgende Werte werden aus `global.get()` gelesen:

- `200.40097_W`  
  Leistung am Netzverknüpfungspunkt  
  - `+` = Netzbezug  
  - `−` = Einspeisung

- `BYD_SoC`  
  SoC Hauptakku in %

- `BYD7.7_SoC`  
  SoC Aux-Akku 2 in %

- `GEN24-8.0_Akku_Laden_W`  
  aktuelle Ladeleistung Hauptakku (W, positiv)

- `GEN24-8.0_Akku_Entladen_W`  
  aktuelle Entladeleistung Hauptakku (W, positiv)

- `GEN24-3.0_Akku_Laden_W`  
  aktuelle Ladeleistung Aux-Akku 2 (W, positiv)

- `GEN24-3.0_Akku_Entladen_W`  
  aktuelle Entladeleistung Aux-Akku 2 (W, positiv)

- `Hausverbrauch_W` (optional)  
  gerechneter Hausverbrauch, wird bevorzugt für DIS_BASE (GRID-Mode) genutzt.

- `OstWest_Freigabe_Akku_Autom_Laden` (Boolean)  
  Auto-Laden Akku 2 erlaubt

- `OstWest_Freigabe_Akku_Autom_Entladen` (Boolean)  
  Auto-Entladen Akku 2 erlaubt

### 2.3 Benötigte Flow-Variablen

Folgende Werte werden aus `flow.get()` gelesen:

- `OstWest_Akku_min_SoC` (Number, %)  
  Minimaler SoC für Entladung von Akku 2, sonst Fallback `SOC_AUX_MIN_DISCHARGE_DEFAULT` (z. B. 5 %).

- `OstWest_Akku_Limit_Charge_Full` (Boolean)  
  `true`: Ladeleistung wird reduziert, wenn Zellspannung hoch ist.

- `byd_1-olli_mVoltMax` (Number, in V)  
  maximale Zellspannung (z. B. `3.428`).

### 2.4 States der State-Machine

- `STATE_IDLE`  
  Kein spezieller Lade-/Entlademodus aktiv, `P_set_aux` geht auf 0.

- `STATE_CHG_SURPLUS`  
  Laden von PV-Überschuss. Ladeleistung wird dynamisch anhand `P_grid` angepasst  
  (Einspeisung → erhöhen, Netzbezug → reduzieren).

- `STATE_DIS_BASE`

  - `DIS_MODE_GRID`: Netzbezug / Grundlast am Netzverknüpfungspunkt reduzieren
  - `DIS_MODE_SUPPORT`: Hauptakku beim Entladen unterstützen (Leistungsteilung nach Kapazitätsverhältnis)

- `STATE_FREEZE`  
  Failsafe bei ungültigen Messwerten → `P_set_aux = 0`.

### 2.5 Wichtige State-Wechsel (Auszug)

**IDLE → CHG_SURPLUS (Laden starten)**

- Auto-Laden freigegeben (`OstWest_Freigabe_Akku_Autom_Laden = true`)
- `SoC_main ≥ SOC_MAIN_MIN_FOR_AUX_CHARGE`
- Hauptakku entlädt nicht stark (`P_main_dis <= MAIN_DISCHARGE_WEAK_W`)
- relevante Einspeisung: `P_grid` deutlich negativ und länger als `CHG_START_DELAY_S`
- kein starker Entladestrom Hauptakku (`P_main_dis <= MAIN_DISCHARGE_STRONG_W`)

**IDLE → DIS_BASE (GRID)**

- Auto-Entladen freigegeben
- `SoC_aux > SOC_AUX_MIN_DISCHARGE`
- relevanter Netzbezug länger als `DIS_START_DELAY_S`

**IDLE → DIS_BASE (SUPPORT)**

- Auto-Entladen freigegeben
- `SoC_aux > SOC_AUX_MIN_DISCHARGE`
- Hauptakku entlädt mit mindestens `MAIN_DIS_SUPPORT_ENTRY_W`

**CHG_SURPLUS → IDLE**

- Auto-Laden nicht freigegeben **ODER**
- `SoC_main < SOC_MAIN_MIN_FOR_AUX_CHARGE` **ODER**
- länger Import / kein Überschuss:  
  `P_grid >= 0` **und** `tImportStop >= CHG_STOP_IMPORT_DELAY_S` **und** `lastPset === 0`

**DIS_BASE → IDLE**

- Auto-Entladen nicht freigegeben **ODER**
- `SoC_aux <= Min-SoC` **ODER**
- `DIS_MODE_GRID`: `P_grid <= 0`  
- `DIS_MODE_SUPPORT`: `P_main_dis < MAIN_DIS_SUPPORT_EXIT_W`

### 2.6 `P_set_aux` – Berechnung (Kurzfassung)

**Laden (`STATE_CHG_SURPLUS`)**

- Korrektur aus `P_grid`:

  ```js
  if (P_grid < -GRID_TOLERANCE_W)      // Einspeisung
      P_surplus = -P_grid;             // mehr laden
  else if (P_grid > GRID_TOLERANCE_W)  // Netzbezug
      P_surplus = -P_grid;             // weniger laden
  else
      P_surplus = 0;
