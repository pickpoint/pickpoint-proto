# tracking.v2 wire format

Little-endian. One WebSocket **binary** frame = one message. Spec version lives in two places:

- Subprotocol string: `tracking.v2` (HTTP upgrade)
- `Hello.version`: `2` (first server frame)

This document is the encoding. Session behaviour (when to Resume, what Staging is) is [reconnect.md](reconnect.md).

---

## 1. Connection

```
Client                              Server
  |-- HTTP GET /v2/ws
  |   Sec-WebSocket-Protocol: tracking.v2
  |   query: client-id + client-secret
  |        OR access-token
  |---------------------------------->|
  |<----- 101 Switching Protocols ----|
  |<----- binary 0x80 Hello ----------|   always first
  |-- application frames ... -------->|
```

Reject **before** a socket:

| Condition | Result |
|-----------|--------|
| Missing / wrong subprotocol | HTTP 400 `missing tracking.v2 subprotocol` |
| Missing / bad auth | HTTP 401 |

Reject **on** the socket:

| Condition | Result |
|-----------|--------|
| Text WS frame | close **1002** |
| Truncated / illegal frame (count=0, coord out of range, oversize string) | close **1002** |
| Unknown **client** type in `0x01–0x7E` | `0x88 Error INVALID`, session **stays up** |
| Unknown **server** type in `0x80–0xFE` | client **ignores** the frame (forward compatible) |
| Extra bytes after a known body | **ignore** (space for optional tails later) |

Reserved type bytes — never send: `0x00`, `0x7F`, `0xFF` (future escape), `0x8C`.

Keepalive: WS ping/pong only. No app Ping/Pong.

---

## 2. Primitive types

| Name | Encoding |
|------|----------|
| `u8` `u16` `u32` `u64` | unsigned, little-endian |
| `i16` `i32` `i64` | two’s complement, little-endian |
| `f64` | IEEE-754 binary64, little-endian. **Only** on `Subscribed` estimates. Never on live Loc. |
| UUID | 16 raw RFC 4122 bytes (not ASCII, not hex). Nil UUID = 16 zero bytes = “none”. |
| string | `u16` byte length + UTF-8, length ≤ 4096 |
| bytes | `u16` length + payload, length ≤ 4096 |

**Coordinates** are integer microdegrees:

```
μ° = round(degrees × 1_000_000)   as i32
degrees = μ° / 1_000_000
```

Valid range: latitude `±90_000_000`, longitude `±180_000_000`. Outside → illegal frame (close 1002).

**Seq** on the wire is `u32`. It is assigned by the **device SDK** when a filtered point leaves Staging (see [reconnect.md](reconnect.md)). The server stores it as the track cursor `last_acked`.

---

## 3. Point (shared tail)

Used in client `Loc` / `TrackStart` start-point, and server listener `Loc` / `Subscribed.last_location`.

```
flags     u8
lat/lon   see below
[alt      i32  millimetres]     if flags bit 0
[acc      u16  centimetres]     if flags bit 1
[time     i64  unix ms]         if flags bit 4
```

| Bit | Meaning |
|-----|---------|
| 0 | altitude present (`i32` mm → metres = mm / 1000) |
| 1 | accuracy present (`u16` cm → metres = cm / 100) |
| 2–3 | **reserved**. Heading and speed are **not** on the wire. |
| 4 | timestamp present |
| 5–7 | reserved for future optional fields |

### Absolute vs delta

- **Always absolute** (`i32` lat, `i32` lon): first point of a client `Loc`, every server `0x86 Loc`, `TrackStart` start point, `Subscribed.last_location`.
- **Delta** (`i16` dlat, `i16` dlon in μ° from the **previous point in this same frame**): client `Loc` points `1 .. count-1` only.

There is **no** delta across frames. If `lat_n - lat_{n-1}` does not fit in `i16` (±32767 μ° ≈ 0.033° ≈ 3.6 km), **do not batch** — send a new `Loc`.

Route vertices (`TrackStart.route`, `Subscribed.route`) are **only** absolute `i32` lat + `i32` lon. No flags, no alt/time.

### Time

- Live ~1 Hz: omit bit 4. Server stamps `now` on ingest.
- Anything that sat in Staging (offline / reconnect flush) **must** set bit 4 to the **capture** time. Otherwise a reconnect would rewrite the whole trail as “now”.

