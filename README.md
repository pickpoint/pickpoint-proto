# pickpoint-proto

Canonical **tracking.v2** protocol: binary WebSocket at `wss://tracking.pickpoint.io/v2/ws`.

HTTP geocoding / routing / search are separate JSON APIs and are not specified here.

## Read this in order

| Doc | What it answers |
|-----|-----------------|
| [spec/api.md](spec/api.md) | URL, auth, device vs listener, limits, HTTP command inject |
| [spec/wire.md](spec/wire.md) | Every byte on the socket: types, frames, examples |
| [spec/reconnect.md](spec/reconnect.md) | Session start, socket drop, Resume vs TrackStart |
| [spec/filter.md](spec/filter.md) | Device GPS noise filter (SDK-side) |
| [spec/goldens.md](spec/goldens.md) | Canonical hex frames for SDK codecs |
| [spec/sharding.md](spec/sharding.md) | `Relocate` when the node is not the device’s home |

An SDK implements **api + wire + reconnect + filter**. Relocate is the only cluster behaviour a client must handle.

## Versioning

- WebSocket subprotocol: `tracking.v2`
- First server frame `Hello.version`: `2`
- A new **server→client** frame type does not bump the version (unknown server types are ignored).
- A breaking layout is a new subprotocol (`tracking.v3`).
