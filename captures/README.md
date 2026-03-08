# Bus Captures

These captures show the RS485 communication between the Enervent ventilation unit main board and the ECC05 control panel.

For these recordings the RS485 bus was connected to an **isolated XY-S485 module**.
The **TTL side of the module was connected directly to the logic analyzer**, which decoded the traffic as UART.

The captures are exported as CSV files so they can be inspected in analysis tools or processed with scripts.

## Capture types

idle_bus
Traffic recorded while the system is running normally with no buttons pressed.
These captures show the periodic state broadcast frames sent by the main board and the acknowledgements from the panel.

startup
Captures recorded during system startup or when the control panel is plugged into the bus.
During startup the main board repeatedly sends state frames until the panel responds and normal operation begins.

fan_tests
Captures taken while cycling the fan mode button on the control panel.
These logs show the button event frames and the updated state broadcasts that follow.

hr_tests
Captures recorded while pressing the heat recovery button.
On the tested unit the command is ignored when the temperature is too low, but the event frames still appear on the bus.

## Notes

All captures in this repository were recorded using a logic analyzer connected through the isolated XYS485 module.
ESP32 + MAX485 sniffing logs are **not included** here since those recordings used a different setup and configuration.

These files are provided so others can inspect the raw traffic and help verify the behaviour of the protocol.

