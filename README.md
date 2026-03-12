### W.I.P full •homeassistant mqtt• code coming soon


# Enervent ECC05 Control Panel RS485 Protocol

This project documents the RS485 communication protocol used between the Enervent ventilation unit main board and the ECC05 control panel.

The goal of this repository is to reverse engineer how the panel and the main unit communicate so that these older systems can be integrated with modern home automation systems or custom controllers.

This work was done using logic analyzer captures, UART sniffing and inspection of the ECC05 hardware.

## What is documented here

• RS485 frame structure  
• message types used on the bus (TQF, TRB, TRC)  
• startup behaviour and boot handshake  
• state polling and periodic broadcasts  
• CRC behaviour and checksum experiments  
• logic analyzer captures of real traffic  
• hardware photos of the ECC05 PCB

The captures show how the ventilation unit continuously broadcasts state information and how the panel responds to it. Button presses from the panel generate specific frames that trigger a new state broadcast from the main board.

## Hardware examined

Control panel: Enervent ECC05  
Microcontroller: Freescale MC908JL8  
RS485 transceiver: SP485EEN  
Bus: half-duplex RS485

The panel connects to the ventilation unit using a simple RS485 bus which also carries power for the panel.

## Current progress


## Major breakthrough update

After extensive bus captures and testing using a logic analyzer and an ESP32-C3 connected through an isolated RS-485 interface, it has now been confirmed that the ECC05 protocol can be actively controlled by injecting valid frames onto the bus.

The system accepts correctly formed `C8` event frames and processes them exactly like a real keypad button press. This means the control panel itself is not the authority of the system state. Instead, it sends event requests which the main ventilation controller interprets and then applies.

This discovery means an external device can emulate a control panel and operate the ventilation unit directly.

### What we discovered

• The protocol uses **CRC-16 XMODEM** for frame integrity.  
• Frames contain a **counter byte** which increments with events and state updates.  
• The control panel sends **event requests**, not direct state changes.  
• The **main controller** decides the final state and broadcasts it to all panels.  
• External devices can inject valid frames and the controller accepts them normally.

### Confirmed bus sequence

The observed interaction sequence on the RS-485 bus appears to be:

```
Panel / external device → TQF (C8 event frame)
Main controller        → TRB (event acknowledgement)
Main controller        → TQF (0A state broadcast)
Panel                  → TRC (state acknowledgement)
```

This confirms that the keypad buttons simply generate events, while the controller applies the change and informs all panels of the new state.

### Status of protocol research

Current confirmed items:

• RS-485 bus speed: **38400 baud**  
• Frame integrity: **CRC-16 XMODEM**  
• Event request frame: **TQF / C8**  
• Controller acknowledgement: **TRB**  
• State broadcast frame: **TQF / 0A**  
• Panel acknowledgement: **TRC**

### Fan mode command

Fan mode changes are performed through the `C8` event frame.

The payload bytes indicate the requested action:

```
01 00 00  → normal fan mode change (next level)
```

Sending a correctly formed `C8` frame with this payload causes the controller to advance the fan speed. The controller then broadcasts the new system state to all panels.

### Test results

Using an ESP32-C3 connected through an isolated RS-485 module, repeated `C8` injections were performed.

Results:

• Fan mode successfully changed multiple times  
• Panels updated LEDs correctly  
• The controller broadcast the updated state after each event  
• No errors or bus instability observed during repeated injections

This confirms that external hardware can reliably emulate a control panel and control the ventilation system through the RS-485 protocol.

### Confirmed control test

Using an ESP32-C3 injector, valid `C8` frames were transmitted onto the bus.

Result:

• Controller accepted the injected event  
• Fan mode advanced correctly  
• Controller broadcast a new `0A` state frame  
• All panels updated LEDs

Repeated injections successfully cycled fan speed.



The following parts of the protocol are already understood:

• periodic state broadcast frames  
• acknowledgement frames from the panel  
• button press event frames  
• basic address structure  
• frame counters  
• CRC calculation

Some parts still need more investigation, especially the extended features used by ECC05E panels that include heating control.


### Test hardware

The protocol was captured and tested using:

• Saleae compatible 24MHz logic analyzer  
• ESP32-C3 SuperMini  
• XY-S485 isolated RS-485 interface module  
• Logic 2 analyzer software

The ESP32 was connected to the bus using:

RX → GPIO20  
TX → GPIO21  
Baud → 38400 8N1


## Update 1
Important observation:
A spare ECC05 panel powered alone on the bench transmits valid TQF/C8 frames
when a button is pressed, even with no main board connected.

This strongly suggests that TQF/C8 is panel-originated and forms the start of
the event-side protocol exchange.

Observed standalone frames:
54 51 46 0F 03 52 57 C8 01 00 00 05 BD
54 51 46 0F 03 52 58 C8 02 00 00 39 14

Both match the CRC-16/XMODEM model used elsewhere in the protocol.

A strong working hypothesis is that TQF/C8 is the panel-originated event descriptor frame and contains the meaningful event context.
TRB appears to be a generic confirmation or commit frame within the event transaction, rather than the frame that actually carries the button identity.
This explains why TRB has almost no visible payload, why C8 can be observed from a standalone panel, and why TRB does not appear when the panel is powered without the main board.
## Goal of the project

The long term goal is to create a small controller (for example using an ESP32) that can act as a virtual control panel and allow the ventilation unit to be controlled through software or home automation systems.

## Disclaimer

This project is not affiliated with Enervent.

All information is based on reverse engineering of hardware and bus captures.  
Use the information here at your own risk. Working with HVAC systems and electronics can damage equipment if done incorrectly.

This repository exists for educational and hobbyist purposes.
