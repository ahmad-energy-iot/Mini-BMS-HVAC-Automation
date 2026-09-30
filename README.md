# Mini BMS / HVAC Automation

**Raspberry Pi · Node-RED · Modbus TCP · FlowFuse Dashboard**

A practical Building Management System simulation for HVAC automation, including temperature control, Modbus communication, AUTO/MANUAL operation, fan monitoring, alarms, trends and dashboard visualization.

---

# 🇩🇪 Deutsch

## Überblick

Dieses Projekt simuliert ein kleines **Building Management System (BMS)** für eine HVAC-Anwendung.

Es wurde mit **Raspberry Pi 4**, **Node-RED**, **Modbus TCP** und **FlowFuse Dashboard** umgesetzt und zeigt typische Funktionen aus der Gebäudeautomation.

### Hauptfunktionen

- Temperatur-Istwert über Modbus
- Einstellbarer Temperatur-Sollwert
- Hysterese-Regelung
- AUTO- und MANUAL-Betrieb
- Fan Command und Fan Feedback
- Alarmüberwachung
- Live-Trend für Temperatur und Sollwert
- FlowFuse Dashboard
- Modbus-TCP-Simulation
- Validierung und Datenfilterung
- Praktisches Troubleshooting

---

## Systemarchitektur

```text
Simulierter Temperatursensor
        ↓
Modbus TCP Server
        ↓
Node-RED Modbus Read
        ↓
Skalierung
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
Dashboard / Trend
```

Der Temperatur-Sollwert wird parallel über ein Modbus Holding Register verwaltet und in der Regelungslogik verwendet.

---

## Technologien

- Raspberry Pi 4
- Raspberry Pi OS 64-bit
- Node-RED
- Node.js
- Modbus TCP
- FlowFuse Dashboard
- JavaScript
- Linux / systemd
- Git / GitHub

---

## Modbus Mapping

| Signal | Modbus-Typ | Function Code | Adresse |
|---|---|---:|---:|
| Temperature Actual Value | Input Register | FC4 | 0 |
| Temperature Setpoint | Holding Register | FC3 / FC6 | 0 |
| Fan Command | Coil | FC1 / FC5 | 0 |
| Fan Feedback | Discrete Input | FC2 | 0 |

Der Modbus-Client und der simulierte Modbus-Server laufen auf demselben Raspberry Pi.

Beispiel:

```text
Host: 127.0.0.1
Port: 10502
Unit ID: 1
```

---

## Temperatur und Skalierung

Die Temperatur wird als Integer im Modbus Register gespeichert.

Beispiel:

```text
246 → 24.6 °C
230 → 23.0 °C
```

Die Werte werden in Node-RED auf ihre physikalische Einheit skaliert.

---

## Hysterese-Regelung

Zur Vermeidung von ständigem EIN-/AUS-Schalten wird eine Hysterese verwendet.

Bei einem Sollwert von:

```text
23 °C
```

gelten beispielsweise:

```text
Cooling ON  ≥ 24 °C
Cooling OFF ≤ 22 °C
```

Zwischen beiden Grenzen bleibt der vorherige Zustand erhalten.

---

## AUTO / MANUAL

### AUTO

Im AUTO-Modus entscheidet die Regelungslogik automatisch über den Fan Command.

```text
Temperature
    ↓
Hysteresis
    ↓
Cooling Demand
    ↓
Fan Command
```

### MANUAL

Im MANUAL-Modus kann der Ventilator unabhängig von der Temperatur manuell EIN oder AUS geschaltet werden.

---

## Command und Feedback

Das Projekt trennt bewusst zwischen:

```text
Command  = Was soll das Gerät tun?
Feedback = Was tut das Gerät tatsächlich?
```

Beispiel:

```text
Command  = ON
Feedback = ON
```

Normaler Betrieb.

```text
Command  = ON
Feedback = OFF
```

Mögliche Störung.

---

## Alarmüberwachung

