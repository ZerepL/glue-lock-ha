# Glue GL11102B — ESPHome MQTT Door Lock Control

An unofficial hardware modification for the discontinued Glue GL11102B smart lock, using an ESP32-C3 and MQTT to integrate the lock with Home Assistant, Node-RED, or other MQTT-based systems.

The original Glue cloud service is no longer available. This project replaces the lock's original controller functionality with an ESP32-C3 running ESPHome while retaining the original BLE module for battery monitoring.

**This project is not affiliated with, endorsed by, or supported by Glue.**

## What You Need

- **Glue GL11102B** (any condition — BLE chip stays in place)
- **ESP32-C3 Super Mini** (or compatible ESP32-C3)
- [Mosquitto](https://mosquitto.org/) or any MQTT broker
- **ESPHome** 

## PCB Solder Points

See the photo below. The 4 numbered solder points are marked directly on it:

1. Motor terminals
2. Button terminal
3. VCC (3.3V), self regulated from the lock 
4. GND

![Glue Lock PCB — 4 solder points numbered 1-4](docs/glue-lock-pcb.png)

## Pinout (ESP32-C3)

| Pin   | Function            | Lock connection     | PCB pad |
|-------|---------------------|---------------------|---------|
| GPIO0 | Button input        | Button terminal     | Pad #2  |
| GPIO1 | Motor B (unlock)    | Motor terminals     | Pad #1  |
| GPIO2 | Motor A (lock)      | Motor terminals     | Pad #1  |
| VIN   | VCC (3.3V)          | TP_VCC              | Pad #3  |
| GND   | Ground              | Common ground       | Pad #4  |

## MQTT Topics

| Topic | Direction | Payload | Description |
| ------------------------- | --------- | ----------- | ------------------------- |
| `glue-lock/command` | In | `LOCK` | Lock the door |
| `glue-lock/command` | In | `UNLOCK` | Unlock the door |
| `glue-lock/state` | Out | `LOCKING` | Lock operation in progress |
| `glue-lock/state` | Out | `LOCKED` | Lock operation completed |
| `glue-lock/state` | Out | `UNLOCKING` | Unlock operation in progress |
| `glue-lock/state` | Out | `UNLOCKED` | Unlock operation completed |
| `glue-lock/state` | Out | `UNKNOWN` | Physical lock position cannot be determined |
| `glue-lock/availability` | Out | `online` | ESP32 is connected |
| `glue-lock/availability` | Out | `offline` | ESP32 is unavailable |

The `glue-lock/state` topic uses retained MQTT messages so Home Assistant can recover the most recently reported state.

The ESP32 uses MQTT Last Will and Testament to publish `offline` if it unexpectedly disconnects.

## Home Assistant

The ESP32 can be integrated with Home Assistant using its MQTT Lock integration.

The motor outputs are kept as internal ESPHome switches and are not exposed as independently controllable entities. This prevents the motors from being operated directly from Home Assistant and ensures that lock/unlock operations go through the firmware safety logic.

Example Home Assistant configuration:

```yaml
    mqtt:
      - lock:
          name: "Glue Lock"
          unique_id: "glue_lock"

          command_topic: "glue-lock/command"
          state_topic: "glue-lock/state"

          availability:
            - topic: "glue-lock/availability"

          payload_lock: "LOCK"
          payload_unlock: "UNLOCK"

          state_locked: "LOCKED"
          state_unlocked: "UNLOCKED"
          state_locking: "LOCKING"
          state_unlocking: "UNLOCKING"

          optimistic: false
          qos: 1
```

### 3. Add this after the Home Assistant section

## Motor Safety and Recovery

The firmware includes software protection around motor operation:

- Motor A and Motor B are explicitly switched off before the opposite motor is activated.
- Motor outputs use `restore_mode: ALWAYS_OFF`.
- Motor outputs are internal ESPHome switches and are not exposed directly to Home Assistant.
- A `motor_busy` flag prevents another lock/unlock operation from starting while a motor sequence is running.
- MQTT commands received while the motor is busy are ignored.
- The physical button is also ignored while a motor sequence is running.

### Reboot and Power-Loss Recovery

The firmware does not automatically resume a motor operation after a reboot.

On boot, both motor outputs are forced OFF and the lock state is set to `UNKNOWN`.

This is intentional because the ESP32 cannot determine the physical position of the lock if power is lost during a motor operation.

For example:

```text
LOCKING
   ↓
Motor running
   ↓
ESP32 loses power
   ↓
ESP32 reboots
   ↓
Both motors OFF
   ↓
UNKNOWN
```

### Battery monitoring note

Battery status currently flows via Home Assistant polling the lock's original BLE module over GATT (opcode `0x57`). This requires a Bluetooth proxy in HA.

## Project Structure
```
├── esp/                # ESPHome firmware (published)
│   ├── glue-lock.yaml  # Main firmware config
│   ├── secrets.yaml.example
├── docs/               # Reference photos and images
│   └── glue-lock-pcb.png
├── README.md
└── .gitignore
```

## Configuration and Secrets

The ESPHome configuration uses `secrets.yaml` for credentials and other environment-specific values.

Example:

```yaml
wifi_ssid: "YOUR_WIFI_SSID"
wifi_password: "YOUR_WIFI_PASSWORD"

mqtt_broker: "YOUR_MQTT_BROKER"
mqtt_username: "YOUR_MQTT_USERNAME"
mqtt_password: "YOUR_MQTT_PASSWORD"

glue_lock_api_key: "GENERATE_A_NEW_API_KEY"
glue_lock_ota_password: "GENERATE_A_STRONG_PASSWORD"
glue_lock_ap_password: "GENERATE_A_STRONG_PASSWORD"
```

## What Else Could Be Done

A few things are already wired but unused in the current firmware:

- **Battery sensing from ESP32** — instead of relying on HA's BLE poll, you could tap the 7.2V battery rail through a voltage divider into an ESP32 ADC pin and publish `glue-lock/battery/voltage` via MQTT directly.
- **LED feedback** — the PCB has LED test points (TP_D201, TP_D202, TP_D301) driven through a 74LV595A shift register. Controlling them from the ESP would give visual lock/unlock status at the door. Tracing the SPI lines to the shift register is needed before wiring up.
- **Better usage of the button** — I still dont took the time to work in the button, so that might be a good update for the future
- **Lock status switch** — Almost sure that is possible to install a micro switch to check if the lock is locked on unlocked

---

## Safety Notes

### ⚠️ Disclaimer

Use this project at your own risk.

Modifying the lock involves soldering and electrical work and may damage the lock, ESP32, battery, PCB, or other components if performed incorrectly. The information in this repository is provided without any guarantee that the modification will work with every unit or installation.

The motor control timing is based on testing with the author's lock. **Verify the required motor run time for your own lock before relying on the modification for normal operation.**

- Motor run time currently defaults to 7.5 seconds forward + 1 second reverse to release the tension if you try to do it manually.
- Use an appropriate motor driver stage; do not connect the motor directly to ESP32 GPIO pins.
- The original BLE module remains installed for battery monitoring.
- The ESP32-C3 uses 3.3 V logic.
- The lock's battery can provide substantially higher voltage than the ESP32 GPIO level; verify voltages before connecting anything.

## Contributing

This project is a community effort and contributions are very welcome. If you see any improvement or just want to help, feel free to open an issue or pull request.

Any help with testing, documentation, reverse engineering, or improving the project is appreciated.

## Reverse Engineering

This project documents observations made while examining and modifying the Glue GL11102B hardware and its communication interfaces. It does not distribute the original Glue firmware or proprietary software.

The goal is to document a hardware modification that allows owners of otherwise-discontinued hardware to continue using their device locally.
```
