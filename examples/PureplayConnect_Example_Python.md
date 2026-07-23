# Pureplay Connect — minimal device in Python

A complete, minimal **device-side** reference for the Pureplay Connect
protocol. The *device* (your launch monitor) **connects to** the game, 
which listens on TCP (default port **9760**, host-as-server).
It implements the core protocol: connect, handshake, telemetry, and keepalive.

**Transport note:** This example is **TCP only**, but the principles 
(framing, CRC, handshake, keepalive) are identical for every transport;
only the connect/read/write plumbing changes. If you're connecting over a 
different transport, contact the Pureplay dev team for a more specific example.

What this example does, in order:

1. Opens a TCP connection to the host.
2. Sends a `Hello` announcing its name/firmware.
3. Waits for `HelloAck` (handshake done), then sends one demo shot
   (`BallData` + `ClubData`).
4. Replies to every `Ping` with a `Heartbeat` — the one hard requirement that
   keeps the link alive.
5. Logs the host's commands (`SetArm` / `SetClubInfo` / `SetBallType`).

### Run

```sh
python3 pureplay_connect_device.py
```

Standard library only (`socket`, `struct`) — no dependencies.

### `pureplay_connect_device.py`

```python
# Minimal Pureplay Connect device.
import socket
import struct

# ---- Protocol constants ----
# MAGIC0/MAGIC1 spell "PL" (Pureplay) — the two bytes every frame starts with, so
# a reader can find/resync to frame boundaries and reject non-PPConnect data.
MAGIC0, MAGIC1, WIRE_VERSION = 0x50, 0x4C, 1
HELLO, HEARTBEAT, BALL, CLUB, READY, ERROR, HELLOACK, PING, SETCLUB, SETARM, SETBALL = (
    0x01, 0x02, 0x10, 0x11, 0x12, 0x1F, 0x81, 0x82, 0x90, 0x91, 0x92
)


# ---- CRC32C: catches corrupted/truncated frames so they're dropped, not
# mis-parsed. Castagnoli variant; must match the host byte-for-byte. ----
def crc32c(data: bytes) -> int:
    crc = 0xFFFFFFFF
    for byte in data:
        crc ^= byte
        for _ in range(8):
            crc = (crc >> 1) ^ 0x82F63B78 if (crc & 1) else (crc >> 1)
    return crc ^ 0xFFFFFFFF


# ---- Wrap a payload in a frame: magic|ver|flags|type|len|payload|crc32c ----
# struct format chars: '<' little-endian, 'B' uint8, 'H' uint16, 'I' uint32, 'f' float32.
def frame(msg_type: int, payload: bytes) -> bytes:
    head = bytes([MAGIC0, MAGIC1, WIRE_VERSION, 0, msg_type])  # flags = 0
    head += struct.pack("<H", len(payload))                    # little-endian length
    body = head + payload
    return body + struct.pack("<I", crc32c(body))              # CRC over header + payload


def pstr(s: str) -> bytes:
    """A length-prefixed UTF-8 string: uint8 length + bytes."""
    raw = s.encode("utf-8")
    return bytes([len(raw)]) + raw


# ---- Message builders ----
def build_hello() -> bytes:
    payload = bytes([WIRE_VERSION])      # ProtocolVersion
    payload += struct.pack("<H", 0)      # DeviceCaps = 0 (no optional capabilities advertised)
    payload += pstr("LMDeviceName")      # DeviceName
    payload += pstr("1.0.0")             # Firmware
    return frame(HELLO, payload)


def build_heartbeat() -> bytes:
    # ConnectionState=Connected(2), ReadyState=Ready(1), Battery=unknown(0xFF)
    return frame(HEARTBEAT, bytes([2, 1, 0xFF]))


def build_ball(shot_id: int) -> bytes:
    payload = struct.pack("<IH", shot_id, 0x7F)   # ShotId + ValidMask (all 7 fields)
    #                       speed  VLA   HLA   total   back    side   axis
    payload += struct.pack("<7f", 64.0, 12.5, -1.5, 2500.0, 2450.0, -300.0, 6.98)
    return frame(BALL, payload)


def build_club(shot_id: int) -> bytes:
    payload = struct.pack("<IH", shot_id, 0xFF)   # ShotId (matches ball) + ValidMask (all 8)
    #                       speed smash  loft  face  path  attack impX  impY
    payload += struct.pack("<8f", 47.5, 1.347, 14.2, -0.8, 2.1, -3.4, 5.0, -2.0)
    return frame(CLUB, payload)


def main() -> None:
    sock = socket.create_connection(("127.0.0.1", 9760))
    sock.sendall(build_hello())          # (1) greet the host on connect

    buf = bytearray()                    # stream reassembly buffer
    shot_id = 1
    shot_sent = False

    while True:
        chunk = sock.recv(4096)
        if not chunk:
            print("disconnected")
            break
        buf += chunk                     # TCP is a stream: append, then parse

        # (2) Pull whole frames out of the buffer. There may be 0, 1, or many.
        while True:
            if len(buf) < 7:
                break                                        # no full header yet
            if buf[0] != MAGIC0 or buf[1] != MAGIC1:
                del buf[0]                                   # resync to the magic
                continue
            length = buf[5] | (buf[6] << 8)
            total = 7 + length + 4
            if len(buf) < total:
                break                                        # frame not fully arrived

            want = crc32c(bytes(buf[:7 + length]))
            got = struct.unpack("<I", bytes(buf[7 + length:7 + length + 4]))[0]
            msg_type = buf[4]
            # payload = bytes(buf[7:7 + length])  # (decode here if you consume host data)
            del buf[:total]                                  # consume the frame

            if want != got:
                print("bad crc, dropped")
                continue

            # (3) Handle by type.
            if msg_type == HELLOACK:
                print("handshake complete")
                if not shot_sent:                            # fire one demo shot
                    sock.sendall(build_ball(shot_id))
                    sock.sendall(build_club(shot_id))
                    shot_id += 1
                    shot_sent = True
            elif msg_type == PING:
                sock.sendall(build_heartbeat())              # MUST answer every Ping (keeps the link alive)
            elif msg_type == SETARM:
                print("host: SetArm")
            elif msg_type == SETCLUB:
                print("host: SetClubInfo")
            elif msg_type == SETBALL:
                print("host: SetBallType")


if __name__ == "__main__":
    main()
```

### Turning this into real firmware

- Send `BallData` (and `ClubData` if you measure club) when the ball is actually
  struck, not on handshake. Increment `shot_id` per swing; a `ClubData` must reuse
  its `BallData`'s `shot_id`.
- Send only the fields you measure: clear the corresponding `ValidMask` bit and
  the host fills gaps (e.g. it derives spin axis / total spin if you dont send it from back+side).
- Push a `ReadyState` (0x12) whenever ball detection changes.
- Keep answering `Ping` — >8 s of silence drops the link.
