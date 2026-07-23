# Pureplay Connect - launch monitor wire protocol

Version 1

Pureplay Connect is an open, compact binary protocol a launch monitor can use to send shot
data to Pureplay over TCP on port 9760.

---

## Overview

- Roles. Two peers: the device (the launch monitor) and the host (Pureplay).
- Transport. The device connects to the host over TCP: the host listens and the
  device connects to it. For other transport layers (BTLE, USB etc), contact the Pureplay dev
  team directly.
- Two data streams, paired downstream. Ball flight and club delivery are separate
  messages correlated by a shared ShotId. A ball-only monitor simply never sends
  club messages.
- Bidirectional. The device streams telemetry and state; the host sends commands
  (club selection, arm/disarm, ball type).
- Optional fields are first class. Every telemetry message carries a validity
  bitmask so a device sends only what it actually measured.

---

## 1. Conventions

- Endianness: all multi byte values are little endian.
- Field types used below:
  - integer - unsigned; its width in bytes is given by each field's Size.
  - float - a 4 byte floating point value. A field whose validity bit is clear
    should be sent as 0.0; the receiver ignores it either way.
  - bitfield - an integer whose individual bits are flags.
  - byte - a single unsigned byte.
- Units:
  - speed - m/s
  - angles - degrees
  - spin - RPM
  - face impact point - millimetres from the sweet spot
- Sign conventions:

  | Quantity | Positive means |
  |---|---|
  | HLA, FaceAngle, ClubPath | right of the target line |
  | VLA, AttackAngle | upward |
  | SpinAxis | tilted right (fade/slice); 0 = pure backspin, +/-90 = pure sidespin |
  | ImpactPointX | toward the toe (heel is negative) |
  | ImpactPointY | above face centre (below is negative) |

- Strings: a single length byte (0 to 255) followed by that many UTF-8 bytes.
  No null terminator.

---

## 2. Frame format

Every message - telemetry, command, or state - is one frame:

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0 | 2 | Magic | 0x50 0x4C ("P","L") |
| 2 | 1 | Version | protocol major version - 1 |
| 3 | 1 | Reserved | must be 0 |
| 4 | 1 | Type | message type (section 3) |
| 5 | 2 | Length | integer, byte count of Payload (0 to 65535) |
| 7 | N | Payload | type specific (sections 4, 5) |
| 7+N | 4 | CRC32C | Castagnoli CRC over bytes 0 to 7+N-1 (header and payload) |

Fixed 7 byte header plus 4 byte trailer = 11 bytes of overhead. The frame is self
delimiting (length prefixed), so a reader can split frames out of the TCP byte
stream.

The CRC32C detects truncation and corruption. When a receiver sees a bad magic or
a CRC mismatch it should drop one byte and resynchronise (scan forward until the
magic and CRC line up on a real frame boundary).

---

## 3. Message types

Device to host:

| Type | Name | Meaning |
|---|---|---|
| 0x01 | Hello | device identity and capabilities |
| 0x02 | Heartbeat | liveness; connection state, ready state, battery snapshot |
| 0x10 | BallData | ball flight reading |
| 0x11 | ClubData | club reading (optional per shot) |
| 0x12 | ReadyState | ready state transition |
| 0x1F | Error | device side error / nack |

Host to device:

| Type | Name | Meaning |
|---|---|---|
| 0x81 | HelloAck | host acknowledgement of the handshake |
| 0x82 | Ping | keepalive request (device replies with Heartbeat) |
| 0x90 | SetClubInfo | selected club and handedness |
| 0x91 | SetArm | arm / disarm ball detection |
| 0x92 | SetBallType | marker / markerless |

An unknown Type should be answered with an Error and otherwise ignored (forward
compatibility).

---

## 4. Telemetry payloads

Both telemetry messages use a fixed size payload: every field is always present
on the wire, and the ValidMask states which fields the device actually measured.
This keeps parsing branch free and makes forward compatible field appends trivial
(section 6). Every field after ValidMask is a float (4 bytes).

### 4.1 BallData (0x10) - payload 34 bytes

| Offset | Size | Field | |
|---|---|---|---|
| 0 | 4 | ShotId | integer, per session, increasing from 1 |
| 4 | 2 | ValidMask | bitfield, bits below |
| 6 | 4 | BallSpeedMS | |
| 10 | 4 | VLA | vertical launch angle |
| 14 | 4 | HLA | horizontal launch angle |
| 18 | 4 | TotalSpinRPM | |
| 22 | 4 | BackSpinRPM | |
| 26 | 4 | SideSpinRPM | |
| 30 | 4 | SpinAxisDeg | |

