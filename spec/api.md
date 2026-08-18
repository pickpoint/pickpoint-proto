# Tracking HTTP / session surface

Realtime GPS is **only** the WebSocket in [wire.md](wire.md). No JSON on that socket. No HTTP ingest for live points.

## Paths

| Path | Role |
|------|------|
| `GET /v2/tracking/ws` | Binary WebSocket. Subprotocol **`tracking.v2`** is required. |
| `POST /v2/devices/{uid}/command` | Inject an opaque command to an **online** device (see below). |
| `GET /healthz` | Process is up. |
| `GET /readyz` | 503 while the node is draining a shard. |

History reads are **not** this service:

| Endpoint | Owner |
|----------|-------|
| `GET /v2/devices/{uid}/tracks`, `…/tracks/{trackUid}` | public-api |
| Finished-track polyline, names, `finish_reason` | HTTP track read (MobilityDB) |

## Two kinds of client

The same URL, different auth, different frames.

### Device (publisher)

Auth as query params:

```
GET /v2/tracking/ws?client-id=<device-uid>&client-secret=<secret>
```

`client-id` **is** the device UID (UUID). The session may:

- start / stop / resume **one** live track
- send `Loc` and `Event` for that track
- receive `Ack`, `TrackStarted` / `TrackStopped`, `ResumeOk`, `Command`, `Error`, `Relocate`

A device never receives live `0x86 Loc` (that would be an echo of its own GPS). Ingest receipt is `0x85 Ack`.

### Listener (subscriber)

Auth:

```
GET /v2/tracking/ws?access-token=<jwt>
```

(`Authorization: Bearer` is accepted on the command HTTP route; WS uses the query token.)

After `Hello`, the listener sends `Subscribe` per device it cares about. It receives a `Subscribed` snapshot plus live `Loc` / `EventAdded` / `Presence` / track start-stop for those devices.

A listener **does not** Resume a track. After a socket drop it dials again, waits for `Hello`, and sends `Subscribe` again. Missed live points are not replayed on WS; use HTTP history.

## Handshake (both roles)

1. Client opens WS with `Sec-WebSocket-Protocol: tracking.v2`. Missing / wrong subprotocol → HTTP 400, no socket.
2. Server authenticates the query. Bad/missing creds → HTTP 401.
3. If this node is not the device’s home shard (device only) → the socket may still open long enough to send `0x81 Relocate` and close (see [reconnect.md](reconnect.md#relocate)).
4. First **binary** frame from the server is always `0x80 Hello` with `version = 2`.
5. If `Hello.version` is not `2`, the client closes. Do not send application frames first.

Keepalive is **WebSocket ping/pong** (browser / OS stack). There is no app-level Ping frame.

One WS **binary** message = one protocol frame. Text frames → close 1002.

## Limits (server enforces)

| What | Limit |
|------|--------|
| Ingest `Loc` | 50 Hz per track (defense; a correct SDK is ~1 Hz after the filter) |
| `Loc.count` | 1…100 |
| Event payload | 4 KiB, 1 Hz per track (excess events dropped silently) |
| Track metadata | 4 KiB |
| Command payload | 4 KiB |
| String / bytes fields | `u16` length, max 4096 |
| Staging / InFlight (SDK) | 10_000 points (RAM only; see [reconnect.md](reconnect.md)) |
| Listener subscriptions | 255 per session (`sub` = 1…255) |

## Device command (HTTP → WS)

`POST /v2/devices/{uid}/command`

- Body: raw bytes (`application/octet-stream`) or JSON `{ "payload": "<base64>" }`
- Server pushes `0x8A Command` on every **online** device WS for that uid
- Waits ~5 s for `0x08 CommandAck`
- Online + ack ok → 200 `{ commandId, delivered, status, message }`
- Offline → **409**
- Ack timeout → **504**
- Payload > 4 KiB → **400**

Commands are ephemeral: not stored, not resumed, not inherited across TrackStart supersede.

## Presence

Listeners get `0x8B Presence` when a device session opens or the last session for that device closes. `last_seen` is also throttled (~5 s) while the device is publishing. `Subscribed.online` / `last_seen` is the snapshot at subscribe time.

## Related

- Byte layouts: [wire.md](wire.md)
- Drop / Resume / Staging: [reconnect.md](reconnect.md)
- GPS filter: [filter.md](filter.md)
