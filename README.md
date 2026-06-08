# Plant Monitor

An ESP32-based smart plant monitoring system with automatic watering, Home Assistant integration via MQTT, and deep sleep power management. Consists of two independent units: the **Plant Monitor** (sensor node) and the **Watering Station** (pump actuator).

---

## System Overview

```
[Plant Monitor ESP32]  --MQTT-->  [MQTT Broker]  --MQTT-->  [Watering Station ESP32]
        |                              |
   Reads sensors                Home Assistant
   Triggers watering cmd         Dashboard / Automations
```

The plant monitor reads sensors and publishes data. When soil moisture drops below the threshold it sends a watering command over MQTT. The watering station subscribes to that command and activates the pump. Both units connect independently to the same WiFi network and MQTT broker.

---

## Hardware

### Plant Monitor

| Component | GPIO |
|-----------|------|
| Soil moisture sensor (analog) | 0 |
| DS18B20 temperature sensor (OneWire) | 1 |
| LDR light sensor (analog) | 2 |
| Buzzer | 3 |

- ESP32 with deep sleep support
- DS18B20 requires a 4.7k pull-up resistor on the data line

### Watering Station

| Component | GPIO |
|-----------|------|
| Pump relay / MOSFET | 0 |
| Manual trigger button | 1 |

- Separate ESP32 unit
- Button is active LOW (internal pull-up enabled)
- Manual button press activates the pump for the configured duration regardless of MQTT

---

## Features

### Plant Monitor

**Sensors**
- Soil moisture (analog ADC)
- Ambient temperature via DS18B20 (Dallas OneWire)
- Light presence detection (LDR with configurable threshold)

**Power Management**
- Deep sleep during dark periods (night), waking every 30 minutes to check and publish sensor data
- Stays awake during daylight and publishes every 60 seconds
- RTC memory (`RTC_DATA_ATTR`) persists boot count, sleep time accumulation, last watering time, and last low-moisture beep time across sleep cycles

**Automation**
- Automatic watering triggered when moisture falls below threshold (`MOISTURE_THRESHOLD = 2900`)
- 5-minute cooldown between watering cycles, tracked across deep sleep via RTC memory
- Low moisture buzzer alert every 5 minutes when soil is dry, also tracked across sleep

**Configuration Portal**
- On cold boot: opens WiFi AP (`Smart-Pot`, no password) for 3 minutes
- Captive portal at `192.168.4.1` for entering WiFi SSID/password and MQTT broker settings
- Settings persisted to NVS (`Preferences`)
- On subsequent wakeups from sleep: connects directly using saved credentials

### Watering Station

- Subscribes to `smartpot/water_command`; activates pump for 5 seconds on receiving `"1"`
- Manual button override: activates pump for 5 seconds regardless of MQTT state
- Same captive portal setup flow as the plant monitor for WiFi + MQTT configuration
- WiFi state machine with automatic retry every 15 seconds on connection failure

---

## MQTT Topics

| Topic | Direction | Description |
|-------|-----------|-------------|
| `smartpot/temperature` | Publish | Temperature in degrees C (string, 2 decimal places) |
| `smartpot/soil_moisture` | Publish | Raw ADC value from moisture sensor |
| `smartpot/sunlight_presence` | Publish | `1` = light, `0` = dark |
| `smartpot/last_watering_time` | Publish (retained) | ISO timestamp of last watering event |
| `smartpot/water_command` | Publish / Subscribe | Plant monitor sends `"1"` to trigger watering |

---

## WiFi / MQTT Configuration

Both units use the same captive portal flow:

1. Power on (cold boot)
2. Connect to the AP (`Smart-Pot` or `Watering-station`)
3. Browser opens the configuration page automatically (captive portal) or navigate to `192.168.4.1`
4. Fill in WiFi credentials and MQTT broker details
5. Submit - the unit connects and begins operation

Configuration is saved to NVS and survives reboot and deep sleep.

---

## Deep Sleep Behavior (Plant Monitor)

| Condition | Behavior |
|-----------|----------|
| Cold boot | AP mode for configuration, then normal operation |
| Dark (LDR below threshold) | Sleep 30 minutes, wake, send data, sleep again |
| Light (LDR above threshold) | Stay awake, send data every 60 seconds |
| MQTT connected + data sent + 15 s minimum awake | Eligible to sleep (if dark) |

Timers that must survive sleep are accumulated in `rtcData.totalSleepTime` so that intervals like watering cooldown and beep suppression work correctly across wakeup cycles.

---

## Thresholds and Timing

### Plant Monitor

| Parameter | Value |
|-----------|-------|
| Moisture threshold (dry) | 2900 (ADC raw) |
| Sunlight threshold | 1500 (ADC raw) |
| Watering cooldown | 5 minutes |
| Low moisture beep interval | 5 minutes |
| Day publish interval | 60 seconds |
| Night sleep interval | 30 minutes |
| AP timeout (cold boot) | 3 minutes |
| WiFi retry interval | 15 seconds |

### Watering Station

| Parameter | Value |
|-----------|-------|
| Pump on duration | 5 seconds |
| AP timeout | 2 minutes |
| WiFi retry interval | 15 seconds |
| Status log interval | 10 seconds |

---

## Home Assistant Integration

Add the following to `configuration.yaml` to expose all entities:

```yaml
mqtt:
  sensor:
    - name: "Smart Pot Temperature"
      state_topic: "smartpot/temperature"
      unit_of_measurement: "C"
      device_class: temperature

    - name: "Smart Pot Soil Moisture"
      state_topic: "smartpot/soil_moisture"

    - name: "Smart Pot Last Watering"
      state_topic: "smartpot/last_watering_time"

  binary_sensor:
    - name: "Smart Pot Sunlight"
      state_topic: "smartpot/sunlight_presence"
      payload_on: "1"
      payload_off: "0"
      device_class: light
```

Restart Home Assistant after editing `configuration.yaml`.

---

## Dependencies

### Plant Monitor

- `OneWire`
- `DallasTemperature`
- `PubSubClient`
- `WiFi`, `WebServer`, `DNSServer`, `Preferences`, `esp_sleep` (built-in ESP32 Arduino)

### Watering Station

- `PubSubClient`
- `WiFi`, `WebServer`, `DNSServer`, `Preferences` (built-in ESP32 Arduino)

---

## Notes

- Both units use `configTime(3600, 3600, "pool.ntp.org")` for NTP time, providing CET offset with DST — adjust the second argument to `0` outside DST periods or use a proper POSIX timezone string for automatic DST handling
- The DS18B20 returns `DEVICE_DISCONNECTED_C` on failure; this is checked before publishing to avoid sending garbage temperature values
- Watering station button is debounced implicitly by the `pumpActive` flag preventing re-trigger while the pump is running
