# Enervent ECC05 RS485 Home Assistant Bridge

This project provides a simple ESP32-based bridge that connects legacy Enervent ECC05 ventilation control systems to Home Assistant using MQTT.

The bridge listens to the RS485 control bus used between the Enervent main controller and the ECC05 wall panels. It decodes state frames from the system and publishes them to MQTT, while also allowing Home Assistant to inject valid control events back onto the bus.

The goal of this project is to make older Enervent ventilation systems usable in modern smart home setups without modifying the original hardware.

This implementation was developed by reverse engineering the proprietary RS485 protocol used by the ECC05 panels.

## Features

Current functionality includes:

• Reading the fan speed level directly from the system
• Changing fan speed using valid control events
• Reading heat recovery (HR) state
• Toggling heat recovery from Home Assistant
• MQTT publishing of system state
• MQTT command interface for control
• Automatic WiFi and MQTT reconnection
• Bus protection with command debounce to prevent event flooding
• Serial debugging output for protocol analysis and troubleshooting

The system operates entirely passively on the RS485 bus and does not interfere with normal keypad operation.

## Hardware Requirements

To build the bridge you need the following components.

### ESP32 microcontroller

An ESP32 is required because the Enervent bus runs continuously and requires stable serial parsing while maintaining WiFi and MQTT connections.

Tested hardware:

• ESP32-C3 SuperMini
• ESP32-C3 SuperMini Plus

Both versions work well. The Plus version simply includes a larger antenna.

### RS485 interface module

An isolated RS485 interface is strongly recommended to protect the ESP32 and your HVAC system.

Recommended module:

• **XY-S485 isolated RS485 to TTL module**

The module automatically handles transmit direction switching which simplifies firmware.

### Wiring

Panel connector cable

```
You need an rj9 (4P4C) cable for connecting to the bus
```

The connection between the ESP32 and the RS485 module is simple.

```
ESP32 GPIO20 → RX (module RXD)
ESP32 GPIO21 → TX (module TXD)
ESP32 GND → GND
ESP32 3.3V → VCC
```

The RS485 side of the module connects directly to the Enervent control bus.

```
RS485 A → Enervent bus A
RS485 B → Enervent bus B
```

The Enervent control bus runs at **38400 baud, 8N1**.

## Firmware

The firmware performs three main tasks:

1. Listening to the RS485 bus and parsing frames
2. Publishing decoded state information to MQTT
3. Injecting valid control events when MQTT commands are received

Important frame types identified during reverse engineering:

TQF 0A
System state broadcast frame

TQF C8
Control event frame

TRB
Event acknowledgement

TRC
State acknowledgement

CRC validation uses **CRC-16 XMODEM**.

## MQTT Topics

The firmware exposes the following MQTT interface.

### State topics

```
enervent/fan_level/state
enervent/hr/state
enervent/availability
```

### Command topics

```
enervent/fan_next/set
enervent/hr_toggle/set
```

### Debug topic

```
enervent/debug
```

Debug messages and protocol information are published here when MQTT is connected.

## Home Assistant Integration

Home Assistant can interact with the system through standard MQTT entities.

Example entity types:

• Fan level sensor
• Fan speed button
• HR state binary sensor
• HR toggle button

Automations can then be used to implement scheduling or smart control behavior.

## Serial Debugging

The firmware outputs useful debugging information to the serial port.

Serial monitor settings:

```
Baud rate: 115200
```

Debug output includes:

• detected frame types
• counters from bus events
• injected commands
• connection status

This is very helpful when analyzing the protocol or diagnosing bus issues.

## Safety Notes

This project interacts with the control bus of a ventilation system. While the RS485 interface is electrically isolated, incorrect wiring or experimental firmware could potentially disrupt system operation.

Use this project at your own risk.

The author is not responsible for damage to HVAC equipment or property.

## Project Status

The following features are confirmed working:

• Fan state reading
• Fan speed control
• HR state detection
• HR toggle control
• MQTT integration
• Home Assistant automation control

Future work may include:

• additional sensor decoding
• after-heating control
• full protocol documentation
• native Home Assistant fan entity support

## Acknowledgements

This project exists thanks to extensive protocol analysis of the Enervent ECC05 control bus using logic analyzer captures and real system testing.

Older ventilation systems should not become obsolete simply because their control interfaces are proprietary.
