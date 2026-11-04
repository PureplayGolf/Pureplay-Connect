# Pureplay Connect — minimal device in C++

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
4. Replies to every `Ping` with a `Heartbeat` — this is the one hard requirement
   that keeps the link alive.
5. Logs the host's commands (`SetDetect` / `SetClubInfo` / `SetBallType` /
   `SetShotContext`), and acknowledges `SetStrictMode`.

### Build & run

```sh
# Linux / macOS
g++ -std=c++17 pureplay_connect_device.cpp -o device && ./device
```

On **Windows**, sockets differ slightly: call `WSAStartup()` at the start, link
`ws2_32.lib`, and use `closesocket()` instead of `close()`. The protocol code is
identical.

### `pureplay_connect_device.cpp`

```cpp
// Minimal Pureplay Connect device. POSIX sockets.
#include <cstdint>
#include <cstring>
#include <cstdio>
#include <string>
#include <vector>
#include <unistd.h>       // close()
#include <arpa/inet.h>    // inet_pton, htons
#include <sys/socket.h>   // socket, connect, send, recv

// ---- Protocol constants ----
// MAGIC0/MAGIC1 spell "PL" (Pureplay) — the two bytes every frame starts with, so
// a reader can find/resync to frame boundaries and reject non-PPConnect data.
static const uint8_t MAGIC0 = 0x50, MAGIC1 = 0x4C, WIRE_VERSION = 1;
enum : uint8_t {
    MSG_HELLO = 0x01, MSG_HEARTBEAT = 0x02, MSG_STRICTACK = 0x04, MSG_BALL = 0x10, MSG_CLUB = 0x11,
    MSG_READY = 0x12, MSG_ERROR = 0x1F, MSG_HELLOACK = 0x81, MSG_PING = 0x82,
    MSG_SETCLUB = 0x90, MSG_SETDETECT = 0x91, MSG_SETBALL = 0x92, MSG_SETSHOT = 0x93, MSG_SETSTRICT = 0x94
};

// ---- CRC32C: catches corrupted/truncated frames so they're dropped, not
// mis-parsed. Castagnoli variant; must match the host byte-for-byte. ----
static uint32_t crc32c(const uint8_t* data, size_t len) {
    uint32_t crc = 0xFFFFFFFFu;
    for (size_t i = 0; i < len; ++i) {
        crc ^= data[i];
        for (int b = 0; b < 8; ++b)
            crc = (crc & 1) ? (crc >> 1) ^ 0x82F63B78u : (crc >> 1);
    }
    return crc ^ 0xFFFFFFFFu;
}

// ---- Little-endian append helpers (all multi-byte values are LE) ----
static void putU16(std::vector<uint8_t>& v, uint16_t x) {
    v.push_back(x & 0xFF); v.push_back((x >> 8) & 0xFF);
}
static void putU32(std::vector<uint8_t>& v, uint32_t x) {
    v.push_back(x & 0xFF); v.push_back((x >> 8) & 0xFF);
    v.push_back((x >> 16) & 0xFF); v.push_back((x >> 24) & 0xFF);
}
static void putF32(std::vector<uint8_t>& v, float f) {
    uint32_t bits; std::memcpy(&bits, &f, 4); putU32(v, bits); // IEEE-754 float32
}
static float getF32(const std::vector<uint8_t>& p, size_t at) {
    uint32_t bits = (uint32_t)p[at] | ((uint32_t)p[at + 1] << 8)
                  | ((uint32_t)p[at + 2] << 16) | ((uint32_t)p[at + 3] << 24);
    float f; std::memcpy(&f, &bits, 4); return f;
}
static void putStr(std::vector<uint8_t>& v, const std::string& s) {
    v.push_back((uint8_t)s.size());                 // uint8 length prefix
    v.insert(v.end(), s.begin(), s.end());          // then UTF-8 bytes
}

// ---- Wrap a payload in a frame: magic|ver|flags|type|len|payload|crc32c ----
static std::vector<uint8_t> frame(uint8_t type, const std::vector<uint8_t>& payload) {
    std::vector<uint8_t> f;
    f.push_back(MAGIC0); f.push_back(MAGIC1); f.push_back(WIRE_VERSION);
    f.push_back(0);                    // flags = 0 (reserved; must be 0)
    f.push_back(type);
    putU16(f, (uint16_t)payload.size());
    f.insert(f.end(), payload.begin(), payload.end());
    putU32(f, crc32c(f.data(), f.size())); // CRC over header + payload
    return f;
}

// ---- Message builders ----
static std::vector<uint8_t> buildHello() {
    std::vector<uint8_t> p;
    p.push_back(WIRE_VERSION);   // ProtocolVersion
    putU16(p, 0);                // DeviceCaps = 0 (no optional capabilities advertised)
    putStr(p, "LMDeviceName");   // DeviceName
    putStr(p, "1.0.0");          // Firmware
    return frame(MSG_HELLO, p);
}
static std::vector<uint8_t> buildHeartbeat() {
    std::vector<uint8_t> p;
    p.push_back(2);      // ConnectionState = Connected
    p.push_back(1);      // ReadyState      = Ready
    p.push_back(0xFF);   // Battery         = unknown
    return frame(MSG_HEARTBEAT, p);
}
static std::vector<uint8_t> buildBall(uint32_t shotId) {
    std::vector<uint8_t> p;
    putU32(p, shotId);
    putU16(p, 0x7F);     // ValidMask: all 7 ball fields present
    putF32(p, 64.0f);    // BallSpeedMS
    putF32(p, 12.5f);    // VLA
    putF32(p, -1.5f);    // HLA
    putF32(p, 2500.0f);  // TotalSpinRPM
    putF32(p, 2450.0f);  // BackSpinRPM
    putF32(p, -300.0f);  // SideSpinRPM
    putF32(p, 6.98f);    // SpinAxisDeg
    return frame(MSG_BALL, p);
}
static std::vector<uint8_t> buildClub(uint32_t shotId) {
    std::vector<uint8_t> p;
    putU32(p, shotId);   // MUST match the paired BallData.ShotId
    putU16(p, 0xFF);     // ValidMask: all 8 club fields present
    putF32(p, 47.5f);    // ClubHeadSpeedMS
    putF32(p, 1.347f);   // SmashFactor
    putF32(p, 14.2f);    // DynamicLoftDeg
    putF32(p, -0.8f);    // FaceAngleDeg
    putF32(p, 2.1f);     // ClubPathDeg
    putF32(p, -3.4f);    // AttackAngleDeg
    putF32(p, 5.0f);     // ImpactPointXMM (+toe)
    putF32(p, -2.0f);    // ImpactPointYMM (+high)
    return frame(MSG_CLUB, p);
}

static void sendAll(int fd, const std::vector<uint8_t>& f) {
    send(fd, (const char*)f.data(), f.size(), 0);
}

int main() {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port   = htons(9760);              // host listen port
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
    if (connect(fd, (sockaddr*)&addr, sizeof(addr)) != 0) { perror("connect"); return 1; }

    sendAll(fd, buildHello());                  // (1) greet the host

    std::vector<uint8_t> buf;                    // stream reassembly buffer
    uint8_t rx[4096];
    uint32_t shotId = 1;
    bool shotSent = false;

    for (;;) {
        ssize_t n = recv(fd, (char*)rx, sizeof(rx), 0);
        if (n <= 0) { printf("disconnected\n"); break; }
        buf.insert(buf.end(), rx, rx + n);       // TCP is a stream: append, then parse

        // (2) Pull whole frames out of the buffer. There may be 0, 1, or many.
        for (;;) {
            if (buf.size() < 7) break;            // not even a full header yet
            if (buf[0] != MAGIC0 || buf[1] != MAGIC1) { buf.erase(buf.begin()); continue; } // resync
            uint16_t len = (uint16_t)(buf[5] | (buf[6] << 8));
            size_t total = 7u + len + 4u;
            if (buf.size() < total) break;        // frame not fully arrived

            uint32_t want = crc32c(buf.data(), 7 + len);
            uint32_t got  = (uint32_t)buf[7 + len]
                          | ((uint32_t)buf[7 + len + 1] << 8)
                          | ((uint32_t)buf[7 + len + 2] << 16)
                          | ((uint32_t)buf[7 + len + 3] << 24);
            uint8_t type = buf[4];
            std::vector<uint8_t> payload(buf.begin() + 7, buf.begin() + 7 + len);
            buf.erase(buf.begin(), buf.begin() + total); // consume the frame

            if (want != got) { printf("bad crc, dropped\n"); continue; }

            // (3) Handle by type.
            if (type == MSG_HELLOACK) {
                printf("handshake complete\n");
                if (!shotSent) {                  // fire one demo shot
                    sendAll(fd, buildBall(shotId));
                    sendAll(fd, buildClub(shotId));
                    ++shotId; shotSent = true;
                }
            } else if (type == MSG_PING) {
                sendAll(fd, buildHeartbeat());     // MUST answer every Ping (keeps the link alive)
            } else if (type == MSG_SETDETECT) {
                printf("host: SetDetect\n");
            } else if (type == MSG_SETCLUB) {
                printf("host: SetClubInfo\n");
            } else if (type == MSG_SETBALL) {
                printf("host: SetBallType\n");
            } else if (type == MSG_SETSHOT) {      // distance to the target in metres; <= 0 = unknown
                if (payload.size() >= 4) printf("host: SetShotContext %.1f m\n", getF32(payload, 0));
            } else if (type == MSG_SETSTRICT) {    // an event or tournament round: measured data only
                // 1 = accepted. A device that cannot switch off interpolation or clamping must send 0 and a
                // reason; the host then closes the link and takes no shots from it.
                std::vector<uint8_t> p; p.push_back(1); putStr(p, "");
                sendAll(fd, frame(MSG_STRICTACK, p));
                printf("host: SetStrictMode %d -> accepted\n", payload.empty() ? -1 : payload[0]);
            }
        }
    }
    close(fd);
    return 0;
}
```

### Turning this into real firmware

- Send `BallData` (and `ClubData` if you measure club) when the ball is actually
  struck, not on handshake. Increment `ShotId` per swing; a `ClubData` must reuse
  its `BallData`'s `ShotId`.
- Send only the fields you measure: clear the corresponding `ValidMask` bit and
  the host fills gaps (e.g. it derives spin axis / total spin if you don't send them from back+side).
- `SetShotContext` arrives just before each `SetDetect(1)` with the distance to the
  pin (or drill target) in metres. Use it to pick a short-game capture mode if
  your device has one; ignore it otherwise.
- `SetStrictMode` (0x94) arrives after the handshake whenever an event or
  tournament round is in progress, and again after every reconnect. Answer with
  `StrictModeAck` (0x04) within 5 s: `1` promises measured data only (no
  clamping, smoothing, interpolation or synthesised fields); `0` plus a reason
  if you cannot, in which case the host drops the link.
- Push a `ReadyState` (0x12) whenever ball detection changes.
- Keep answering `Ping` — >8 s of silence drops the link.
