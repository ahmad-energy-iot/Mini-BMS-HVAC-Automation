# Mini BMS / HVAC Automation – Technical Documentation

Raspberry Pi · Node-RED · Modbus TCP · FlowFuse Dashboard

---

# 🇩🇪 Deutsch

## 1. Projektübersicht

Dieses Projekt ist eine praktische Simulation eines kleinen **Building Management Systems (BMS)** für eine HVAC-Anwendung.

Das System wurde mit folgenden Technologien aufgebaut:

- Raspberry Pi 4
- Raspberry Pi OS
- Node-RED
- Modbus TCP
- FlowFuse Dashboard
- JavaScript Function Nodes
- Linux / systemd
- SSH
- Git / GitHub

Das Ziel des Projekts ist nicht nur, eine funktionierende Automation zu bauen, sondern typische Konzepte aus der Gebäudeautomation praktisch zu verstehen.

Dazu gehören unter anderem:

- Temperatur-Istwert
- Sollwert / Setpoint
- Modbus-Kommunikation
- Register und Coils
- Hysterese
- AUTO / MANUAL Betrieb
- Fan Command
- Fan Feedback
- Alarmüberwachung
- Trend
- Dashboard
- Stateful Control
- Input Validation
- Troubleshooting
- Root Cause Analysis

---

# 2. Grundidee des Systems

Das Projekt simuliert eine einfache HVAC-Regelung.

Ein Temperaturwert wird in einem simulierten Modbus Controller gespeichert.

Node-RED liest diesen Wert und verarbeitet ihn.

Anschließend wird die Temperatur mit einem Sollwert verglichen.

Wenn Kühlung benötigt wird, erzeugt die Steuerungslogik einen Fan Command.

Zusätzlich wird ein Fan Feedback simuliert.

Command und Feedback werden verglichen.

Wenn der Controller den Fan einschalten möchte, aber kein entsprechendes Feedback bekommt, wird ein Alarm erzeugt.

---

# 3. Systemarchitektur

Der Hauptdatenfluss ist:

```text
Simulierter Temperatursensor
        ↓
Modbus Server / Controller Simulation
        ↓
Modbus Read
        ↓
Temperatur-Skalierung
        ↓
Temperatur-Istwert
        ↓
Hysterese-Regelung
        ↓
AUTO / MANUAL Logik
        ↓
Fan Command
        ↓
Modbus Coil
        ↓
Fan Feedback
        ↓
Alarm Logic
        ↓
Dashboard / Trend / Statusanzeige
```

Der Setpoint läuft parallel:

```text
Setpoint
    ↓
Holding Register
    ↓
Modbus Read
    ↓
Skalierung
    ↓
Flow Context
    ↓
Hysterese-Regelung
```

---

# 4. Raspberry Pi und Node-RED

Das Projekt läuft auf einem Raspberry Pi 4.

Node-RED läuft als systemd-Service.

Der Zugriff kann über SSH erfolgen.

Beispiel:

```bash
ssh <user>@<raspberry-pi-ip>
```

Node-RED Editor:

```text
http://<raspberry-pi-ip>:1880
```

Dashboard:

```text
http://<raspberry-pi-ip>:1880/dashboard/page1
```

---

# 5. Modbus TCP Simulation

Für dieses Trainingsprojekt wird kein echter PLC- oder DDC-Controller verwendet.

Stattdessen wird ein virtueller Modbus Server innerhalb von Node-RED verwendet.

Der simulierte Controller stellt verschiedene Modbus-Datentypen bereit:

- Coils
- Discrete Inputs
- Holding Registers
- Input Registers

Der Modbus Client und der Modbus Server laufen auf demselben Raspberry Pi.

Daher kann für die interne Verbindung verwendet werden:

```text
Host: 127.0.0.1
Port: 10502
Unit ID: 1
```

---

# 6. Modbus Mapping

| Signal | Modbus-Typ | Function Code | Adresse | Beschreibung |
|---|---|---:|---:|---|
| Temperature Istwert | Input Register | FC4 | 0 | Temperatur-Rohwert |
| Temperature Setpoint | Holding Register | FC3 / FC6 | 0 | Temperatur-Sollwert |
| Fan Command | Coil | FC1 / FC5 | 0 | EIN/AUS-Befehl |
| Fan Feedback | Discrete Input | FC2 | 0 | Rückmeldung des Gerätes |

