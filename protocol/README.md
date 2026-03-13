# Enervent ECC05 Protocol Documentation

This document describes the current understanding of the RS485 communication protocol used between the Enervent main controller and ECC05 control panels.

The protocol was reverse engineered by observing the live bus with a logic analyzer, testing multiple original ECC05 panels, and later injecting valid frames back onto the bus using an ESP32-C3 and an isolated RS485 interface.

The goal of this document is to explain the protocol clearly enough that another person can understand how the system works, reproduce the results, and build their own bridge or controller.

## Overview

The ECC05 system uses a half-duplex RS485 bus between the main ventilation controller and one or more wall panels.

The bus carries two main categories of traffic:

1. **State traffic**
The main controller broadcasts the current system state to the wall panels.

2. **Event traffic**
A wall panel sends a control event when the user presses a button, and the main controller processes that event and then broadcasts the resulting updated state.

This is an important distinction.

The wall panel does **not** directly decide the final system state.
Instead, the panel requests a change, and the main controller decides the new accepted state.

That is why the protocol naturally splits into:

- event frames
- state frames

## Physical Layer

The bus uses:

- RS485
- half duplex
- 38400 baud
- 8 data bits
- no parity
- 1 stop bit

In short:

```text
38400 8N1
```

This has been confirmed repeatedly from live captures.

## Hardware Used for Reverse Engineering

The protocol was analyzed using:

- original Enervent ECC05 wall panels
- Enervent main controller board
- ESP32-C3 SuperMini
- isolated XY-S485 / XYS485 RS485-to-TTL interface
- logic analyzer
- Saleae Logic / compatible software

The isolated RS485 interface is strongly recommended when connecting custom electronics to the bus.

## Protocol Families

Several frame families have been observed on the bus.

The important confirmed ones are:

### TQF / 0A
Main system state broadcast

### TQF / C8
Control event frame

### TRB
Event acknowledgement or event-side response

### TRC
State acknowledgement

In addition, rare special frames have also been observed:

### TQC / 0F
Rare special frame

### TRD / 0F
Rare special frame

These rare frames are real and CRC-valid, but their exact meaning is still not fully confirmed.

## Frame Structure

Most observed frames follow a common layout:

```text
[Header 3 bytes] [Address bytes] [Counter] [Type] [Payload] [CRC]
```

In practice this looks like:

```text
54 51 46 02 0F 52 F7 0A 02 00 01 19 4B
```

Breaking that down:

- `54 51 46` = frame header (`TQF`)
- `02 0F 52` = address / role bytes
- `F7` = counter
- `0A` = frame subtype
- `02 00 01` = payload
- `19 4B` = CRC

Not every frame has the same total length, but the same general pattern appears throughout the protocol.

## ASCII-like Frame Headers

The first 3 bytes of many frames are ASCII-like and make the traffic easier to recognize.

Examples:

- `54 51 46` = `TQF`
- `54 52 43` = `TRC`
- `54 52 42` = `TRB`
- `54 51 43` = `TQC`
- `54 52 44` = `TRD`

This was one of the useful clues during reverse engineering because it immediately showed that the system was using structured message families, not random binary blobs.

## CRC

All major confirmed frame types use **CRC-16/XMODEM**.

Current confirmed properties:

- polynomial: `0x1021`
- initial value: `0x0000`
- big-endian CRC bytes
- CRC is calculated over:

```text
frame[1:-2]
```

This means:

- skip the first byte of the frame
- calculate CRC over everything up to but excluding the final 2 CRC bytes

This CRC model was confirmed by:

- matching captured live traffic
- matching injected frames
- successful state spoofing
- successful event injection

At this point the CRC should be considered confirmed.

## Counter System

The protocol uses counters, and they matter.

This was one of the most important findings because valid CRC alone is **not enough**.
Frames also need the correct next counter to be accepted.

There are effectively **two counter domains**.

### State counter domain

Used by:

- `TQF / 0A`
- `TRC`

Observed behavior:

- main controller sends `TQF / 0A` with counter `N`
- panel replies with `TRC` using the same counter `N`

Example:

```text
Main -> TQF/0A counter F6
Panel -> TRC counter F6
```

A forged `TQF / 0A` state frame was only accepted by the panel when the counter was advanced to the next expected value.

This experimentally confirmed that the state counter is actively checked.

### Event counter domain

Used by:

- `TQF / C8`
- `TRB`

Observed behavior:

- a real button press causes `C8` with some event counter
- the matching `TRB` uses that same event counter

This strongly suggests that `C8` and `TRB` belong to the same event transaction.

The event counter is independent from the state counter.

## Main State Frame: TQF / 0A

This is the most important state frame in the protocol.

It is broadcast by the main controller and tells the wall panels what the accepted current system state is.