Wenn der Fan Command aktiv ist, aber kein Fan Feedback erkannt wird, erzeugt das System einen Alarm.

```text
Fan Command = ON
Fan Feedback = OFF

→ Fan Fault Alarm
```

---

## Dashboard

Das FlowFuse Dashboard zeigt unter anderem:

- Temperature Actual Value
- Temperature Setpoint
- Fan Command
- Fan Feedback
- Alarm Status
- Temperatur-Trend
- Setpoint-Trend

---

## Troubleshooting

Während der Entwicklung wurden mehrere reale Fehler untersucht und behoben, unter anderem:

- Dashboard Reconnect Loop
- Socket.IO `too many attachments`
- ungültiger Setpoint / `NaN`
- Setpoint Reset nach Deploy
- mehrere aktive Temperaturquellen
- unnötige Binärdaten in Dashboard-Nachrichten

Eine wichtige Verbesserung war die Trennung von Steuerungs- und Visualisierungsdaten.

Für den Chart werden nur die benötigten Daten weitergegeben:

```javascript
return {
    payload: msg.payload,
    topic: msg.topic
};
```

Dadurch bleibt die Visualisierung stabil und unabhängig von zusätzlichen Modbus-Metadaten.

---

## Projektstatus

Das aktuelle System beinhaltet:

- ✅ Modbus TCP Simulation
- ✅ Temperature Measurement
- ✅ Setpoint Handling
- ✅ Scaling
- ✅ Hysteresis Control
- ✅ AUTO / MANUAL
- ✅ Fan Command
- ✅ Fan Feedback
- ✅ Alarm Logic
- ✅ Trend Visualization
- ✅ FlowFuse Dashboard
- ✅ Troubleshooting und Fehlerbehandlung

---

## Dokumentation

Eine ausführlichere technische Beschreibung mit detaillierten Erklärungen, Function Codes und Troubleshooting-Fällen befindet sich unter:




[📘 Detaillierte Projektdokumentation](docs/PROJECT_DOCUMENTATION.md)


---

## Mögliche Erweiterungen

Das Projekt kann später erweitert werden um:

- BACnet/IP
- Modbus RTU / RS485
- mehrere HVAC-Zonen
- Pumpen- und Ventilsteuerung
- Alarmhistorie
- Energie-Monitoring
- Datenbankintegration
- reale ESP32-Sensoren
- reale Aktoren
- SCADA-/BMS-Visualisierung
- Digital Twin

---

## Ziel des Projekts

Das Projekt wurde entwickelt, um typische Konzepte aus **BMS, HVAC und Gebäudeautomation** praktisch zu verstehen.

Der Fokus liegt auf:

- Datenfluss
- Modbus-Kommunikation
- Steuerungslogik
- Zustandsverwaltung
- Command / Feedback
- Alarmierung
- Visualisierung
- strukturierter Fehlersuche

---

# 🇬🇧 English

## Overview

This project simulates a small **Building Management System (BMS)** for an HVAC application.

It was built using **Raspberry Pi 4**, **Node-RED**, **Modbus TCP**, and **FlowFuse Dashboard** and demonstrates common building automation concepts.

### Main Features

- Temperature actual value via Modbus
- Adjustable temperature setpoint
- Hysteresis control
- AUTO and MANUAL operation
- Fan Command and Fan Feedback
- Alarm monitoring
- Live temperature and setpoint trends
- FlowFuse Dashboard
- Modbus TCP simulation
- Input validation and data filtering
- Practical troubleshooting

---

## System Architecture

```text
Simulated Temperature Sensor
        ↓
Modbus TCP Server
        ↓
Node-RED Modbus Read
        ↓
Scaling
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
Dashboard / Trend
```

The temperature setpoint is handled in parallel through a Modbus Holding Register and used by the control logic.

---

## Technologies

- Raspberry Pi 4
- Raspberry Pi OS 64-bit
- Node-RED
- Node.js
- Modbus TCP
- FlowFuse Dashboard
- JavaScript
- Linux / systemd
- Git / GitHub