---

# 7. Modbus Function Codes

## FC1 – Read Coils

Liest binäre Coil-Zustände.

Im Projekt wird FC1 verwendet, um den geschriebenen Fan Command wieder auszulesen.

---

## FC2 – Read Discrete Inputs

Liest binäre Eingangssignale.

Im Projekt wird FC2 verwendet, um das Fan Feedback zu lesen.

---

## FC3 – Read Holding Registers

Liest Holding Registers.

Im Projekt wird FC3 für den Temperatur-Sollwert verwendet.

---

## FC4 – Read Input Registers

Liest Input Registers.

Im Projekt wird FC4 für den Temperatur-Istwert verwendet.

---

## FC5 – Write Single Coil

Schreibt einen einzelnen binären Coil-Wert.

Im Projekt wird FC5 verwendet, um den Fan Command zu schreiben.

---

## FC6 – Write Single Holding Register

Schreibt einen einzelnen Holding Register Wert.

Im Projekt wird FC6 verwendet, um den Temperatur-Sollwert zu schreiben.

---

# 8. Temperatur-Istwert

Der simulierte Temperaturwert wird als Integer gespeichert.

Beispiel:

```text
246
```

entspricht:

```text
24.6 °C
```

Node-RED liest den Wert mit Modbus FC4.

Die Modbus Read Node liefert beispielsweise:

```text
[246]
```

Die Skalierungsfunktion wandelt den Rohwert anschließend in eine nutzbare Temperatur um.

Beispiel:

```javascript
let rawValue;

if (Array.isArray(msg.payload)) {
    rawValue = msg.payload[0];
} else {
    rawValue = msg.payload;
}

msg.payload = Number(rawValue) / 10;
msg.topic = "Temperature Istwert";

return msg;
```

Ergebnis:

```text
246 → 24.6 °C
```

---

# 9. Temperatur-Sollwert

Der Temperatur-Sollwert wird in einem Holding Register gespeichert.

Beispiel:

```text
23 °C
```

wird beim Schreiben umgerechnet zu:

```text
230
```

Das Holding Register enthält also:

```text
230
```

Beim Lesen wird wieder skaliert:

```text
230 → 23.0 °C
```

Der Wert wird zusätzlich im Node-RED Flow Context gespeichert:

```javascript
flow.set("setpoint", msg.payload);
```

Dadurch können andere Function Nodes auf den aktuellen Sollwert zugreifen.

---

# 10. Setpoint Input Validation

Während der Entwicklung trat zeitweise folgende Ausgabe auf:

```text
NaN
```

`NaN` bedeutet:

```text
Not a Number
```

Die Ursache war, dass nicht jede eingehende Nachricht einen gültigen numerischen Wert enthielt.

Deshalb wurde eine Validierung eingebaut:

```javascript
if (!Array.isArray(msg.payload) || msg.payload.length === 0) {
    return null;
}

let rawValue = Number(msg.payload[0]);

if (Number.isNaN(rawValue)) {
    return null;
}

msg.payload = rawValue * 0.1;

flow.set("setpoint", msg.payload);

msg.topic = "Setpoint";

return msg;
```

`return null` bedeutet:

Die Nachricht wird verworfen und nicht weiter durch den Flow geschickt.

---

# 11. Hysterese

Das System verwendet eine Hysterese, um schnelles Ein- und Ausschalten zu verhindern.

Beispiel:

```text
Setpoint = 23 °C
```

Dann gilt:

```text
Cooling ON  = 24 °C
Cooling OFF = 22 °C
```

Wenn:

```text
Temperature >= 24 °C
```

dann:

```text
Cooling Demand = ON
```

Wenn:

```text
Temperature <= 22 °C
```

dann:

```text
Cooling Demand = OFF
```

Zwischen 22 °C und 24 °C bleibt der vorherige Zustand erhalten.

---

# 12. Stateful Control

Die Hysterese muss den vorherigen Zustand kennen.

Dafür wird der Zustand gespeichert.

Beispiel:

```javascript
flow.set("coolingState", coolingState);
```

und später wieder gelesen:

```javascript
flow.get("coolingState");
```

Das ist ein Beispiel für:

**Stateful Control**

Das bedeutet:

