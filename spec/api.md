# Tracking API

Host: **`tracking.pickpoint.io`**. Live session is a WebSocket; the frame layout is [wire.md](wire.md). Track history is REST on public-api, not this host.

| | |
|--|--|
| `wss://tracking.pickpoint.io/v2/ws` | Device or listener session. Subprotocol `tracking.v2`. |
| `POST /v2/devices/{uid}/command` | Opaque command to an online device (same host). |

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

After `Hello`, `Subscribe` per device. The listener gets a `Subscribed` snapshot plus live `Loc` / `EventAdded` / `Presence` / track start-stop.

A listener does not Resume a track. After a drop: new socket → `Hello` → `Subscribe` again. Missed live points are not replayed; use HTTP history on public-api.

## Handshake

1. Upgrade with `Sec-WebSocket-Protocol: tracking.v2`. Wrong or missing → HTTP 400.
2. Query auth. Missing or bad → HTTP 401.
3. Wrong home shard (device) → `0x81 Relocate` then close ([reconnect.md](reconnect.md#7-relocate)).
4. First binary frame is always `0x80 Hello` with `version = 2`. Unknown version → client closes.
5. Do not send application frames before `Hello`.

Keepalive is WebSocket ping/pong. One binary WS message = one protocol frame. Text → close 1002.

## Limits

| | |
|--|--|
| Ingest `Loc` | 50 Hz per track (defense; a correct SDK is ~1 Hz after the filter) |
| `Loc.count` | 1…100 |
| Event | 4 KiB, 1 Hz per track (extras dropped) |
| Track metadata | 4 KiB |
| Command | 4 KiB |
| String / bytes fields | `u16` length, max 4096 |
| Staging / InFlight (SDK) | 10_000 points ([reconnect.md](reconnect.md)) |
| Listener subscriptions | 255 per session (`sub` = 1…255) |

## Command

`POST https://tracking.pickpoint.io/v2/devices/{uid}/command`

- Body: raw bytes or JSON `{ "payload": "<base64>" }`
- Server pushes `0x8A Command` on every online device session for that uid
- Waits ~5 s for `0x08 CommandAck`
- 200 `{ commandId, delivered, status, message }` · offline **409** · ack timeout **504** · too large **400**

Commands are not stored, not resumed, not inherited across TrackStart supersede. `Authorization: Bearer` is accepted here; the WebSocket uses the query token.

## Presence

`0x8B Presence` when a device session opens or the last one closes. `last_seen` is throttled (~5 s) while publishing. `Subscribed.online` / `last_seen` is the snapshot at subscribe time.

## Related

[wire.md](wire.md) · [reconnect.md](reconnect.md) · [filter.md](filter.md)
