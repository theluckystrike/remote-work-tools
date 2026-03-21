---
layout: default
title: "MicroPython code for ESP32 desk sensor node"
description: "A technical guide to implementing desk sensors for hybrid offices. Covers hardware options, MQTT data pipelines, API integrations, and code examples"
date: 2026-03-16
author: "Remote Work Tools Guide"
permalink: /best-desk-sensor-technology-for-hybrid-offices-tracking-real/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools, best-of]
---


{% raw %}

Desk sensor technology enables hybrid offices to track real-time occupancy and optimize space use by detecting whether desks are in use. ESP32-based microcontrollers combined with PIR motion sensors and pressure sensors provide reliable occupancy data through MQTT pipelines into InfluxDB. This architecture enables REST APIs for desk booking systems and heat maps showing which areas are actually used, supporting hot-desking policies and smart real estate decisions.

## Understanding Desk Sensor Technologies

Desk sensors detect whether a desk or workstation is currently in use. Several technologies power these systems, each with distinct advantages and trade-offs.

**Pressure-based sensors** use force-sensitive resistors (FSRs) placed under desk mats or chair cushions. When someone sits, the pressure triggers a detection event. These sensors are inexpensive and easy to deploy but require physical contact and may not detect someone standing at a standing desk.

**Infrared (PIR) sensors** detect motion through heat signatures. Passive infrared sensors consume minimal power and work well for detecting human presence within a defined zone. They're non-intrusive but can struggle in warm environments or with stationary individuals.

**Ultrasonic distance sensors** measure the presence of objects by bouncing sound waves off surfaces. They're effective for detecting both sitting and standing positions but may produce false positives from bags or coats left at desks.

**Capacitive proximity sensors** detect the electrical field changes caused by human bodies. These can work through desk surfaces without direct contact but require careful calibration to distinguish between humans and other objects.

**Camera-based solutions** with privacy-preserving edge processing offer the richest data but raise legitimate privacy concerns that require careful policy consideration and employee communication.

## Hardware Implementation

For a practical deployment, consider ESP32-based microcontroller boards as the sensor hub. They offer WiFi connectivity, sufficient processing power for sensor data aggregation, and low power consumption for battery-backed deployments.

### Sensor Configuration Example

```python
# MicroPython code for ESP32 desk sensor node
from machine import Pin, PWM
import network
import umqtt.simple as mqtt
import json
import time

# Configuration
WIFI_SSID = 'your-office-network'
WIFI_PASSWORD = 'your-password'
MQTT_BROKER = 'mqtt://office-sensors.local'
DESK_ID = 'desk-101'

# Initialize sensors
pir_sensor = Pin(14, Pin.IN)  # PIR motion detector
fsr_pin = Pin(15, Pin.IN)      # Force-sensitive resistor

def read_occupancy():
    """Read combined sensor data for occupancy state"""
    motion_detected = pir_sensor.value() == 1

    # Simple pressure threshold
    pressure = fsr_pin.value()

    # Occupancy logic: either motion OR pressure indicates use
    occupied = motion_detected or pressure

    return {
        'desk_id': DESK_ID,
        'occupied': occupied,
        'motion': motion_detected,
        'pressure': pressure,
        'timestamp': time.time()
    }

def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(WIFI_SSID, WIFI_PASSWORD)
    while not wlan.isconnected():
        time.sleep(1)
    return wlan

def main():
    wlan = connect_wifi()
    client = mqtt.MQTTClient('desk_sensor_' + DESK_ID, MQTT_BROKER)
    client.connect()

    while True:
        data = read_occupancy()
        client.publish(f'office/desks/{DESK_ID}', json.dumps(data))
        time.sleep(30)  # Publish every 30 seconds
```

This example demonstrates a combined sensor approach that reduces false positives by requiring either motion or pressure detection. Adjust polling intervals based on your responsiveness requirements—shorter intervals provide real-time data but increase network traffic and power consumption.

## Data Pipeline Architecture

Building an occupancy tracking system requires reliable data collection, storage, and presentation layers. MQTT serves as the message broker for sensor-to-server communication, providing the lightweight pub/sub model ideal for IoT deployments.

### MQTT to InfluxDB Pipeline

```javascript
// Node.js: MQTT consumer that stores desk occupancy in InfluxDB
const mqtt = require('mqtt');
const { InfluxDB, Point } = require('influxdb');
const client = mqtt.connect('mqtt://localhost:1883');

// Configure InfluxDB
const influx = new InfluxDB({
  host: 'localhost',
  database: 'office_sensors',
  schema: [
    {
      measurement: 'desk_occupancy',
      fields: {
        occupied: 'boolean',
        motion: 'boolean',
        pressure: 'boolean'
      },
      tags: ['desk_id']
    }
  ]
});

client.on('connect', () => {
  console.log('Connected to MQTT broker');
  client.subscribe('office/desks/#');
});

client.on('message', async (topic, message) => {
  const deskId = topic.split('/').pop();
  const data = JSON.parse(message.toString());

  const point = new Point('desk_occupancy')
    .tag('desk_id', deskId)
    .booleanField('occupied', data.occupied)
    .booleanField('motion', data.motion)
    .booleanField('pressure', data.pressure)
    .timestamp(new Date(data.timestamp * 1000));

  try {
    await influx.writePoint(point);
  } catch (error) {
    console.error('Error writing point:', error);
  }
});
```