Die Logik entscheidet nicht nur anhand der aktuellen Eingabe, sondern berücksichtigt auch einen gespeicherten vorherigen Zustand.

---

# 13. AUTO / MANUAL Betrieb

Das Projekt unterstützt zwei Betriebsarten.

## AUTO

Im AUTO-Modus entscheidet das System automatisch.

Der Fan Command folgt dem Cooling Demand.

```text
Temperature
    ↓
Hysteresis
    ↓
Cooling Demand
    ↓
Fan Command
```

---

## MANUAL

Im MANUAL-Modus kann der Bediener den Ventilator unabhängig von der Temperatur schalten.

Der manuelle Zustand wird gespeichert.

Beispiel:

```javascript
flow.set("manualFanCommand", msg.payload);
```

---

# 14. Command und Feedback

In der Gebäudeautomation ist die Trennung zwischen Command und Feedback sehr wichtig.

## Command

Command bedeutet:

```text
Was soll das Gerät tun?
```

Beispiel:

```text
Fan Command = ON
```

Der Controller verlangt, dass der Ventilator läuft.

---

## Feedback

Feedback bedeutet:

```text
Was tut das Gerät tatsächlich?
```

Beispiel:

```text
Fan Feedback = ON
```

Das Gerät meldet zurück, dass es läuft.

---

# 15. Reale Feedback-Signale

In einem realen Gebäudeautomationssystem könnte ein Feedback beispielsweise kommen von:

- Hilfskontakt eines Schützes
- VFD Run Status
- Druckschalter
- Strömungsschalter
- Pumpenstatus
- Motorstatus
- Gerätefreigabe
- potentialfreiem Kontakt

Im Projekt wird dieses Feedback simuliert.

---

# 16. Alarm Logic

Das System vergleicht Command und Feedback.

Normaler Zustand:

```text
Fan Command = ON
Fan Feedback = ON
```

Ergebnis:

```text
No alarm
```

Fehlerzustand:

```text
Fan Command = ON
Fan Feedback = OFF
```

Ergebnis:

```text
ALARM - Fan fault
```

Dadurch wird ein typisches Prinzip aus der Gebäudeautomation simuliert:

**Command / Feedback Monitoring**

---

# 17. Dashboard / Visualisierung

Das Projekt verwendet FlowFuse Dashboard.

Das Dashboard zeigt:

- Fan Command Status
- Fan Feedback Status
- Alarm Status
- Temperature Istwert
- Temperature Setpoint
- Temperature Trend
- Setpoint Trend

---

# 18. Fan Command Status

Für die Anzeige wird der Boolean-Wert in Text umgewandelt:

```javascript
msg.payload = msg.fanCommand ? "ON" : "OFF";
return msg;
```

Dadurch sieht der Bediener:

```text
ON
```

oder:

```text
OFF
```

---

# 19. Fan Feedback Status

Auch das Feedback wird für das Dashboard umgewandelt:

```javascript
msg.payload = msg.fanFeedback ? "ON" : "OFF";
return msg;
```

---

# 20. Alarm Status

Die Alarmanzeige wird ebenfalls in Text umgewandelt:

```javascript
if (msg.alarm === true) {
    msg.payload = "ALARM - Fan fault";
} else {
    msg.payload = "No alarm";
}

return msg;
```

---

# 21. Trend

Ein Trend zeigt den zeitlichen Verlauf einer Messgröße.

Im Projekt werden dargestellt:

- Temperature Istwert
- Temperature Setpoint

Dadurch kann man erkennen:

- wie sich die Temperatur verändert
- wann ein Sollwert geändert wurde
- ob das System stabil reagiert
- ob ungewöhnliche Zustände entstehen

---

# 22. Message Filtering für den Chart

Während der Entwicklung entstand eine wichtige Dashboard-Störung.

Der Chart erhielt ursprünglich vollständige Modbus-Nachrichten.

Diese Nachrichten enthielten unter anderem:

```text
responseBuffer.buffer
```

Dadurch wurden große Mengen an Binärdaten in der Trendhistorie gespeichert.

Die Lösung war eine eigene Filterfunktion:

```javascript
return {
    payload: msg.payload,
    topic: msg.topic
};
```

Dadurch erhält der Chart nur noch:

```text
payload
topic
```

und keine unnötigen Modbus-Metadaten mehr.

