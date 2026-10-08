# ESP32-C6-PMS5303-ESPHome-
ESP32 C6 + PMS5303 (Indoor PM readings monitoring with ESPHome)

# PMS5303 ESP32-C6 Air Quality Monitor

## Overview

* Indoor particulate monitoring system built around:

  * **Seeed Studio XIAO ESP32-C6**
  * **Plantower PMS5303 particulate matter sensor**
  * **ESPHome**
  * **MQTT / Mosquitto**
  * **PostgreSQL**
  * **Grafana**
* Designed for continuous indoor air-quality monitoring.
* Current focus is **particulate matter (PM)** rather than VOCs, CO₂, etc.
* The system monitors:

  * **PM1.0**
  * **PM2.5**
  * **PM10**

## Hardware

* XIAO ESP32-C6
* PMS5303
* PMS5303 powered from the XIAO **5V** supply.
* UART communication at **9600 baud**.

### PMS5303 UART wiring

| PMS5303 | XIAO ESP32-C6 |
| ------- | ------------- |
| VCC     | 5V            |
| GND     | GND           |
| TXD     | D7 / GPIO17   |
| RXD     | D6 / GPIO16   |

* PMS5303 airflow inlet/outlet must remain unobstructed and exposed to room air.
* SET/RST pins are currently unused.

## ESPHome

* PMS5303 is configured through ESPHome's `pmsx003` component.
* PMS sensor update interval: **3 minutes**.

### Statistical filtering

A **median filter** is applied to PM1.0, PM2.5 and PM10:

```yaml
window_size: 5
send_every: 1
send_first_at: 1
```

Meaning:

* The PMS5303 produces one reading every **3 minutes**.
* A rolling window contains **5 readings**.
* Therefore the statistical window represents approximately **15 minutes** of measurements.
* The median of the 5 readings is used as the filtered value.
* Median filtering reduces the influence of short-lived spikes and anomalous readings.
* It does **not** perform sensor calibration or correct systematic sensor bias.

### MQTT publication timing

* A global counter is used to prevent publishing immediately after boot.
* The first MQTT publication occurs after the **5th filtered PMS reading**.
* At a 3-minute sensor interval, this means approximately **15 minutes after startup**.
* After that, a filtered reading is published approximately every **3 minutes**.
* The rolling median window continues updating normally.

### MQTT payload

The ESP32 publishes:

```json
{
  "sensor": "PMS5303",
  "pm1_0": 41,
  "pm2_5": 46,
  "pm10": 54
}
```

* Only the filtered PM values are sent.
* An ESPHome-generated AQI value is currently **not** used.
* AQI calculations, if required, are performed separately in Grafana.

## MQTT

### Broker

* Mosquitto runs on the `graphs` server.
* MQTT topic:

```text
sensors/air_quality/pms5303
```

### Secure remote MQTT

The ESP32 communicates with the broker over:

```text
MQTT over TLS 1.2
TCP port 8883
```

Connection:

```text
ESP32-C6
    ↓
MQTT + TLS 1.2
    ↓
Internet
    ↓
Mosquitto :8883
```

Overview:

```text
                 INTERNET
                    │
                    │ MQTT + TLS 1.2
                    │ TCP 8883
                    ▼
             ┌──────────────┐
             │   Mosquitto  │
             │              │
             │ :8883 public │
             │ :1883 local  │
             └──────┬───────┘
                    │
                    │ localhost
                    │ MQTT :1883
                    ▼
          pms5303-mqtt-postgres.py
                    │
                    ▼
              PostgreSQL
              air_quality
                    │
                    ▼
                 Grafana
```


## Grafana

Grafana reads the PostgreSQL data for visualization.

Typical measurements displayed:

* PM1.0
* PM2.5
* PM10
* Historical trends
* Latest PM readings
* PM2.5 AQI-style calculations where required

## Current Data Processing

The complete processing chain is:

```text
PMS5303
   │
   │ Reading every 3 minutes
   ▼
ESP32-C6
   │
   │ 5-reading rolling median
   │ ≈ 15-minute statistical window
   ▼
Filtered PM1.0 / PM2.5 / PM10
   │
   │ First publish after 5th reading
   │ Then every ~3 minutes
   ▼
MQTT over TLS 1.2
   │
   │ TCP 8883
   ▼
Mosquitto
   │
   │ localhost MQTT
   │ TCP 1883
   ▼
Python PostgreSQL bridge
   │
   ▼
PostgreSQL
   │
   ▼
Grafana
```

## Important Notes

* The median filter is intended to improve **measurement stability and robustness**, not calibration.
* A 5-reading window at a 3-minute interval represents approximately **15 minutes** of sensor history.
* The system currently does **not** apply an empirical PM calibration factor.
* PM2.5 is the primary value used for indoor air-quality assessment.
* The PMS5303's standard and atmospheric PM measurements are distinct; the current system uses the **atmospheric PM concentration** for the main PM readings.
* MQTT traffic from the ESP32 to the Internet-facing broker is encrypted using **TLS 1.2**.
* Port 1883 is restricted to localhost for internal bridge communication.
* The ESP32 automatically reconnects if the MQTT/TLS connection is temporarily interrupted.
