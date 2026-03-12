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

The following parts of the protocol are already understood:

• periodic state broadcast frames  
• acknowledgement frames from the panel  
• button press event frames  
• basic address structure  
• frame counters  
• CRC calculation

Some parts still need more investigation, especially the extended features used by ECC05E panels that include heating control.

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

## Goal of the project

The long term goal is to create a small controller (for example using an ESP32) that can act as a virtual control panel and allow the ventilation unit to be controlled through software or home automation systems.

## Disclaimer

This project is not affiliated with Enervent.

All information is based on reverse engineering of hardware and bus captures.  
Use the information here at your own risk. Working with HVAC systems and electronics can damage equipment if done incorrectly.

This repository exists for educational and hobbyist purposes.