---

# 23. Troubleshooting Methodik

Für die Fehlersuche wurde folgende Struktur verwendet:

```text
Symptom
    ↓
Investigation
    ↓
Root Cause
    ↓
Fix
    ↓
Verification
```

Diese Struktur ist auch in realen technischen Projekten sinnvoll.

---

# 24. Troubleshooting Fall 1 – Dashboard Reconnect Loop

## Symptom

Das Dashboard wechselte regelmäßig zwischen:

```text
Connected
Disconnected
```

---

## Investigation

Es wurden mehrere mögliche Ursachen untersucht:

- Browser
- Netzwerk
- WLAN
- Raspberry Pi
- Node-RED Logs
- Dashboard-Version
- Socket.IO
- lokaler Zugriff direkt auf dem Raspberry Pi

Dadurch konnte ausgeschlossen werden, dass ausschließlich der Laptop oder das WLAN die Ursache war.

---

## Root Cause

Der Trend speicherte vollständige Modbus-Nachrichten.

Diese enthielten Binärdaten.

Beim Laden der Trendhistorie versuchte Socket.IO eine sehr große Anzahl Binäranhänge zu übertragen.

Die Fehlermeldung war:

```text
parse error: too many attachments
```

---

## Fix

Vor dem Chart wurde eine Message-Filter-Function eingefügt:

```javascript
return {
    payload: msg.payload,
    topic: msg.topic
};
```

---

## Verification

Nach dem Fix wurden:

- mehrere hundert Trendpunkte gesammelt
- Trendhistorien wiederholt geladen
- die Verbindung über mehrere Minuten beobachtet

Danach trat der Fehler nicht mehr auf.

---

# 25. Troubleshooting Fall 2 – NaN Setpoint

## Symptom

Der Setpoint erschien teilweise als:

```text
NaN
```

---

## Root Cause

Nicht jede eingehende Modbus-Nachricht enthielt einen gültigen numerischen Wert.

---

## Fix

Es wurde Input Validation eingebaut.

Ungültige Nachrichten werden verworfen:

```javascript
return null;
```

---

## Verification

Danach wurde der Setpoint stabil angezeigt.

---

# 26. Troubleshooting Fall 3 – Setpoint Reset nach Deploy

## Symptom

Nach einem Deploy oder Neustart wurde der simulierte Setpoint teilweise auf:

```text
0
```

zurückgesetzt.

Dadurch entstanden falsche Hysterese-Grenzen.

Beispiel:

```text
Setpoint = 0
ON Threshold = 1
OFF Threshold = -1
```

Dadurch konnte eine normale Raumtemperatur fälschlicherweise Cooling Demand erzeugen.

---

## Root Cause

Der simulierte Modbus-Speicher wurde nach Deploy oder Restart auf einen Ausgangswert zurückgesetzt.

---

## Fix

Der gewünschte Setpoint wurde erneut in das Holding Register geschrieben.

Beispiel:

```text
23 °C → 230
```

---

## Verification

Danach waren wieder die korrekten Schwellen aktiv:

```text
Setpoint = 23 °C
Cooling ON = 24 °C
Cooling OFF = 22 °C
```

---

# 27. Troubleshooting Fall 4 – Mehrere Temperaturquellen

## Symptom

Während eines Tests war zunächst nicht eindeutig, ob die Temperatur aus:

- einem alten Inject Node

oder:

- dem Modbus Read

kam.

---

## Root Cause

Mehrere Datenquellen konnten dieselbe Verarbeitungskette speisen.

---

## Fix

Der alte Inject Node wurde während des Tests getrennt.

---

## Verification

Danach war der Datenpfad eindeutig:

```text
Modbus FC4
    ↓
[246]
    ↓
Scaling
    ↓
24.6 °C
    ↓
Hysteresis
    ↓
Fan Logic
```

---

# 28. Wichtige gelernte Begriffe

## Istwert

Der aktuell gemessene Wert.

Beispiel:

```text
24.6 °C
```

---

## Sollwert / Setpoint

Der gewünschte Zielwert.

Beispiel:

```text
23 °C
```

---

## Threshold

Eine Schaltschwelle.

---

## Trigger

Ein Ereignis, das eine Aktion oder Logik auslöst.

---

## Boolean

Ein Wert mit zwei Zuständen:

