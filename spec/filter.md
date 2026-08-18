# Device GPS filter (SDK, not on the wire)

The server does not filter. It caps ingest at 50 Hz as defense. Heading and speed are **not** sent; the server (and listeners) derive motion from consecutive **accepted** points.

This filter runs on the **device SDK** before Staging / seq / `Loc`.

## When it is on

| Incoming GPS rate | Behaviour |
|-------------------|-----------|
| ~1 Hz | Pass-through (still honour heartbeat / first point). |
| ≳ 5 Hz | Filter on. 50 Hz on a straight road should become ~1 Hz heartbeat plus vertices on turns. |

## State

- `last_emitted` — last point actually given to Staging / send
- `candidate` — latest sample held back as “maybe collinear junk”
- `last_emit_at` — time of `last_emitted`

Local **GPS heading** (chip) is used only inside the filter (turn detector). It is not written to the frame.

## Emit a point when any of these hold

1. **First point** of the track.
2. **Heartbeat:** `now - last_emit_at >= 1000 ms` — emit **current** position (not a stale candidate), even if the device has not moved.
3. **Moved enough:** haversine(`last_emitted`, current) ≥ `max(2 m, 2 × accuracy)`.
4. **Collinear break:** perpendicular distance from `candidate` to the line `last_emitted → current` ≥ `ε`, where  
   `ε = max(2 m, accuracy, 0.5 s × speed)`.
5. **Stop / start** of motion, or **heading jump ≥ 25°** (local GPS heading).

Otherwise keep `candidate` and do not emit.

Accuracy missing: treat as 0 for the `max(...)` formulas (so the 2 m floor applies).

## After emit

`last_emitted = emitted`, clear `candidate`, `last_emit_at = now`. The emitted point then goes to Staging (offline) or gets a seq and `Loc` (online). See [reconnect.md](reconnect.md).

## Fast GPS (up to 50 Hz)

No extra transport:

1. Filter turns 50 Hz straight-line into ~1 Hz.
2. Seq is assigned **after** the filter, so raw samples never burn the cursor.
3. If the filter still emits a burst (hairpin), coalesce 20–50 ms into one `Loc count=N` with intra-frame Δ.
4. Staging flush uses the same `Loc count ≤ 100`.
5. Server 50 Hz cap is not the target over-the-air rate.

## What the server still does

Even a buggy SDK cannot exceed 50 Hz fan-out / ingest per track. Extra points are acked (seq advances) but may not be forwarded to listeners (`fanout=false`).
