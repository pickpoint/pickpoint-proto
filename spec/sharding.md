# Relocate

Clients do not pick a shard. The server may close a device session with `0x81 Relocate` if this node is not that device’s home (or is draining).

```
retry_ms      u32
endpoint      string   wss://…/v2/ws
```

The SDK:

1. Closes the current socket.
2. Waits `retry_ms`.
3. Dials `endpoint` with the **same** auth and subprotocol `tracking.v2`.
4. Waits for `Hello`.
5. Sends **Resume** if it still has a `track_uid` (device), or **Subscribe** again (listener).

Never `TrackStart` only because of Relocate — that would start a new trip. Details: [reconnect.md](reconnect.md#7-relocate).