```text
true
false
```

---

## State

Ein gespeicherter Zustand.

---

## Register

Ein numerischer Speicherplatz im Modbus-System.

---

## Coil

Ein binärer ON/OFF Speicherplatz.

---

## Discrete Input

Ein binärer Eingang, der gelesen wird.

---

## Alarm

Eine Meldung über einen ungewöhnlichen oder fehlerhaften Zustand.

---

## Trend

Zeitlicher Verlauf eines Signals.

---

## Visualization

Grafische Darstellung von Prozessdaten.

---

## Troubleshooting

Systematische Fehlersuche.

---

## Root Cause

Die eigentliche Ursache eines Problems.

---

## Fault Detection

Erkennung eines Fehlerzustands.

---

## Fault Simulation

Gezieltes Simulieren eines Fehlers.

---

## Input Validation

Überprüfung, ob eingehende Daten gültig sind.

---

## Message Filtering

Entfernen unnötiger Daten aus einer Nachricht.

---

# 29. Reale Anwendung

Der simulierte Fan ist nur ein Beispiel.

Das gleiche Prinzip kann auf reale Gebäudeautomation übertragen werden.

Zum Beispiel:

- Lüftungsanlage
- Klimaanlage
- Heizungsanlage
- Pumpe
- Ventilator
- Ventil
- Fan Coil Unit
- Air Handling Unit
- Frequenzumrichter
- Kaltwasseranlage
- Warmwasseranlage

Der Grundaufbau bleibt ähnlich:

```text
Sensor
    ↓
Controller
    ↓
Control Logic
    ↓
Command
    ↓
Aktor
    ↓
Feedback
    ↓
Alarm / BMS
```

---

# 30. Projektstatus

Aktuell umgesetzt:

- Modbus TCP Simulation
- Temperature Measurement
- Setpoint Handling
- Scaling
- Hysteresis
- Stateful Control
- AUTO / MANUAL
- Fan Command
- Fan Feedback
- Alarm Logic
- Dashboard
- Trend
- Input Validation
- Troubleshooting
- Message Filtering
- Documentation

---

# 31. Mögliche Erweiterungen

Mögliche zukünftige Erweiterungen:

- BACnet/IP
- Modbus RTU
- RS485
- mehrere HVAC-Zonen
- Pumpensteuerung
- Ventilsteuerung
- Heizungsregelung
- Zeitprogramme
- Alarmhistorie
- Datenbankintegration
- Energy Monitoring
- echte ESP32 Sensoren
- echte Aktoren
- SCADA-ähnliche Visualisierung
- BACnet Device Simulation
- Digital Twin
- 3D Gebäudemodell

---

# 32. Fazit

Dieses Projekt verbindet mehrere wichtige Grundlagen der Gebäudeautomation.

Dazu gehören:

- Kommunikation
- Steuerungslogik
- Visualisierung
- Alarmierung
- Zustandsverwaltung
- Fehleranalyse
- Modbus
- HVAC-Grundlagen

Besonders wichtig war, nicht nur ein funktionierendes System aufzubauen, sondern auch zu verstehen:

- wie Daten durch das System laufen
- warum ein Zustand entsteht
- wie Command und Feedback zusammenhängen
- wie Fehler erkannt werden
- wie Probleme systematisch untersucht werden
- wie Steuerungs- und Visualisierungsdaten getrennt werden

---

# 🇬🇧 English

## 1. Project Overview

This project is a practical simulation of a small **Building Management System (BMS)** for an HVAC application.

The system was built using:

- Raspberry Pi 4
- Raspberry Pi OS
- Node-RED
- Modbus TCP
- FlowFuse Dashboard
- JavaScript Function Nodes
- Linux / systemd
- SSH
- Git / GitHub

The purpose of the project is not only to create a working automation system, but also to understand important building automation concepts in practice.

These include:

- Temperature actual value
- Temperature setpoint
- Modbus communication
- Registers and coils
- Hysteresis
- AUTO / MANUAL operation
- Fan Command
- Fan Feedback
- Alarm monitoring
- Trend visualization
- Dashboard
- Stateful control
- Input validation
- Troubleshooting
- Root Cause Analysis

---

# 2. System Concept

The project simulates a simple HVAC control process.

A temperature value is stored inside a simulated Modbus controller.

