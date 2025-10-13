# HestiaAI FlueMonitor  
_ProMicro nRF52840 + MAX6675 BLE Thermocouple Sensor_

---

## Overview

**FlueMonitor** is a low-power BLE thermocouple node used by the HestiaAI system to measure boiler flue temperature and battery voltage.  
It automatically enables BLE broadcasting when the boiler is hot and sleeps when idle, sending data to Home Assistant via a **Waveshare ESP32-S3-ETH** proxy running **ESPHome**.

---

## Hardware

- **MCU:** ProMicro-form nRF52840 board (SuperMini derivative)  
- **Sensor:** MAX6675 thermocouple amplifier  
- **Power:** 2 × 2500 mAh Li-ion cells (parallel, 3.7 V nominal)  
- **Charging:** On-board 100 mA (jumper to 300 mA with `BOOST`)  
- **Gateway:** Waveshare ESP32-S3-ETH (Ethernet-based ESPHome BLE proxy)

### Pin connections

| MAX6675 | nRF52840 Pin | Notes |
|----------|--------------|-------|
| VCC | P0.22 | Switched VCC (MAX_POWER_PIN) |
| SCK | P0.17 | SPI clock |
| CS | P0.24 | SPI chip select |
| SO | P0.20 | SPI data out |
| GND | GND | Common ground |

---

## Firmware Features

- Reads flue temperature every **5 s**.  
- Reads battery voltage via **SAADC VDDHDIV5** every **hour**.  
- **BLE ON** when temperature ≥ 30 °C; **BLE OFF** below 28 °C (hysteresis).  
- Broadcasts two service-data blocks:  
  - `0x181A` – Environmental Sensing (temperature)  
  - `0x180F` – Battery Service (voltage)  
- Performs a **12 s “VBAT burst”** each hour when idle so Home Assistant still sees voltage updates.  
- Typical current draw:  
  - **Idle:** 0.4 – 1 mA  
  - **BLE active:** ≈ 14 mA  

---

## Building and Flashing

1. Open the PlatformIO project in VS Code.  
2. Select environment: `nicenano` (or any Adafruit nRF52 target).  
3. Connect board via USB-C and run:

   ```bash
   pio run -t upload
   ```

4. **Important:** remove any blocking `while (!Serial);` lines so the device boots from battery.  
5. Once flashed, the board boots automatically on Li-ion power.

---

## ESPHome Proxy Configuration  
_(Waveshare ESP32-S3-ETH BLE Gateway)_

```yaml
esp32_ble_tracker:
  on_ble_advertise:
    - mac_address: D0:A2:A7:5B:7A:34
      then:
        - lambda: |-
            static uint32_t last_rssi = 0;
            uint32_t now = millis();
            if (now - last_rssi > 300000) {  # 5 min
              last_rssi = now;
              id(flue_rssi).publish_state(x.get_rssi());
            }

  on_ble_service_data_advertise:
    - mac_address: D0:A2:A7:5B:7A:34
      service_uuid: "181A"    # Temperature
      then:
        - lambda: |-
            if (x.size() >= 2) {
              uint16_t raw = x[0] | (x[1] << 8);
              id(flue_temp).publish_state(raw / 100.0f);
            }

    - mac_address: D0:A2:A7:5B:7A:34
      service_uuid: "180F"    # Battery voltage
      then:
        - lambda: |-
            if (x.size() >= 2) {
              uint16_t raw = x[0] | (x[1] << 8);
              id(flue_voltage).publish_state(raw / 100.0f);
            }

sensor:
  - platform: template
    id: flue_temp
    name: "FlueMonitor Temperature"
    unit_of_measurement: "°C"
    device_class: temperature
    accuracy_decimals: 2

  - platform: template
    id: flue_voltage
    name: "FlueMonitor Battery Voltage"
    unit_of_measurement: "V"
    device_class: voltage
    accuracy_decimals: 2

  - platform: template
    id: flue_rssi
    name: "FlueMonitor RSSI"
    unit_of_measurement: "dBm"
    icon: mdi:signal
    accuracy_decimals: 0
```

---

## Power Behaviour Summary

| Mode | Description | Avg Current |
|------|--------------|-------------|
| Idle (radio off) | MCU in soft sleep, 5 s temp reads | 0.4 – 1 mA |
| Sensor active | MAX6675 powered ~ 200 ms | +30 mA |
| BLE advertising | Continuous, flue > 30 °C | ≈ 14 mA |
| Hourly VBAT burst | 12 s at 14 mA | — |
| Two × 2500 mAh cells | Expected runtime ≈ months | — |

---

## Troubleshooting

- If the board won’t start on battery, ensure the serial init is non-blocking.  
- Add a 47 µF cap across **3V3 ↔ GND** to smooth USB ↔ battery transitions.  
- Press **RESET** once after unplugging USB to clear latch-up.  
- Confirm the MAC address in the YAML matches the actual device (check in ESPHome logs).  
- RSSI entity updates every 5 min to reduce database writes.  

---

## License & Attribution

Part of the **HestiaAI Project**  
© 2025 John Warwicker / HestiaAI
