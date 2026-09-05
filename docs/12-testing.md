# 12. Testing Your Implementation

Verify each layer independently before trusting the score.

## 12.1 adb harness

```bash
# 1. Feed a fake fix stream (tests Layer 1 flag end-to-end)
adb shell settings put secure mock_location 1   # API ≤ 22 style
# API 23+: set a mock-location app in Developer Options, then:
adb shell appops set <your.pkg> android:mock_location allow

# 2. Teleport test (physics check)
#    send fix A, then 10 km away 5 s later — expect teleport points

# 3. Kill the UI process mid-report — history endpoint must backfill
adb shell am force-stop <your.pkg>

# 4. Proxy/VPN on — expect environment points
adb shell settings put global http_proxy <host>:<port>

# 5. Airplane mode + GPS-only fixes — expect anomaly points
adb shell settings put global airplane_mode_on 1

# 6. Confirm the override is closed in release builds
adb shell "run-as <your.pkg> cat shared_prefs/global.xml | grep url_mock" \
  && echo "LEAK: mock override shippable"
```

## 12.2 Per-layer test matrix

| Layer | Test | Expected |
|---|---|---|
| 1 — OS flag | mock app on, real GPS off | `is_mockup=true` on every fix |
| 2 — re-check | mock on for 60 s, then off; keep app running | stale mock still flagged via last-known for a window |
| 3 — sensors | `speed=0` + round `accuracy` for 10 min | synthetic-pattern points |
| 3 — teleport | 10 km jump in < 30 s | teleport points (see math below) |
| 4 — env | VPN/proxy/USB/airplane/emulator image | matching environment points |
| 5 — integrity | repacked APK (re-sign locally) | integrity verdict fails server-side |
| Pipeline | force-stop during upload | gap appears, then history backfill closes it |
| Pipeline | airplane 5 min, then reconnect | buffered fixes arrive via history endpoint, in order |

## 12.3 Teleport math (server snippet)

```python
from math import radians, sin, cos, asin, sqrt

def haversine_km(a, b):
    R = 6371.0
    la1, lo1, la2, lo2 = map(radians, (*a, *b))
    h = sin((la2 - la1) / 2) ** 2 + cos(la1) * cos(la2) * sin((lo2 - lo1) / 2) ** 2
    return 2 * R * asin(sqrt(h))

def teleport_points(prev, cur, max_speed_kmh=150.0):
    dt_h = (cur["t"] - prev["t"]) / 3600.0
    if dt_h <= 0:
        return 40  # non-monotonic clock is itself suspicious
    need = haversine_km((prev["lat"], prev["lon"]), (cur["lat"], cur["lon"])) / dt_h
    return 40 if need > max_speed_kmh else 0
```

Use 150 km/h as the generous ceiling for road vehicles (covers highway +
GPS noise); tighten per use-case (walking courier: 15 km/h).

## 12.4 Log fingerprints to grep

Client-side anchors worth asserting in your own logging:

- Router tags: `"History"` / `"Real-time"` per upload branch
- Daemon tags: `[processSendLocationCommand]`, `[processSendHistoryLocationCommand]`
- Failure branches: `"Request failed"`, `"Response body is empty"`,
  `"Response data filed required nonNull but null"`, `"Unknown Error"`,
  `"Network not connected"`, `"IPC report failed"`
- Buffer branch: `"insertAllWithLimit FAILED …"`

If a test scenario produces none of these, your logging — not just your
detection — has a hole.