Node-RED reads and processes this value.

The temperature is then compared with a setpoint.

If cooling is required, the control logic generates a Fan Command.

A Fan Feedback signal is also simulated.

Command and feedback are compared.

If the controller requests the fan to run but no corresponding feedback is received, the system generates an alarm.

---

# 3. System Architecture

Main data flow:

```text
Simulated Temperature Sensor
        ↓
Modbus Server / Controller Simulation
        ↓
Modbus Read
        ↓
Temperature Scaling
        ↓
Temperature Actual Value
        ↓
Hysteresis Control
        ↓
AUTO / MANUAL Logic
        ↓
Fan Command
        ↓
Modbus Coil
        ↓
Fan Feedback
        ↓
Alarm Logic
        ↓
Dashboard / Trend / Status Visualization
```

The setpoint is handled in parallel:

```text
Setpoint
    ↓
Holding Register
    ↓
Modbus Read
    ↓
Scaling
    ↓
Flow Context
    ↓
Hysteresis Control
```

---

# 4. Raspberry Pi and Node-RED

The project runs on a Raspberry Pi 4.

Node-RED runs as a systemd service.

SSH access can be used.

Example:

```bash
ssh <user>@<raspberry-pi-ip>
```

Node-RED Editor:

```text
http://<raspberry-pi-ip>:1880
```

Dashboard:

```text
http://<raspberry-pi-ip>:1880/dashboard/page1
```

---

# 5. Modbus TCP Simulation

No real PLC or DDC controller is required for this training project.

Instead, a virtual Modbus Server inside Node-RED is used.

The simulated controller provides:

- Coils
- Discrete Inputs
- Holding Registers
- Input Registers

The Modbus client and server run on the same Raspberry Pi.

Example configuration:

```text
Host: 127.0.0.1
Port: 10502
Unit ID: 1
```

---

# 6. Modbus Mapping

| Signal | Modbus Type | Function Code | Address | Description |
|---|---|---:|---:|---|
| Temperature Actual Value | Input Register | FC4 | 0 | Raw temperature value |
| Temperature Setpoint | Holding Register | FC3 / FC6 | 0 | Temperature target value |
| Fan Command | Coil | FC1 / FC5 | 0 | Fan ON/OFF command |
| Fan Feedback | Discrete Input | FC2 | 0 | Fan running feedback |

---

# 7. Modbus Function Codes

## FC1 – Read Coils

Reads binary Coil states.

Used to read the Fan Command state.

---

## FC2 – Read Discrete Inputs

Reads binary input signals.

Used to read Fan Feedback.

---

## FC3 – Read Holding Registers

Reads Holding Registers.

Used to read the temperature setpoint.

---

## FC4 – Read Input Registers

Reads Input Registers.

Used to read the temperature actual value.

---

## FC5 – Write Single Coil

Writes one binary Coil value.

Used to write the Fan Command.

---

## FC6 – Write Single Holding Register

Writes one Holding Register.

Used to write the temperature setpoint.

---

# 8. Temperature Actual Value

The simulated temperature is stored as an integer.

Example:

```text
246
```

represents:

```text
24.6 °C
```

The Modbus Read node may return:

```text
[246]
```

Node-RED then scales the raw value.

Example:

```javascript
let rawValue;

if (Array.isArray(msg.payload)) {
    rawValue = msg.payload[0];
} else {
    rawValue = msg.payload;
}

msg.payload = Number(rawValue) / 10;
msg.topic = "Temperature Istwert";

return msg;
```

Result:

```text
246 → 24.6 °C
```

---

# 9. Temperature Setpoint

The temperature setpoint is stored inside a Holding Register.

Example:

```text
23 °C
```

is written as:

```text
230
```

When the register is read, the value is scaled back.

The current setpoint is also stored inside the Node-RED Flow Context:

```javascript
flow.set("setpoint", msg.payload);
```

This allows other parts of the control logic to access the current setpoint.

---

# 10. Setpoint Input Validation

During development, the setpoint occasionally appeared as:

```text
NaN
```

`NaN` means:

```text
Not a Number
```

Input validation was added:

```javascript
if (!Array.isArray(msg.payload) || msg.payload.length === 0) {
    return null;
}

let rawValue = Number(msg.payload[0]);

if (Number.isNaN(rawValue)) {
    return null;
}

msg.payload = rawValue * 0.1;

flow.set("setpoint", msg.payload);

msg.topic = "Setpoint";

return msg;
```

