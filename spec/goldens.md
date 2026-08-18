# tracking.v2 golden frames

Little-endian. These bytes are the shared fixture for every SDK. A codec that does not match them is wrong.

UUID used below: `00112233-4455-6677-8899-aabbccddeeff` →
`00 11 22 33 44 55 66 77 88 99 aa bb cc dd ee ff`.

Empty / none UUID is 16 zero bytes.

## Ack (`0x85`) seq = 1

```
85 01 00 00 00
```

## Loc (`0x04`) one live point

`seq=1`, `count=1`, `55°N 37°E`, no altitude / accuracy / time:

```
04 01 00 00 00 01 00 c0 3b 47 03 40 93 34 02
```

`round(55 × 1e6) = 55_000_000 = 0x03473BC0` LE `c0 3b 47 03`.
`round(37 × 1e6) = 37_000_000 = 0x02349340` LE `40 93 34 02`.

## Resume (`0x01`) last_seq = 45

```
01 00 11 22 33 44 55 66 77 88 99 aa bb cc dd ee ff 2d 00 00 00
```

## TrackStop (`0x03`)

Empty body (active track):

```
03
```

## Intra-frame Δ

Consecutive points whose μ° delta does not fit in `i16` (±32767) **must** be separate Loc frames. First point of each frame is always absolute `i32`. Do not wrap.

See [wire.md](wire.md) and [reconnect.md](reconnect.md).
