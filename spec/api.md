# Tracking API

Host: **`tracking.pickpoint.io`**. Live session is a WebSocket; the frame layout is [wire.md](wire.md). Track history is REST on public-api, not this host. Beacons that cannot keep a socket use HTTP ingest with the same client frames.

| | |
|--|--|
| `wss://tracking.pickpoint.io/v2/ws` | Device or listener session. Subprotocol `tracking.v2`. |
| `POST /v2/ingest` | Beacon ingest. Concatenated `tracking.v2` **client** frames. No Hello. |
| `POST /v2/devices/{uid}/command` | Opaque command to an online **WebSocket** device (same host). |

## Device

```
wss://tracking.pickpoint.io/v2/ws?client-id=<device-uid>&client-secret=<secret>
```

`client-id` is the device UID (UUID). After `Hello` the session may start / stop / resume one live track, send `Loc` and `Event`, and receive `Ack`, `TrackStarted` / `TrackStopped`, `ResumeOk`, `Command`, `Error`, `Relocate`.

A device does not receive live `0x86 Loc`. Ingest receipt is `0x85 Ack`.

## Listener

```
wss://tracking.pickpoint.io/v2/ws?access-token=<jwt>
```

`<jwt>` is a Pickpoint **client-token** `access_token` (`POST /v2/client-tokens`, scope `devices`) — the same Bearer token as HTTP `client_auth`. Mint it on the integrator’s backend; the secret API key must not go in the dashboard. The session is scoped to that account’s devices.

After `Hello`, `Subscribe` per device. The listener gets a `Subscribed` snapshot plus live `Loc` / `EventAdded` / `Presence` / track start-stop.

A listener does not Resume a track. After a drop: new socket → `Hello` → `Subscribe` again. Missed live points are not replayed; use HTTP history on public-api.

## Handshake

1. Upgrade with `Sec-WebSocket-Protocol: tracking.v2`. Wrong or missing → HTTP 400.
2. Query auth. Missing or bad → HTTP 401.
3. Wrong home shard (device) → `0x81 Relocate` then close ([reconnect.md](reconnect.md#7-relocate)).
4. First binary frame is always `0x80 Hello` with `version = 2`. Unknown version → client closes.
5. Do not send application frames before `Hello`.

Keepalive is WebSocket ping/pong. One binary WS message = one protocol frame. Text → close 1002.

## HTTP ingest (beacons)

`POST https://tracking.pickpoint.io/v2/ingest`

For GPS beacons that wake a modem, POST, and sleep. Same binary frames as the device WebSocket. No `Hello`. No listener. No command mailbox (`0x8A`).

### Auth

Server accepts **either** (headers win if both are present):

1. Query (lowest common denominator; SIM800-class modules):  
   `?client-id=<device-uid>&client-secret=<secret>`
2. Headers: `X-Client-Id` / `X-Client-Secret`

`access-token` is rejected (**401**). HTTPS still covers the air; a secret in the query may appear in proxy logs.

`Content-Type` should be `application/octet-stream`. The server parses **body bytes**, not the header.

### Body

Concatenated complete client frames (little-endian). Not JSON. No extra-bytes-ignore: leftover bytes after a frame are the next frame.

| Wake | Frames |
|------|--------|
| First | `0x02 TrackStart` then `0x04 Loc` |
| Later | `0x04 Loc` only (repeat unacked tail + new points) |
| Stop | `0x03 TrackStop` (same POST as last `Loc` when possible) |

Do not send `Resume`, `Subscribe`, `Event`, or `CommandAck` on this path.

200 response body is concatenated **server** frames: `0x83 TrackStarted`, `0x85 Ack`, `0x84 TrackStopped`, `0x88 Error`, `0x81 Relocate`. No `0x80 Hello`, no `0x86 Loc`, no `0x8A Command`.

`Relocate.endpoint` is the node’s public URL (often `ws://host:port`). HTTP clients take host/port and POST `/v2/ingest` on that host (`https` in production).

Idempotency: `seq <= last_acked` is a no-op Ack. There is no separate Resume round-trip.

### Limits (requests, not WebSocket 50 Hz)

WebSocket caps **points** at 50 Hz on one socket. HTTP cannot copy that: each POST is TLS + auth. Cap **requests**. A batch of points in one POST is a normal drain, not an attack.

| | |
|--|--|
| Token bucket | **1 POST/s** sustained, burst **2**, per `client-id` |
| In-flight | **1** POST per device; a second overlapping POST → **429** |
| Empty bucket | **429** + `Retry-After: 1` **before** parsing the body |
| Body | **≤ 8 KiB** → else **413** |
| Frames | **≤ 8** client frames per POST |
| Points | **≤ 100** total (`TrackStart` location + `Loc` points) |
| Fan-out | L1 still 50 Hz to listeners inside a batch |

Oversize frames/points → **413** (not 200). Truncated / illegal bytes → **400** with `0x88 Error INVALID` when a body is useful.

### Presence

HTTP ingest does **not** open a command-capable device session (inject stays **409**). `last_seen` updates on an accepted POST. Listeners see `online` if a WebSocket device session exists **or** an HTTP ingest succeeded within **5 minutes**. Do not flicker offline at the end of each POST.

## Limits

| | |
|--|--|
| WS ingest `Loc` | 50 Hz per track (defense; a correct SDK is ~1 Hz after the filter) |
| HTTP ingest | 1 POST/s, burst 2 (see above). Not 50 POST/s. |
| `Loc.count` | 1…100 |
| Event | 4 KiB, 1 Hz per track (extras dropped) |
| Track metadata | 4 KiB |
| Command | 4 KiB |
| String / bytes fields | `u16` length, max 4096 |
| Staging / InFlight (WS SDK) | 10_000 points ([reconnect.md](reconnect.md)) |
| Beacon queue (`pickpoint-nano`) | 256–1024 points, drop oldest |
| Listener subscriptions | 255 per session (`sub` = 1…255) |

## Command

`POST https://tracking.pickpoint.io/v2/devices/{uid}/command`

- Body: raw bytes or JSON `{ "payload": "<base64>" }`
- Server pushes `0x8A Command` on every online device session for that uid
- Waits ~5 s for `0x08 CommandAck`
- 200 `{ commandId, delivered, status, message }` · offline **409** · ack timeout **504** · too large **400**

Commands are not stored, not resumed, not inherited across TrackStart supersede. `Authorization: Bearer` is accepted here; the WebSocket uses the query token.

## Presence

`0x8B Presence` when a **WebSocket** device session opens or the last one closes. `last_seen` is throttled (~5 s) while publishing. HTTP ingest updates `last_seen` and may emit `Presence` online (throttled) without a matching offline on request end. `Subscribed.online` / `last_seen` is the snapshot at subscribe time.

## Track history (HTTP)

Finished tracks on public-api include `finishReason`: `client_stop` | `superseded` | `idle`. It is not on the WebSocket `TrackStopped` frame.

## Related

[wire.md](wire.md) · [reconnect.md](reconnect.md) · [filter.md](filter.md) · [goldens.md](goldens.md)