Example:

```text
54 51 46 02 0F 52 F7 0A 02 00 01 19 4B
```

The payload bytes currently decode as:

```text
0A [fan] [after-heating] [HR]
```

### Byte 8: fan level

Confirmed values:

- `01` = fan 1
- `02` = fan 2
- `03` = fan 3
- `04` = fan 4

This has been experimentally confirmed both by passive captures and by successful state spoofing.

### Byte 9: after-heating level

Strong current interpretation:

- this byte represents the after-heating control level
- likely range `0–4`

This matches the physical control layout of ECC05E-family systems and the likely protocol design.

### Byte 10: heat recovery state

Confirmed values:

- `00` = HR off
- `01` = HR on

This was confirmed during live testing where HR could finally be toggled under warm enough conditions.

### Example payloads

```text
02 00 00 = fan 2, after-heating 0, HR off
02 00 01 = fan 2, after-heating 0, HR on
```

## State Acknowledgement: TRC

`TRC` is the panel acknowledgement for state broadcasts.

Observed behavior:

```text
Main -> TQF / 0A
Panel -> TRC
```

The `TRC` counter matches the `TQF / 0A` counter.

This confirms that `TRC` belongs to the state transaction domain.

## Event Frame: TQF / C8

This is the key event frame used when buttons are pressed.

This was one of the major breakthroughs in the project.

Originally it was unclear whether the main controller or the panel originated this frame.

That uncertainty was resolved by powering a spare panel alone and observing that the panel itself could generate valid `C8` frames without the main controller being present.

That means `C8` is panel-originated.

This was a major architectural insight because it showed that:

- the panel initiates control events
- the main controller remains the authority for final accepted state

### Confirmed event payloads

The event payload uses the 3 payload bytes in the same logical order as the 3 control functions:

```text
[fan event] [after-heating event] [HR event]
```

Confirmed or strongly supported values:

- `01 00 00` = fan button event
- `00 00 01` = HR button event
- `00 01 00` = strongly likely after-heating event

### Fan event

Confirmed working payload:

```text
C8 01 00 00
```

This event has been successfully injected with an ESP32 and causes the main controller to step the fan mode exactly like a real wall panel button press.

### HR event

Confirmed from captures:

```text
C8 00 00 01
```

This was observed during successful HR toggling once conditions allowed HR to switch.

### After-heating event

Strongly likely payload:

```text
C8 00 01 00
```

This is not yet runtime-confirmed in the same way as fan and HR, but it fits the payload layout and the known panel families very well.

## Meaning of 01 and 02 in Event Payloads

One of the more subtle findings came from comparing slow clean button presses with rapid repeated button presses.

Observed behavior:

- slow single fan presses used `01 00 00`
- rapid or repeated event sequences sometimes used `02 00 00`

This suggests that the event payload value is not just a simple one-hot button ID.

A strong working interpretation is:

- `01` = normal event start
- `02` = repeated / continued / retried event within the same active event flow

This theory became much stronger after testing a standalone panel.

### Standalone panel behavior

When a spare panel was powered alone:

- it sent valid `C8 01 00 00` frames
- it repeatedly retried the same counter
- then advanced to `C8 02 00 00`
- it still did not receive the normal event response sequence because the controller was absent

This strongly suggests that:

- `01` is the start of a new event request
- `02` is a repeated / continued retry state when the event is not being completed normally

## TRB

`TRB` is an event-side acknowledgement or response frame.

The exact semantic meaning is still slightly less obvious than `C8`, `0A`, and `TRC`, but the overall role is now much clearer.

Important observations:

- `TRB` appears during normal button-event transactions
- `TRB` shares the event counter with `C8`
- `TRB` did **not** appear when a panel was powered alone
- `TRB` appears after `C8`, not immediately at the instant of the physical button press

This strongly supports the idea that `TRB` is main-controller-originated, or at least controller-dependent.

The current best interpretation is:

- `C8` = panel event request
- `TRB` = controller-side acknowledgement or continuation of event processing
- `0A` = authoritative accepted final state
- `TRC` = panel state acknowledgement

## Event Transaction Flow

The confirmed event flow now looks like this:

```text
Panel / external device -> TQF / C8
Main controller -> TRB
Main controller -> TQF / 0A
Panel -> TRC
```

This is the core transaction model of the protocol.

### Why this matters

It means the wall panel is basically an event source, not the actual authority of system state.

That is why an external device like an ESP32 can act as a virtual panel simply by generating valid `C8` event frames.

## State Spoofing Breakthrough

Before event injection was fully working, a state spoofing test was performed.

A forged `TQF / 0A` was injected onto the bus using:

- valid CRC
- valid payload
- next expected state counter

Result:

- the panel accepted the frame
- the panel changed displayed fan state
- the panel sent a valid `TRC`
- the next real controller state frame later restored the true state

This was a very important result because it confirmed:

- the `0A` frame layout
- fan byte position
- CRC model
- state counter behavior
- panel acceptance logic

It also showed that the panel trusts believable next-state broadcasts.

## Event Injection Breakthrough

The major breakthrough of the whole project came when a valid `C8 01 00 00` frame was injected using the correct next event counter.

Result:

- the controller accepted it
- fan mode stepped exactly once
- the controller broadcast the new `0A` state
- the panels updated normally

Repeated tests confirmed that this could be done reliably.

This proved that:

- an ESP32 can act as a virtual ECC05 keypad
- the event injection model is correct
- fan mode can be controlled externally

At this point fan control should be considered confirmed.

## Multi-Panel Behavior

The protocol was also observed with multiple real ECC05 panels on the same bus.

Interesting result:

- with two panels connected, idle traffic did **not** show obvious duplicate acknowledgements
- the bus still looked like a normal `0A -> TRC` pattern

This suggests that:

- not every panel necessarily replies independently
- or there is some simple arbitration / passive behavior
- or only one panel acts as the active responder in many situations

However, button events from different panels were observed to behave equivalently, which means the bus is not encoding a simple visible “which wall panel pressed the button” distinction in the normal captured event payloads.

## Rare Special Frames

Several rare special frames have been observed, especially near startup or certain transitions.

Examples:

- `TQC / 0F`
- `TRD / 0F`

These frames are CRC-valid and real.

They are not random decode errors.

Current best interpretation:

- startup / rejoin / diagnostic / special maintenance traffic

Their exact function is still unknown, and they are not required for the currently working fan and HR integration.

## Standalone Panel Test

A spare panel was powered on the bench without the main controller.

This test was very important because it revealed:

- the panel can generate valid `C8` event frames on its own
- the panel does not complete the normal controller response chain alone
- `TRB` does not appear in the same way without the main controller
- repeated `C8` retries happen when the event cannot complete normally

This strongly supports the event-request model described above.

## Home Assistant / MQTT Bridge

A working bridge was built using:

- ESP32-C3 SuperMini
- isolated XYS485 module
- MQTT
- Home Assistant

Current confirmed working integration features:

- fan state publish
- fan next command
- HR state publish
- HR toggle command
- command debounce to prevent bus flooding
- automations in Home Assistant for scheduled fan changes

This proves that the protocol is not only understood passively, but can be actively used in a real smart-home integration.

## Current Payload Interpretation Summary

### TQF / 0A state payload

```text
[fan level] [after-heating level] [HR state]
```

Current interpretation:

- byte 8 = fan level `1–4`
- byte 9 = after-heating level `0–4`
- byte 10 = HR state `0/1`

### TQF / C8 event payload

```text
[fan event] [after-heating event] [HR event]
```

Current interpretation:

- `01 00 00` = fan event
- `00 01 00` = after-heating event
- `00 00 01` = HR event

Likely event mode values:

- `01` = normal new event
- `02` = repeated / continued / retried event

## What Is Confirmed

The following parts of the protocol are strongly confirmed by both captures and live injection tests:

- bus speed `38400 8N1`
- CRC-16/XMODEM
- `TQF / 0A` as state broadcast
- `TRC` as state acknowledgement
- `TQF / C8` as event request
- `TRB` as event-side acknowledgement / response
- fan level in `0A` byte 8
- HR state in `0A` byte 10
- HR event as `C8 00 00 01`
- fan event as `C8 01 00 00`
- successful fan control by event injection
- successful panel state spoofing using forged `0A`

## What Is Strongly Likely But Not Fully Final

The following interpretations are strong but still not as experimentally complete as fan and HR:

- `0A` byte 9 = after-heating level
- `C8 00 01 00` = after-heating event
- `01` and `02` as event start / event continue semantics
- exact role of some rare startup/special frames

## Practical Takeaway

At this point the protocol is understood well enough to:

- read fan level
- control fan level
- read HR state
- toggle HR
- build an MQTT / Home Assistant bridge
- emulate a control panel for the main user-facing functions

That is already enough for a very useful modern integration of an old Enervent ventilation system.

## Future Work

Possible future areas of investigation:

- fully confirm after-heating write control
- decode the rare `TQC / TRD` special frames
- determine if additional service or alarm states are available on the panel bus
- add MQTT discovery and more polished Home Assistant entities
- produce a more formal byte-level frame reference table

## Final Note

This protocol was not documented publicly and had to be worked out from captures, experiments, and repeated live validation.

The most important architectural insight is simple:

**The wall panel generates control events.
The main controller decides and broadcasts the final accepted state.**

Once that became clear, the rest of the protocol started to make sense.