---

## Modbus Mapping

| Signal | Modbus Type | Function Code | Address |
|---|---|---:|---:|
| Temperature Actual Value | Input Register | FC4 | 0 |
| Temperature Setpoint | Holding Register | FC3 / FC6 | 0 |
| Fan Command | Coil | FC1 / FC5 | 0 |
| Fan Feedback | Discrete Input | FC2 | 0 |

The Modbus client and simulated Modbus server run on the same Raspberry Pi.

Example:

```text
Host: 127.0.0.1
Port: 10502
Unit ID: 1
```

---

## Temperature and Scaling

Temperature values are stored as integers inside the Modbus registers.

Example:

```text
246 → 24.6 °C
230 → 23.0 °C
```

Node-RED scales these raw values into engineering units.

---

## Hysteresis Control

Hysteresis is used to prevent rapid ON/OFF switching.

For a setpoint of:

```text
23 °C
```

the example thresholds are:

```text
Cooling ON  ≥ 24 °C
Cooling OFF ≤ 22 °C
```

Between both thresholds, the previous cooling state is maintained.

---

## AUTO / MANUAL

### AUTO

In AUTO mode, the control logic automatically determines the Fan Command.

```text
Temperature
    ↓
Hysteresis
    ↓
Cooling Demand
    ↓
Fan Command
```

### MANUAL

In MANUAL mode, the operator can force the fan ON or OFF independently of the temperature.

---

## Command and Feedback

The project clearly separates:

```text
Command  = What should the equipment do?
Feedback = What is the equipment actually doing?
```

Example:

```text
Command  = ON
Feedback = ON
```

Normal operation.

```text
Command  = ON
Feedback = OFF
```

Possible equipment fault.

---

## Alarm Monitoring

If the Fan Command is active but no Fan Feedback is detected, the system generates an alarm.

```text
Fan Command = ON
Fan Feedback = OFF

→ Fan Fault Alarm
```

---

## Dashboard

The FlowFuse Dashboard displays:

- Temperature Actual Value
- Temperature Setpoint
- Fan Command
- Fan Feedback
- Alarm Status
- Temperature Trend
- Setpoint Trend

---

## Troubleshooting

Several real development issues were investigated and resolved, including:

- Dashboard reconnect loops
- Socket.IO `too many attachments`
- invalid setpoint / `NaN`
- setpoint reset after deploy
- multiple active temperature sources
- unnecessary binary data reaching the dashboard

An important improvement was separating control data from visualization data.

Only the required data is forwarded to the chart:

```javascript
return {
    payload: msg.payload,
    topic: msg.topic
};
```

This keeps the visualization stable and independent from additional Modbus metadata.

---

## Project Status

The current system includes:

- ✅ Modbus TCP Simulation
- ✅ Temperature Measurement
- ✅ Setpoint Handling
- ✅ Scaling
- ✅ Hysteresis Control
- ✅ AUTO / MANUAL
- ✅ Fan Command
- ✅ Fan Feedback
- ✅ Alarm Logic
- ✅ Trend Visualization
- ✅ FlowFuse Dashboard
- ✅ Troubleshooting and Fault Handling

---

## Documentation

More detailed technical documentation, including Modbus function codes and troubleshooting cases, is available in:

[📘 Detailed Project Documentation](docs/PROJECT_DOCUMENTATION.md)
```

---

## Possible Future Extensions

The project can later be extended with:

- BACnet/IP
- Modbus RTU / RS485
- multiple HVAC zones
- pump and valve control
- alarm history
- energy monitoring
- database integration
- real ESP32 sensors
- real actuators
- SCADA/BMS-style visualization
- Digital Twin

---

## Project Purpose

This project was created to gain practical experience with **BMS, HVAC, and building automation** concepts.

The main focus is on:

- data flow
- Modbus communication
- control logic
- state management
- command / feedback
- alarm handling
- visualization
- structured troubleshooting