`return null` means the invalid message is discarded.

---

# 11. Hysteresis Control

The system uses hysteresis to prevent rapid ON/OFF switching.

Example:

```text
Setpoint = 23 °C
```

Thresholds:

```text
Cooling ON  = 24 °C
Cooling OFF = 22 °C
```

If:

```text
Temperature >= 24 °C
```

then:

```text
Cooling Demand = ON
```

If:

```text
Temperature <= 22 °C
```

then:

```text
Cooling Demand = OFF
```

Between both thresholds, the previous cooling state is maintained.

---

# 12. Stateful Control

The hysteresis logic needs to know its previous state.

The state is stored using:

```javascript
flow.set("coolingState", coolingState);
```

and read using:

```javascript
flow.get("coolingState");
```

This is an example of:

**Stateful Control**

---

# 13. AUTO / MANUAL Operation

The project supports two operating modes.

## AUTO

In AUTO mode, the system makes the decision automatically.

```text
Temperature
    ↓
Hysteresis
    ↓
Cooling Demand
    ↓
Fan Command
```

---

## MANUAL

In MANUAL mode, the operator can manually switch the fan independently of temperature.

Example:

```javascript
flow.set("manualFanCommand", msg.payload);
```

---

# 14. Command and Feedback

Command and Feedback are two different concepts.

## Command

Command means:

```text
What should the equipment do?
```

Example:

```text
Fan Command = ON
```

---

## Feedback

Feedback means:

```text
What is the equipment actually doing?
```

Example:

```text
Fan Feedback = ON
```

---

# 15. Real Feedback Signals

In real systems, feedback can come from:

- contactor auxiliary contacts
- VFD run status
- pressure switches
- flow switches
- pump status
- motor status
- equipment status contacts

In this project, the feedback is simulated.

---

# 16. Alarm Logic

Normal operation:

```text
Fan Command = ON
Fan Feedback = ON
```

Result:

```text
No alarm
```

Fault condition:

```text
Fan Command = ON
Fan Feedback = OFF
```

Result:

```text
ALARM - Fan fault
```

This represents typical Command / Feedback Monitoring.

---

# 17. Dashboard / Visualization

The project uses FlowFuse Dashboard.

The dashboard displays:

- Fan Command Status
- Fan Feedback Status
- Alarm Status
- Temperature Actual Value
- Temperature Setpoint
- Temperature Trend
- Setpoint Trend

---

# 18. Fan Command Status

The Boolean value is converted to readable text:

```javascript
msg.payload = msg.fanCommand ? "ON" : "OFF";
return msg;
```

---

# 19. Fan Feedback Status

The Fan Feedback is converted in the same way:

```javascript
msg.payload = msg.fanFeedback ? "ON" : "OFF";
return msg;
```

---

# 20. Alarm Status

The alarm is converted to dashboard text:

```javascript
if (msg.alarm === true) {
    msg.payload = "ALARM - Fan fault";
} else {
    msg.payload = "No alarm";
}

return msg;
```

---

# 21. Trend Visualization

A trend displays how a value changes over time.

The project displays:

- Temperature Actual Value
- Temperature Setpoint

This makes it possible to observe:

- temperature changes
- setpoint changes
- control response
- abnormal behavior

---

# 22. Message Filtering for the Chart

An important issue was discovered during development.

The chart originally received complete Modbus messages.

These messages contained binary data such as:

```text
responseBuffer.buffer
```

This caused Socket.IO communication problems.

A dedicated filtering function was therefore added:

```javascript
return {
    payload: msg.payload,
    topic: msg.topic
};
```

The chart now receives only:

```text
payload
topic
```

The visualization path is therefore separated from unnecessary Modbus metadata.

---

# 23. Troubleshooting Method

The following troubleshooting structure was used:

```text
Symptom
    ↓
Investigation
    ↓
Root Cause
    ↓
Fix
    ↓
Verification
```

---

# 24. Troubleshooting Case 1 – Dashboard Reconnect Loop

## Symptom

The dashboard repeatedly changed between:

```text
Connected
Disconnected
```

---

## Investigation

The following areas were checked:

- browser
- different browsers
- network connection
- Raspberry Pi
- Node-RED logs
- dashboard version
- Socket.IO
- direct local access on the Raspberry Pi

---

## Root Cause

The trend stored complete Modbus messages.

These contained binary data.

When the history was loaded, Socket.IO attempted to transfer a large number of binary attachments.

The error was:

```text
parse error: too many attachments
```

---

## Fix

A filtering function was added:

```javascript
return {
    payload: msg.payload,
    topic: msg.topic
};
```

---

## Verification

After the fix:

- hundreds of trend points were collected
- trend history was repeatedly loaded
- the connection remained stable

---

# 25. Troubleshooting Case 2 – NaN Setpoint

## Symptom

The setpoint occasionally appeared as:

```text
NaN
```

---

## Root Cause

Some incoming Modbus messages did not contain a valid numeric value.

---

## Fix

Input validation was added.

Invalid messages are discarded using:

```javascript
return null;
```

---

## Verification

The setpoint remained stable after the fix.

---

# 26. Troubleshooting Case 3 – Setpoint Reset After Deploy

## Symptom

After a deploy or restart, the simulated setpoint sometimes returned to:

```text
0
```

This caused incorrect hysteresis thresholds.

---

## Root Cause

The simulated Modbus memory was reset.

---

## Fix

The desired setpoint was written again.

Example:

```text
23 °C → 230
```

---

## Verification

Correct thresholds were restored:

```text
Setpoint = 23 °C
Cooling ON = 24 °C
Cooling OFF = 22 °C
```

---

# 27. Troubleshooting Case 4 – Multiple Temperature Sources

## Symptom

It was initially unclear whether the temperature came from:

- the old Inject node

or:

- Modbus Read

---

## Root Cause

Multiple sources were connected to the same processing path.

---

## Fix

The old Inject node was temporarily disconnected during testing.

---

## Verification

The active data path was confirmed as:

```text
Modbus FC4
    ↓
[246]
    ↓
Scaling
    ↓
24.6 °C
    ↓
Hysteresis
    ↓
Fan Logic
```

---

# 28. Key Concepts Learned

The project provided practical experience with:

- BMS
- HVAC Automation
- Sensor → Controller → Actuator
- Actual Value
- Setpoint
- Hysteresis
- AUTO / MANUAL
- Command
- Feedback
- Alarm
- Trend
- Modbus TCP
- Input Register
- Holding Register
- Coil
- Discrete Input
- Function Codes
- Scaling
- Boolean
- State
- Flow Context
- Stateful Control
- Threshold
- Trigger
- Input Validation
- Fault Detection
- Fault Simulation
- Troubleshooting
- Root Cause Analysis
- Message Filtering
- Visualization

---

# 29. Real-World Relevance

The simulated fan is only one example.

The same concepts can be applied to real equipment such as:

- ventilation systems
- air-conditioning systems
- heating systems
- pumps
- fans
- valves
- fan coil units
- air handling units
- VFDs
- chilled-water systems
- hot-water systems

The basic principle remains:

```text
Sensor
    ↓
Controller
    ↓
Control Logic
    ↓
Command
    ↓
Actuator
    ↓
Feedback
    ↓
Alarm / BMS
```

---

# 30. Project Status

The current project includes:

- Modbus TCP simulation
- temperature measurement
- setpoint handling
- scaling
- hysteresis control
- stateful control
- AUTO / MANUAL
- Fan Command
- Fan Feedback
- alarm logic
- trend visualization
- dashboard
- input validation
- troubleshooting
- message filtering
- documentation

---

# 31. Possible Future Extensions

Possible future extensions include:

- BACnet/IP
- Modbus RTU
- RS485
- multiple HVAC zones
- pump control
- valve control
- heating control
- schedules
- alarm history
- database integration
- energy monitoring
- real ESP32 sensors
- real actuators
- SCADA-style visualization
- BACnet device simulation
- digital twin
- 3D building model

---

# 32. Conclusion

This project combines important building automation fundamentals:

- communication
- control logic
- visualization
- alarm handling
- state management
- troubleshooting
- Modbus
- HVAC concepts

The main learning objective was not only to create a working system, but also to understand:

- how data moves through the system
- why control states occur
- how Command and Feedback are related
- how faults are detected
- how problems are investigated systematically
- how control data and visualization data should be separated
