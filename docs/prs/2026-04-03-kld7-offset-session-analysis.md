# K-LD7 Offset Session Analysis

## Summary

This PR does not change the detector math.

It documents analysis of the new April 3 session and raw K-LD7 captures after
the fixed `angle_offset_deg` mounting correction was added.

The main conclusion is:

> The `+13°` offset looks like a real, stable correction for the current mount,
> and it moves the April 3 launch angles into a much more believable range.

That said, the raw K-LD7 data still looks like it is locking onto a longer
post-impact / net-plane burst rather than a pure 1-3 frame ball-only return.

So the offset improves the output materially, but it does not yet prove that the
underlying burst selection problem is fully solved.

## What Was Analyzed

- `session_logs/session_20260403_133805_range.jsonl`
- `session_logs/kld7_capture_20260403_125012-7i.pkl`
- `session_logs/kld7_capture_20260403_125230-8i.pkl`
- `session_logs/kld7_capture_20260403_125603-8i.pkl`

All analysis in this PR was done against current `main`, including the merged
K-LD7 offset support.

## Main Findings

### 1. The `+13°` offset is consistent

Replaying the 10 logged K-LD7 buffers from `session_20260403_133805_range.jsonl`
shows:

- mean raw vertical angle without offset: `4.1°`
- mean vertical angle with `+13°` offset: `17.1°`

The logged `shot_detected.launch_angle_vertical` values match the `+13°`
replay exactly, shot-for-shot.

Examples:

- Shot 1: raw `-7.2°` -> offset `5.8°`
- Shot 2: raw `12.2°` -> offset `25.2°`
- Shot 6: raw `9.1°` -> offset `22.1°`
- Shot 8: raw `6.8°` -> offset `19.8°`

That is strong evidence that the current setup has a stable mounting bias rather
than random per-shot angle drift.

### 2. The new session looks much better than the older backyard driver session

The April 3 session contains:

- `10` accepted shots
- `10` `kld7_buffer` entries
- no negative final launch angles
- all `10` K-LD7 vertical angles accepted by the current sanity guard

Club-by-club summary:

- `7-iron`: `104.6-115.7 mph` ball speed, `5.8-25.2°` launch, `136.9-174.8 yds` carry
- `8-iron`: `106.8-112.6 mph` ball speed, `14.6-19.8°` launch, `139.3-158.2 yds` carry

Those numbers are not perfect ground truth, but they are broadly believable and
far more reasonable than the earlier session with negative driver launches.

### 3. The offset improves the output more than the raw detector itself

The three new raw `.pkl` captures still show the same pattern the older files
did: without an offset, the paired ball angles are often too low or simply odd.

Current offline pairings:

- `kld7_capture_20260403_125012-7i.pkl`: `11.9°, -11.9°, 9.4°`
- `kld7_capture_20260403_125230-8i.pkl`: `-1.0°, 8.3°, 4.3°, 28.7°`
- `kld7_capture_20260403_125603-8i.pkl`: `17.2°, -23.0°, 38.6°`

That is consistent with the live-session replay:

- the raw angles are still often low
- the fixed offset shifts them into the expected club range

So the new data supports the mounting-offset theory more than it supports a
claim that raw K-LD7 selection is fully calibrated.

### 4. The selected K-LD7 "ball" bursts are still long and tightly clustered in distance

Across the 10 April 3 live session shots, the selected K-LD7 ball bursts were:

- distance: `4.15-4.46 m`
- frames: `6-11`
- mean distance: `4.29 m`
- mean burst length: `7.5` frames

That is important because a true golf-ball-only return at this frame rate should
usually be much shorter.

Interpretation:

- the offset likely corrected a real mount geometry issue
- but the detector still appears to be riding a stable impact / net-plane burst
  rather than isolating only the earliest ball-flight frames

This is not necessarily bad for current launch-angle usefulness, but it is a
real caveat for anyone treating the K-LD7 output as strict ball-only truth.

## Two Logging Gaps Worth Fixing Later

These are not part of this PR, but the new session made them obvious:

1. The applied K-LD7 angle offset is not written into `session_start`, so later
   analysis has to infer it from the outputs.
2. `shot_detected` does not persist `angle_source`, even though the runtime shot
   object sets it to `"radar"` for these K-LD7 launches.

Those would make future calibration work easier and less ambiguous.

## Validation

Commands run:

```bash
PYTHONPATH=src .venv/bin/pytest -q
PYTHONPATH=src .venv/bin/pytest tests/test_kld7.py tests/test_server.py -q
PYTHONPATH=src .venv/bin/python scripts/analyze_kld7.py session_logs/kld7_capture_20260403_125012-7i.pkl --pair-shots
PYTHONPATH=src .venv/bin/python scripts/analyze_kld7.py session_logs/kld7_capture_20260403_125230-8i.pkl --pair-shots
PYTHONPATH=src .venv/bin/python scripts/analyze_kld7.py session_logs/kld7_capture_20260403_125603-8i.pkl --pair-shots
```

Results:

- `259 passed, 2 skipped`
- `72 passed, 2 skipped` for `tests/test_kld7.py` and `tests/test_server.py`

## Ask

The next useful step is more data, not more theory.

Most valuable follow-up captures:

1. More sessions across additional clubs, not just `7i` and `8i`
2. Another session after the mount is made more rigid
3. A few shots with known reference launch angles from a commercial unit
4. Logs that explicitly include the applied K-LD7 angle offset

That would tell us whether the current good-looking numbers are:

- a stable mount-corrected measurement
- or a mount-corrected approximation riding the net-plane burst

Either way, the April 3 session is a meaningful improvement and worth building on.
