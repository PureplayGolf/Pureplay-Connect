# Pureplay Connect — minimal device in JavaScript (Node.js)

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
5. Logs the host's commands (`SetDetect` / `SetClubInfo` / `SetBallType`).

### Run

```sh
node pureplay_connect_device.js
```

Node's built-in `net` module is all that's needed — no dependencies.

### `pureplay_connect_device.js`

```js
// Minimal Pureplay Connect device.
const net = require('net');

// ---- Protocol constants ----
// MAGIC0/MAGIC1 spell "PL" (Pureplay) — the two bytes every frame starts with, so
// a reader can find/resync to frame boundaries and reject non-PPConnect data.
const MAGIC0 = 0x50, MAGIC1 = 0x4C, WIRE_VERSION = 1;
const MSG = {
  HELLO: 0x01, HEARTBEAT: 0x02, BALL: 0x10, CLUB: 0x11, READY: 0x12,
  ERROR: 0x1F, HELLOACK: 0x81, PING: 0x82, SETCLUB: 0x90, SETDETECT: 0x91, SETBALL: 0x92,
};

// ---- CRC32C: catches corrupted/truncated frames so they're dropped, not
// mis-parsed. Castagnoli variant; must match the host byte-for-byte.
// `>>> 0` keeps the value an unsigned 32-bit int throughout. ----
function crc32c(buf) {
  let crc = 0xFFFFFFFF;
  for (const b of buf) {
    crc ^= b;
    for (let i = 0; i < 8; i++) crc = (crc & 1) ? ((crc >>> 1) ^ 0x82F63B78) : (crc >>> 1);
  }
  return (crc ^ 0xFFFFFFFF) >>> 0;
}

// ---- Wrap a payload Buffer in a frame: magic|ver|flags|type|len|payload|crc32c ----
function frame(type, payload) {
  const head = Buffer.alloc(7);
  head[0] = MAGIC0; head[1] = MAGIC1; head[2] = WIRE_VERSION;
  head[3] = 0;                       // flags = 0 (reserved; must be 0)
  head[4] = type;
  head.writeUInt16LE(payload.length, 5);   // little-endian length
  const body = Buffer.concat([head, payload]);
  const crc = Buffer.alloc(4);
  crc.writeUInt32LE(crc32c(body), 0);      // CRC over header + payload
  return Buffer.concat([body, crc]);
}

// Pack an array of numbers as little-endian float32s.
function floats(values) {
  const b = Buffer.alloc(values.length * 4);
  values.forEach((f, i) => b.writeFloatLE(f, i * 4));
  return b;
}
// A length-prefixed UTF-8 string (uint8 length + bytes).
function pstr(s) {
  const bytes = Buffer.from(s, 'utf8');
  return Buffer.concat([Buffer.from([bytes.length]), bytes]);
}

// ---- Message builders ----
function buildHello() {
  const caps = Buffer.alloc(2); caps.writeUInt16LE(0, 0);   // DeviceCaps = 0
  const payload = Buffer.concat([
    Buffer.from([WIRE_VERSION]),   // ProtocolVersion
    caps,                          // DeviceCaps (no optional capabilities advertised)
    pstr('LMEmulator'),            // DeviceName
    pstr('1.0.0'),                 // Firmware
  ]);
  return frame(MSG.HELLO, payload);
}
function buildHeartbeat() {
  // ConnectionState=Connected(2), ReadyState=Ready(1), Battery=unknown(0xFF)
  return frame(MSG.HEARTBEAT, Buffer.from([2, 1, 0xFF]));
}
function buildBall(shotId) {
  const head = Buffer.alloc(6);
  head.writeUInt32LE(shotId, 0);
  head.writeUInt16LE(0x7F, 4);     // ValidMask: all 7 ball fields present
  //          speed  VLA   HLA   total  back   side  axis
  const body = floats([64.0, 12.5, -1.5, 2500, 2450, -300, 6.98]);
  return frame(MSG.BALL, Buffer.concat([head, body]));
}
function buildClub(shotId) {
  const head = Buffer.alloc(6);
  head.writeUInt32LE(shotId, 0);   // MUST match the paired BallData.ShotId
  head.writeUInt16LE(0xFF, 4);     // ValidMask: all 8 club fields present
  //          speed smash  loft  face  path  attack impX  impY
  const body = floats([47.5, 1.347, 14.2, -0.8, 2.1, -3.4, 5.0, -2.0]);
  return frame(MSG.CLUB, Buffer.concat([head, body]));
}

// ---- Connection ----
let buf = Buffer.alloc(0);   // stream reassembly buffer
let shotId = 1, shotSent = false;

const sock = net.createConnection({ host: '127.0.0.1', port: 9760 }, () => {
  sock.write(buildHello());   // (1) greet the host as soon as we connect
});

sock.on('data', (chunk) => {
  buf = Buffer.concat([buf, chunk]);   // TCP is a stream: append, then parse

  // (2) Pull whole frames out of the buffer. There may be 0, 1, or many.
  for (;;) {
    if (buf.length < 7) break;                                   // no full header
    if (buf[0] !== MAGIC0 || buf[1] !== MAGIC1) { buf = buf.slice(1); continue; } // resync
    const len = buf.readUInt16LE(5);
    const total = 7 + len + 4;
    if (buf.length < total) break;                               // frame incomplete

    const want = crc32c(buf.slice(0, 7 + len));
    const got = buf.readUInt32LE(7 + len);
    const type = buf[4];
    // const payload = buf.slice(7, 7 + len);  // (decode here if you consume host data)
    buf = buf.slice(total);                                      // consume the frame

    if (want !== got) { console.log('bad crc, dropped'); continue; }

    // (3) Handle by type.
    if (type === MSG.HELLOACK) {
      console.log('handshake complete');
      if (!shotSent) {                    // fire one demo shot
        sock.write(buildBall(shotId));
        sock.write(buildClub(shotId));
        shotId++; shotSent = true;
      }
    } else if (type === MSG.PING) {
      sock.write(buildHeartbeat());        // MUST answer every Ping (keeps the link alive)
    } else if (type === MSG.SETDETECT)  { console.log('host: SetDetect'); }
    else if (type === MSG.SETCLUB) { console.log('host: SetClubInfo'); }
    else if (type === MSG.SETBALL) { console.log('host: SetBallType'); }
  }
});

sock.on('close', () => console.log('disconnected'));
sock.on('error', (e) => console.log('error:', e.message));
```

### Turning this into real firmware

- Send `BallData` (and `ClubData` if you measure club) when the ball is actually
  struck, not on handshake. Increment `shotId` per swing; a `ClubData` must reuse
  its `BallData`'s `shotId`.
- Send only the fields you measure: clear the corresponding `ValidMask` bit and
  the host fills gaps (e.g. it derives spin axis / total spin from back+side).
- Push a `ReadyState` (0x12) whenever ball detection changes.
- Keep answering `Ping` — >8 s of silence drops the link.