This pipeline collects sensor readings and stores them in InfluxDB, a time-series database optimized for sensor data. The schema uses tags for desk identification (indexed for efficient queries) and fields for the raw sensor readings.

### Occupancy State Aggregation

Raw sensor data needs aggregation into meaningful occupancy metrics. Calculate desk use rates over time windows to understand usage patterns.

```python
# Python: Query and analyze desk occupancy
from influxdb_client import InfluxDBClient
from datetime import datetime, timedelta

client = InfluxDBClient(
    url='http://localhost:8086',
    token='your-token',
    org='your-org'
)

def get_desk_utilization(desk_id: str, start: datetime, end: datetime):
    """Calculate desk utilization percentage for a time period"""
    query = f'''
    from(bucket: "office_sensors")
      |> range(start: {start.isoformat()}, stop: {end.isoformat()})
      |> filter(fn: (r) => r._measurement == "desk_occupancy")
      |> filter(fn: (r) => r.desk_id == "{desk_id}")
      |> filter(fn: (r) => r._field == "occupied")
      |> aggregateWindow(every: 1h, fn: mean)
    '''

    result = client.query_api().query(query)

    # Calculate utilization from hourly averages
    readings = []
    for table in result:
        for record in table.records:
            readings.append(record.get_value())

    if readings:
        utilization = sum(readings) / len(readings) * 100
        return round(utilization, 1)
    return 0.0

# Example: Get utilization for the past week
week_start = datetime.now() - timedelta(days=7)
for desk in ['desk-101', 'desk-102', 'desk-103']:
    util = get_desk_utilization(desk, week_start, datetime.now())
    print(f'{desk}: {util}% utilized')
```

## API Integration for Space Management

Modern desk booking systems need real-time availability data. Expose occupancy data through a REST API or GraphQL endpoint for integration with workspace management platforms.

```javascript
// Express.js: Simple API for desk availability
const express = require('express');
const { InfluxDB } = require('influxdb');
const app = express();

const influx = new InfluxDB({ host: 'localhost', database: 'office_sensors' });

app.get('/api/desks/availability', async (req, res) => {
  const { floor, zone } = req.query;

  // Get latest occupancy for all desks
  const query = `
    from(bucket: "office_sensors")
      |> range(start: -5m)
      |> filter(fn: (r) => r._measurement == "desk_occupancy")
      |> filter(fn: (r) => r._field == "occupied")
      |> last()
  `;

  const result = await influx.query(query);

  // Transform to availability response
  const desks = {};
  for (const table of result) {
    for (const record of table.records) {
      const deskId = record.desk_id;
      desks[deskId] = {
        id: deskId,
        occupied: record._value === 1,
        last_updated: record._time
      };
    }
  }

  res.json({
    timestamp: new Date().toISOString(),
    floors: [{ floor, zones: [{ zone, desks }] }]
  });
});

app.listen(3000, () => {
  console.log('Desk availability API running on port 3000');
});
```

## Practical Deployment Considerations

Deploying desk sensors at scale requires addressing several operational concerns. Battery-powered sensors need regular maintenance—plan for 6-12 month replacement cycles depending on polling frequency. WiFi-connected sensors may experience connectivity issues in buildings with dense networks; consider mesh networking or wired connections for critical deployments.

Calibration significantly impacts sensor accuracy. PIR sensors need clear sightlines without obstruction. FSR sensors require appropriate sensitivity settings for different body weights and seating positions. Test extensively in your actual office environment before full deployment.

Privacy remains paramount. Clearly communicate sensor placement and data usage to employees. Store occupancy data in aggregate form rather than tracking individuals. Many jurisdictions regulate employee monitoring—consult legal counsel for compliance requirements specific to your location.


## Related Articles

- [Best Desk Booking App for Hybrid Offices Using Microsoft 365](/remote-work-tools/best-desk-booking-app-for-hybrid-offices-using-microsoft-365/)
- [Best Visitor Management System for Hybrid Offices Tracking W](/remote-work-tools/best-visitor-management-system-for-hybrid-offices-tracking-w/)
- [Node.js and npm](/remote-work-tools/claude-code-npm-package-development-guide/)
- [Best Hot Desking Software for Hybrid Offices with Under 100](/remote-work-tools/best-hot-desking-software-for-hybrid-offices-with-under-100-employees-2026/)
- [Upload to your analytics backend](/remote-work-tools/best-occupancy-analytics-platform-for-hybrid-offices-trackin/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)
{% endraw %}