### Worked example — one live point

`55°N, 37°E`, no alt/accuracy/time, `seq = 1`, `count = 1`:

```
μlat = 55_000_000 = 0x0346DC40  → LE  40 DC 46 03
μlon = 37_000_000 = 0x0234B740  → LE  40 B7 34 02

04                  type Loc
01 00 00 00         seq = 1
01                  count = 1
00                  flags (abs lat/lon only)
40 DC 46 03         lat
40 B7 34 02         lon
```

15 bytes application payload. (WS mask + TLS + TCP/IP around it is ~80–100 bytes on the wire; at 1 Hz that is ~100 B/s. Do not enable `permessage-deflate`.)

---

## 4. Who may send what

Client types `0x01–0x7E`. Server types `0x80–0xFE`.

| | Device | Listener |
|--|--------|----------|
| `0x01` Resume | yes, after Hello, if `track_uid` remembered | no |
| `0x02` TrackStart | yes | `UNAUTHORIZED` |
| `0x03` TrackStop | yes | `UNAUTHORIZED` |
| `0x04` Loc | yes, only with `active_track` | `UNAUTHORIZED` |
| `0x05` Subscribe | `UNAUTHORIZED` | yes |
| `0x06` Unsubscribe | — | yes |
| `0x07` Event | yes, active track | `UNAUTHORIZED` |
| `0x08` CommandAck | yes | — |
| `0x80` Hello | receive | receive |
| `0x81` Relocate | receive | rare |
| `0x82` ResumeOk | receive | — |
| `0x83` / `0x84` TrackStarted / Stopped | receive (own trip) | receive (subscribed device) |
| `0x85` Ack | receive | never |
| `0x86` Loc | never | receive |
| `0x87` Subscribed | — | receive |
| `0x88` Error | receive | receive |
| `0x89` EventAdded | never (not echoed) | receive |
| `0x8A` Command | receive | — |
| `0x8B` Presence | never | receive |

---

## 5. Client frames

### `0x01` Resume

Sent by the SDK after Hello when it still holds a `track_uid`. The app never calls this. See [reconnect.md](reconnect.md).

```
track_uid     uuid[16]
last_seq      u32      last seq the SDK *assigned* (not a request to set the server cursor)
```

The server **ignores** `last_seq` as truth and replies with its own `last_acked`.

### `0x02` TrackStart

```
flags         u8       bit 0 = start point present
[point]                if bit 0: one absolute Point (own flags)
n_route       u16      0 = none
route[i]               i32 lat, i32 lon   (absolute, no point-flags)
metadata      bytes    opaque, may be length 0
```

Means “this is my current trip”. If the server still has track A open, it **supersedes** A (see reconnect). Seq on the new track starts at 0.

### `0x03` TrackStop

Empty body. Uses the session’s `active_track`.

- No active track → **no-op** (not `INVALID`).
- Otherwise finish that track, `0x84 TrackStopped`.

### `0x04` Loc

Device only. No `track_uid` — the session’s `active_track`.

```
seq           u32      seq of the **last** point in this frame
count         u8       1…100
point[0]               absolute Point
point[1..n)            delta Point (i16 dlat/dlon)
```

The points occupy the contiguous range `seq-count+1 … seq`. Holes inside a frame are forbidden.

```
count=1  seq=41     → point 41
count=3  seq=45     → points 43, 44, 45
```

- No `active_track` / unknown track → `TRACK_NOT_FOUND`
- `seq <= last_acked` → ingest **no-op**, still `Ack` that `seq` (idempotent replay)
- `seq > last_acked` → accept, `last_acked = seq`, `Ack(seq)` to the device; listeners may get `0x86` (rate-capped)

### `0x05` Subscribe

Listener only.

```
device_uid    uuid[16]
flags         u8       bit 0 = include_events (SDK should set 1 unless the app opted out)
min_interval  u16      extra Loc throttle in ms, 0 = none beyond the server 50 Hz cap
```

Reply is `0x87 Subscribed` with a `sub` handle `1…255`. Subscribe again to the same device → **same** `sub`, filters updated. 256th distinct device → `INVALID`.

### `0x06` Unsubscribe

```
sub           u8       handle from Subscribed (not a UUID)
```

Unknown `sub` → no-op.

