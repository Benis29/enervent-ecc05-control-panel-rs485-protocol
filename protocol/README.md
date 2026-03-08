# ECC05 RS485 Protocol Documentation

This document describes the current understanding of the RS485 communication protocol used between the Enervent ventilation unit main board and the ECC05 control panel.

The information here is based on logic analyzer captures of the RS485 bus and inspection of the ECC05 control panel hardware. The protocol is still under investigation, and some behaviours described here are hypotheses derived from observed traffic patterns.

All captures referenced in this document can be found in the `captures` directory.

## Physical Layer

The ECC05 panel communicates with the main board using a half-duplex RS485 bus.

During capture the RS485 A/B lines were connected to an **isolated XYS485 module**, and the TTL interface of that module was connected directly to the logic analyzer.

Observed configuration:

Bus type: RS485
Mode: half duplex
UART format: 8 data bits, no parity, 1 stop bit
Baud rate: 38400

The panel receives power through the same cable used for the communication bus.

Multiple panels can be connected in a daisy chain configuration.

## Hardware

The ECC05 control panel contains the following main components:

Microcontroller: Freescale MC908JL8
RS485 transceiver: SP485EEN
Oscillator: 8 MHz crystal

The MCU communicates with the RS485 transceiver through a standard UART interface.

## General Communication Model

The ventilation unit main board acts as the **bus master** and periodically broadcasts system state information.

The control panel acts as a **bus participant** that acknowledges state messages and reports user input events.

Communication consists of several frame types exchanged between the main board and the panel.

The most common message families observed are:

TQF
TRC
TRB

## Frame Structure

Most frames follow a similar structure:

Header
Address bytes
Counter byte
Frame type byte
Payload bytes
CRC

Example frame:

```id="qv0qog"
54 51 46 02 0F 52 8F 0A 02 00 01 C0
```

Observed components:

Header
ASCII identifier such as `TQF`, `TRB`, or `TRC`.

Address bytes
Three bytes that appear to represent source and destination identifiers.

Counter
A sequence value used to track message order.

Frame type
Indicates the subtype of the frame.

Payload
Optional data bytes depending on frame type.

CRC
Two byte checksum used for error detection.

## Frame Families

### TQF Frames

TQF frames are sent by the main board and appear to carry system state or command information.

Two commonly observed subtypes:

0A – state broadcast
C8 – event related frame

### TRC Frames

TRC frames are responses sent by the panel that acknowledge state broadcasts.

These frames appear immediately after the panel receives a state frame.

### TRB Frames

TRB frames are sent by the panel during certain events.

They appear in two situations:

User button press events
Panel startup synchronisation

## Normal Operation

When the system is idle the main board periodically broadcasts a state frame.

Observed interval is approximately **10 seconds**.

Typical sequence:

Main board sends TQF state frame
Panel replies with TRC acknowledgement

This exchange keeps the panel synchronised with the current system state.

## Message Flow

The following simplified diagram shows the typical communication flow between the main board and the panel.

### Periodic state update

```id="0l3g0k"
Main board            Control panel

TQF (state)     --->

                  <--- TRC (acknowledgement)
```

### Button press event

```id="t3h0kl"
Main board            Control panel

TQF (event frame) --->

                  <--- TRB (panel response)

TQF (new state)  --->

                  <--- TRC (acknowledgement)
```

### Startup synchronisation

```id="ympndq"
Main board            Control panel

TQF (state)     --->
TQF (state)     --->
TQF (state)     --->

                  <--- TRB (panel detected)

Normal operation begins
```

These flows were reconstructed from bus captures and represent the currently understood behaviour of the system.

## Button Events

When a button on the panel is pressed the following sequence has been observed.

1. A TQF frame appears containing an event indicator
2. The panel sends a TRB response
3. The main board broadcasts a new state frame reflecting the updated setting
4. The panel acknowledges the new state with TRC

If the button press does not result in a valid change (for example heat recovery when the temperature is too low) the event frames still appear but the system state does not change.

## Startup Behaviour

During system startup the main board repeatedly transmits the same state frame.

This continues until a panel sends a TRB frame.

Once the TRB response is received the main board stops repeating the startup frames and normal operation begins.

This behaviour suggests the TRB frame acts as a **panel presence confirmation**.

## Bus Timing Behaviour

Analysis of the logic analyzer captures reveals several consistent timing patterns on the RS485 bus.

Periodic state broadcast interval
Approximately **10 seconds**

Panel acknowledgement delay
TRC frames typically appear **within a few milliseconds** after the corresponding state broadcast.

Button press event timing

Observed sequence:

Button pressed
TQF event frame appears on bus
~1 ms later panel transmits TRB
~20 ms later main board sends updated state frame
~1 ms later panel acknowledges with TRC

These timings indicate that the main board rapidly reacts to panel input once the TRB response is received.

## Addressing

Frames include three address bytes that appear to identify the sender and receiver.

Two common patterns observed:

Main board to panel
0F 02 52

Panel to main board
02 0F 52

The meaning of the third address byte is not yet fully understood.

## Counters

Captured frames show counter values that increment over time.

Evidence suggests that the protocol may use separate counters for different frame types or transactions.

Further analysis is required to confirm how these counters are managed.

## Payload Fields

State frames contain several payload bytes that likely represent the system state.

From observation these appear to correspond to panel functions:

Fan mode
Heat control
Heat recovery

Panels with heating capability (ECC05E) include a third control button located between the fan and heat recovery buttons.

## Hypothesis of Panel Control Encoding

Based on observed traffic patterns the payload may represent the three panel controls using separate bytes.

Possible interpretation:

Fan control
Heat control
Heat recovery

Example payload patterns:

```id="caybik"
01 00 00   fan change
00 01 00   heating change
00 00 01   heat recovery change
```

The ECC05 panel only exposes fan and heat recovery controls, while ECC05E panels add a heating control.

Capturing traffic from an ECC05E panel would help confirm this structure.

## CRC

Frames include a two byte checksum at the end which is used for error detection.

Current analysis of captured frames strongly suggests that the protocol uses a **CRC-16 algorithm compatible with the CRC-16/XMODEM variant**.

Observed properties so far:

CRC size: 16 bits
Polynomial: 0x1021 (CRC-16/XMODEM)
Initial value: likely 0x0000
No reflected input or output

In the CRC-16/XMODEM algorithm the polynomial 0x1021 is used and the calculation is performed over the message bytes with an initial value of 0x0000. The resulting 16-bit value is appended to the end of the frame.

Testing of captured frames against this CRC variant has produced promising results, although the exact byte range used for the calculation still requires confirmation. It is not yet fully clear whether the CRC includes the header bytes or only the payload portion of the frame.

Further captures and automated CRC testing are planned in order to fully confirm the exact implementation used by the ECC05 protocol.

## Current Status

The following aspects of the protocol are reasonably well understood:

Physical layer configuration
Frame headers and basic structure
Periodic state broadcast behaviour
Panel acknowledgement frames
Button event sequences
Startup synchronisation behaviour

The following areas require further investigation:

Exact CRC implementation
Full meaning of payload bytes
Address byte roles
Counter behaviour
Extended features used by ECC05E panels

## Future Work

Capture traffic from ECC05E panels to analyse heating control behaviour.

Fully reverse engineer the CRC calculation.

Develop a microcontroller based panel emulator capable of communicating with the ventilation unit.

Such an emulator could allow integration of the ventilation unit with home automation systems and remote control interfaces.