ValidMask bits: 0 = BallSpeed, 1 = VLA, 2 = HLA, 3 = TotalSpin, 4 = BackSpin,
5 = SideSpin, 6 = SpinAxis.

Spin gap fill (host side). A device sends whatever spin components it measured and
clears the rest. The host derives the missing ones:

- if TotalSpin absent but back and side present: TotalSpin = sqrt(Back^2 + Side^2)
- if SpinAxis absent but back and side present: SpinAxis = atan2(Side, Back) in degrees

This is why there is no "supports spin axis" capability - the mask plus this math
covers every reporting style.

### 4.2 ClubData (0x11) - payload 38 bytes

Send only if the monitor measured club delivery. Omitting it entirely is valid.

| Offset | Size | Field | |
|---|---|---|---|
| 0 | 4 | ShotId | integer - must equal the paired BallData ShotId |
| 4 | 2 | ValidMask | bitfield, bits below |
| 6 | 4 | ClubHeadSpeedMS | |
| 10 | 4 | SmashFactor | omit (clear bit) to let the host derive it |
| 14 | 4 | DynamicLoftDeg | |
| 18 | 4 | FaceAngleDeg | |
| 22 | 4 | ClubPathDeg | |
| 26 | 4 | AttackAngleDeg | |
| 30 | 4 | ImpactPointXMM | mm from sweet spot, +toe / -heel |
| 34 | 4 | ImpactPointYMM | mm from sweet spot, +high / -low |

ValidMask bits: 0 = ClubHeadSpeed, 1 = SmashFactor, 2 = DynamicLoft,
3 = FaceAngle, 4 = ClubPath, 5 = AttackAngle, 6 = ImpactPointX, 7 = ImpactPointY.

### 4.3 Pairing rule

Ball and club for one swing carry the same ShotId. The device emits BallData
first, then ClubData if it has it. The host pairs by ShotId, and if SmashFactor
was not sent it derives SmashFactor = BallSpeedMS / ClubHeadSpeedMS. A ClubData
with no matching BallData is held briefly, then dropped.

---

## 5. State, command and handshake payloads

### 5.1 Hello (0x01)

| Size | Field | |
|---|---|---|
| 1 | ProtocolVersion | highest wire version the device supports |
| 2 | DeviceCaps | bitfield (below) |
| 1+n | DeviceName | string (see section 1) |
| 1+n | Firmware | string |

DeviceCaps bits: bit 4 = BatteryPowered. All other bits are reserved and must be
0. A device that is not battery powered sends DeviceCaps = 0.

### 5.2 HelloAck (0x81)

| Size | Field | |
|---|---|---|
| 1 | ProtocolVersion | the lowest common version the host selected |
| 2 | HostCaps | integer, reserved (0) |
| 1 | Reserved | byte, reserved (0) |

### 5.3 Heartbeat (0x02)

| Size | Field | |
|---|---|---|
| 1 | ConnectionState | see section 9 |
| 1 | ReadyState | see section 9 |
| 1 | Battery | 0 to 100, or 0xFF = unknown |

Liveness is host driven and adaptive: the host sends a Ping (0x82) only after the
device has been silent for about 4 seconds, and the device must reply with a
Heartbeat. Any frame the device sends - telemetry, ready state, or an unsolicited
Heartbeat - resets this timer, so a device that is actively streaming is never
pinged. The host treats more than 8 seconds without any frame from the device as
a lost link and drops the connection. The only hard requirement on the device is
to answer each Ping. Battery is only surfaced by the host when the device set the
BatteryPowered capability; a device on external power sends 0xFF.

### 5.4 ReadyState (0x12)

| Size | Field |
|---|---|
| 1 | ReadyState (see section 9) |

Push this on every ready state transition (do not wait for the next heartbeat).

### 5.5 Commands (host to device)

| Message | Payload |
|---|---|
| SetClubInfo (0x90) | Handedness (byte), Club (byte) - both in section 9 |
| SetArm (0x91) | Arm (byte): 0 = disarm, 1 = arm |
| SetBallType (0x92) | BallType (byte, section 9) - a single mode device ignores it |
| Ping (0x82) | (empty) - host sends after about 4 s of device silence; the device must reply with a Heartbeat |

### 5.6 Error (0x1F)

| Size | Field |
|---|---|
| 2 | Code (integer) |
| 1+n | Message (string) |