### `0x07` Event

```
payload       bytes    ≤ 4 KiB
time          i64      unix ms (0 = server may treat as absent)
```

Active track only. Not stored, not resumed. Server 1 Hz: extra events dropped, no error. No track → `TRACK_NOT_FOUND`.

### `0x08` CommandAck

```
command_id    uuid[16]
status        u8       1 = ok, 2 = rejected, 3 = failed
message       string   empty = none
```

Unknown `command_id` → ignore.

---

## 6. Server frames

### `0x80` Hello — always first

```
version       u8       2
shard         u16      home shard for a device session; 0 for a listener
node_id       uuid[16]
```

Client that does not implement this `version` closes the socket.

### `0x81` Relocate

```
retry_ms      u32
endpoint      string   e.g. wss://n2.example/v2/ws
```

This node is not the home (or is draining). Client closes, dials `endpoint` **with the same query auth**, waits for Hello, then **Resume** if it has a `track_uid` (device) or **Subscribe** again (listener). Never TrackStart just because of Relocate.

### `0x82` ResumeOk

```
track_uid     uuid[16]
last_acked    u32      server cursor; SDK ackThrough then flush (reconnect.md)
```

Binds `active_track` so subsequent `Loc` frames need no uid.

### `0x83` TrackStarted

```
track_uid     uuid[16]
metadata      bytes
```

### `0x84` TrackStopped

```
track_uid     uuid[16]
```

No `finish_reason` on WS. That lives on the HTTP track read (`client_stop` | `superseded` | `idle`).

### `0x85` Ack — device only

```
seq           u32      ack **through** (inclusive)
```

SDK drops InFlight entries with `seq <= ack`. This is not a live map event. Device `onLocation` does not exist.

### `0x86` Loc — listeners only

Always one absolute point (no count, no delta). Throttle may skip points, so you cannot reconstruct a polyline from these alone.

```
sub           u8       handle from Subscribed for this device
seq           u32
point                  absolute Point (flags + lat/lon + optionals)
```

### `0x87` Subscribed

Ack of Subscribe **and** a snapshot. There is no separate “listener subscribed” frame.

```
sub           u8
device_uid    uuid[16]
track_uid     uuid[16]    16 zeros = device has no live track
online        u8          0 or 1
flags         u8          bit0 last_location, bit1 last_seen, bit2 route
[last_loc]                absolute Point if bit0
[last_seen    i64]        if bit1
[n_route u16 + route[i] i32 lat lon]   if bit2
est_dist      f64         metres, 0 if unknown
est_dur       f64         seconds, 0 if unknown
start_name    string
end_name      string
metadata      bytes
```

`f64` appears only here. Re-subscribe to the same device reuses `sub`.

### `0x88` Error

```
code          u8
retry_ms      u32         0 = none
track_uid     uuid[16]    zeros = none
message       string
```

| code | Name | Typical meaning |
|------|------|-----------------|
| 1 | AUTH | credentials / token |
| 2 | TRACK_NOT_FOUND | no/wrong/finished track; fatal for Resume |
| 3 | FENCED | shard draining; retry Resume |
| 4 | TRY_AGAIN | backoff, retry |
| 5 | INVALID | bad frame contents; session usually lives |
| 6 | UNAUTHORIZED | wrong role (listener sent Loc, …) |

Resume: **fatal** = AUTH, TRACK_NOT_FOUND. **Retry** = FENCED, TRY_AGAIN (do not TrackStart).

### `0x89` EventAdded — listener

```
sub           u8
payload       bytes
time          i64
```

### `0x8A` Command — device

```
command_id    uuid[16]
payload       bytes
time          i64
```

### `0x8B` Presence — listener

```
sub           u8
online        u8
last_seen     i64         0 = none
```

---

## 7. Evolution

1. New **server→client** frame: next free `0x8D…0xFE`. Old clients ignore. Do not bump `Hello.version`.
2. New **client→server** frame: next free `0x09…0x7E`. Old server replies `INVALID`. Ship the frame only if `Hello.version` is high enough, or bump version if the feature is required.
3. New field on an existing frame: only as a **tail** after the known body, plus a bit in an existing `flags` if needed. Decoders must ignore trailing bytes.
4. Break the layout → new subprotocol (`tracking.v3`), not a new type byte.
