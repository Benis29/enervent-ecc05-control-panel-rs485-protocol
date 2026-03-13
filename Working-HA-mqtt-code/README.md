# Enervent ECC05 RS485 Home Assistant Bridge

This project connects an Enervent ECC05 ventilation system to Home Assistant using an ESP32 and MQTT.

The bridge listens to the RS485 control bus used between the Enervent main controller and ECC05 wall panels. It reads the current fan state and HR state from the bus and can also send valid control events back onto the bus.

This allows Home Assistant to monitor and control the ventilation system without modifying the original Enervent electronics.

## What this project can do

Currently confirmed working features:

- Read current fan level from the bus
- Change fan speed
- Read heat recovery (HR) state
- Toggle heat recovery
- Publish state to MQTT
- Control the system from Home Assistant automations

## Hardware you need

You need the following hardware.

### ESP32

Recommended:

- ESP32-C3 SuperMini
- ESP32-C3 SuperMini Plus

Other ESP32 boards may work but the code is written for ESP32-C3.

### RS485 to TTL module

Recommended:

- XY-S485 or XYS485 isolated RS485 to TTL module

Using an isolated module is strongly recommended to protect the ESP32 and the HVAC electronics.

### RJ9 (4P4C) cable

You need access to the ECC05 panel bus.

The easiest method is connecting behind a wall panel using the RJ9 cable.

Options:

- RJ9 cable with cut end
- RJ9 breakout adapter
- Access wires behind the wall panel

### Computer

You need a computer with Arduino IDE to upload the firmware.

### Home Assistant

A running Home Assistant installation is required.

### Mosquitto MQTT broker

Install the Mosquitto broker add-on inside Home Assistant.

## Warning

You are connecting custom electronics to a ventilation control system.

Incorrect wiring may stop communication or damage electronics.

Use an isolated RS485 module and double check wiring before powering the system.

You do this at your own risk.

## ECC05 bus overview

The ECC05 wall panels communicate with the main controller using a half-duplex RS485 bus.

Bus settings:

- 38400 baud
- 8 data bits
- no parity
- 1 stop bit

The bus normally carries:

- state broadcasts from the controller
- control events from the wall panels

This bridge acts like an additional virtual wall panel.

## Wiring ESP32 to RS485 module

Connect the ESP32 to the XY-S485 module like this:

ESP32 pin → RS485 module

GPIO20 → RXD
GPIO21 → TXD
GND → GND
3.3V → VCC

The firmware uses:

RX pin = GPIO20
TX pin = GPIO21

## Wiring RS485 module to the Enervent bus

On the RS485 side of the module:

Module A → Enervent A
Module B → Enervent B

Do not guess these wires. Identify them first.

## Finding A and B wires

The RJ9 cable has four wires.

Two wires are power and two are the RS485 pair.

Recommended method:

1. Open the wall panel carefully
2. Locate the RS485 chip on the PCB
3. Trace the two RS485 lines to the cable
4. Use a multimeter continuity test to identify the wires

If A and B are swapped, nothing usually breaks but communication will not work.

If the bus shows nonsense data, swap A and B and test again.

## Installing Mosquitto MQTT broker

1. Open Home Assistant
2. Go to Settings
3. Open Add-ons
4. Install Mosquitto Broker
5. Start the add-on
6. Create a username and password

You will need:

- MQTT username
- MQTT password
- Home Assistant IP address

## Arduino libraries required

Install this library:

PubSubClient by Nick O'Leary

Also install the ESP32 board support package in Arduino IDE.

## Serial monitor

The firmware outputs debug information.

Use these settings:

Baud rate: 115200

This allows you to see:

- WiFi connection
- MQTT connection
- bus frame detection
- injected commands
- debug messages

## Editing the firmware

Before uploading the code, edit these fields in the firmware.

```
static const char* WIFI_SSID = "YOUR_WIFI_NAME";
static const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";

static const char* MQTT_HOST = "YOUR_HOME_ASSISTANT_IP";
static const uint16_t MQTT_PORT = 1883;

static const char* MQTT_USER = "YOUR_MQTT_USERNAME";
static const char* MQTT_PASSWORD = "YOUR_MQTT_PASSWORD";
```

Replace them with your real values.

## Uploading firmware

Steps:

1. Connect ESP32 to your computer
2. Open Arduino IDE
3. Install required libraries
4. Paste the firmware code
5. Fill in WiFi and MQTT settings
6. Select ESP32-C3 board
7. Select correct COM port
8. Upload firmware
9. Open Serial Monitor at 115200 baud

If everything works you should see:

WiFi connecting
MQTT connecting
MQTT connected
Bus frames detected

## MQTT topics used

Published topics:

enervent/fan_level/state
enervent/hr/state
enervent/availability
enervent/debug

Command topics:

enervent/fan_next/set
enervent/hr_toggle/set

## Home Assistant configuration

Add the following to your `configuration.yaml`.

```
mqtt:
sensor:
- name: "Enervent Fan Level"
unique_id: enervent_fan_level
state_topic: "enervent/fan_level/state"

binary_sensor:
- name: "Enervent HR State"
unique_id: enervent_hr_state
state_topic: "enervent/hr/state"
payload_on: "ON"
payload_off: "OFF"

button:
- name: "Enervent Fan Next"
unique_id: enervent_fan_next
command_topic: "enervent/fan_next/set"
payload_press: "1"

- name: "Enervent HR Toggle"
unique_id: enervent_hr_toggle
command_topic: "enervent/hr_toggle/set"
payload_press: "1"
```

After editing configuration.yaml:

1. Save the file
2. Restart Home Assistant

New entities should appear.

## What the controls do

Fan Next button

This sends the same event as pressing the fan button on the wall panel.

Each press moves the fan to the next level.

HR Toggle button

This toggles heat recovery on or off.

## First test procedure

1. Power the ESP32
2. Confirm WiFi connection
3. Confirm MQTT connection
4. Check `enervent/availability` becomes `online`
5. Verify fan level appears in Home Assistant
6. Press the real wall panel fan button and confirm state updates
7. Press the Home Assistant Fan Next button
8. Confirm the fan level changes

## Troubleshooting

If nothing appears in Home Assistant

Check:

Mosquitto broker is running
MQTT credentials are correct
Home Assistant was restarted

If MQTT connects but commands fail

Check:

RS485 wiring
A and B wires
UART pins
Bus speed

If bus data looks like garbage

Swap A and B wires.

## Example automations

Example uses:

- Set fan level to 4 every Tuesday and Friday at 20:00
- Reduce fan level to 2 Wednesday and Saturday at 10:00
- Toggle HR depending on conditions
- Add dashboard buttons for ventilation control

## Current status

Working features:

- Fan state reading
- Fan control
- HR state reading
- HR toggle
- MQTT bridge
- Home Assistant automation support

## Need help

If you need help setting this up you can:

Use ChatGPT and send it the GitHub repository link.

or email:

softaim67@gmail.com

I will try to help when possible.