Code 0 is a generic error. Other values are reserved.

Enumerations are wire frozen. Club, Handedness, BallType, ConnectionState and
ReadyState are serialised as single byte values in section 9. Adding new
enumerators is allowed; renumbering existing ones is a major version change.

---

## 6. Versioning and extensibility

- Every frame carries the Version byte; Hello and HelloAck negotiate the lowest
  common version.
- Appending a new optional telemetry field: add it after the existing fields and
  define a new ValidMask bit. Old receivers read the prefix they know and use
  Length to skip the tail - no version bump needed.
- Reordering or resizing existing fields, or renumbering any enum below, is a
  breaking change and requires a major version bump.

---

## 7. Handshake sequence

```
Device                         Host
  |  --- TCP connect -------->  |   (host listens; device connects)
  |  --- Hello ------------->   |
  |  <------------ HelloAck --  |
  |                             |   Connected
  |  --- ReadyState -------->   |
  |  --- BallData(ShotId=n) ->  |
  |  --- ClubData(ShotId=n) ->  |
  |  <--------- SetArm / ... -  |   (host commands, any time)
  |  <------------- Ping -----  |   (host, after ~4 s of silence)
  |  --- Heartbeat -------->    |   (device reply; >8 s silence drops the link)
```

---

## 8. CRC32C reference

Castagnoli CRC-32 (reflected polynomial 0x82F63B78), initial value 0xFFFFFFFF,
final XOR 0xFFFFFFFF, computed over the frame header and payload (the bytes
preceding the 4 byte trailer), stored little endian.

Pseudocode (crc is a 32 bit unsigned value):

```
crc = 0xFFFFFFFF
for each byte in (header + payload):
    crc = crc XOR byte
    repeat 8 times:
        if (crc AND 1) is nonzero:
            crc = (crc >> 1) XOR 0x82F63B78
        else:
            crc = crc >> 1
result = crc XOR 0xFFFFFFFF        // store as 4 little endian bytes
```

---

## 9. Enum values (frozen)

These single byte values are part of the wire contract.

ConnectionState - 0 Disconnected, 1 Connecting, 2 Connected, 3 Reconnecting,
4 Error.

ReadyState - 0 NotReady, 1 Ready, 2 BallDetected, 3 Processing.

Handedness - 0 Right, 1 Left.

BallType - 0 Marker, 1 Markerless.

Club -
0 Driver;
1 to 8 Wood2 to Wood9;
9 to 16 Hybrid2 to Hybrid9;
17 to 25 Iron1 to Iron9;
26 PitchingWedge, 27 GapWedge, 28 SandWedge, 29 LobWedge;
30 Putter.

---

## 10. Worked examples

Byte accurate frames (spaces added for readability). All values little endian.

BallData - ShotId 1, all fields valid (mask 0x007F), BallSpeed 64.0 m/s, VLA
12.5, HLA -1.5, TotalSpin 2500, BackSpin 2450, SideSpin -300, SpinAxis 6.98:

```
50 4C 01 00 10 22 00 01 00 00 00 7F 00 00 00 80 42 00 00 48 41
00 00 C0 BF 00 40 1C 45 00 20 19 45 00 00 96 C3 29 5C DF 40 0A 14 D6 31
```

50 4C magic, 01 version, 00 flags, 10 type, 22 00 length = 34, payload (34
bytes), 0A 14 D6 31 CRC32C.

ClubData - ShotId 1, all valid (mask 0x00FF), ClubHeadSpeed 47.5, Smash 1.347,
DynamicLoft 14.2, FaceAngle -0.8, ClubPath 2.1, AttackAngle -3.4, ImpactX 5.0 mm,
ImpactY -2.0 mm:

```
50 4C 01 00 11 26 00 01 00 00 00 FF 00 00 00 3E 42 7F 6A AC 3F
33 33 63 41 CD CC 4C BF 66 66 06 40 9A 99 59 C0 00 00 A0 40 00 00 00 C0 5D D2 D9 F1
```

Hello - version 1, caps 0x0010 (BatteryPowered), name "ACME LM", firmware "1.0":

```
50 4C 01 00 01 0F 00 01 10 00 07 41 43 4D 45 20 4C 4D 03 31 2E 30 C0 C9 B1 D0
```

HelloAck - version 1, HostCaps 0, Reserved 0:

```
50 4C 01 00 81 04 00 01 00 00 00 64 CA B4 DA
```

Ping - empty payload:

```
50 4C 01 00 82 00 00 D0 A9 7C 7D
```
