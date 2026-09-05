# 4. Defense in Depth (5 Layers)

Report from the client, decide on the server. Every report should carry all of
these — the server correlates them; no single flag is trusted alone.

## Layer 1 — OS mock flag (cheap, mandatory)

```kotlin
val isMock = location.isFromMockProvider // API 18+
```

Send as `is_mockup` on every report. Cheap and mandatory, but trivial to hook —
never the sole decision.

## Layer 2 — Independent re-check (separate module)

Don't trust one code path. In a separate module:

```kotlin
// stale mock survives in last-known locations even after mock is off
fun anyProviderMocked(mgr: LocationManager): Boolean =
    mgr.getProviders(true).any { name ->
        mgr.getLastKnownLocation(name)?.isFromMockProvider == true
    }
```

If the app-layer check is patched, the module-layer check still reports mock.
Putting the second check in a different module makes one-patch bypasses
insufficient.

## Layer 3 — Sensor consistency (client → server)

A good report payload:

```json
{
  "latitude": -6.2, "longitude": 106.8,
  "accuracy": 12.3, "altitude": 8.0,
  "speed": 8.5, "head": 174.0,
  "provider": "fused", "c_time": 1710000000,
  "is_mockup": false,
  "extra_info": {
    "is_enable_gps": true,
    "gps_accuracy": 12.3,
    "last_gps_client_time": "2026-09-06 07:55:12",
    "prev_update_status": { "is_success": true, "error_message": "" }
  }
}
```

Server cross-checks:

- `speed == 0` forever + perfectly round `accuracy` + `altitude == 0` → classic Fake-GPS signature
- `gps_accuracy` vs `accuracy` divergence → spoof
- GPS disabled in settings but locations flowing → odd
- `last_gps_client_time` vs distance → teleport (10 km in 5 s = impossible)
- `prev_update_status` always success while positions jump → fake history

## Layer 4 — Environment fingerprint (per session, not per second)

Sample once per session to save battery, send with reports:

- `airplane_mode_on` — airplane on but GPS moving
- VPN / `http.proxyHost` / `http.proxyPort`
- USB connected (ADB spoofing hint)
- CPU info (`/sys/devices/system/cpu/`) — emulator hint
- Device IDs (`advertising_id` + `ro.serialno`) — consistency check
- Farm agents (`stfagent` etc.) — cloud-phone farm
- `PACKAGE_ADDED` — new app installed (timing of Fake-GPS install)
- Active `input_method` — bot / auto-clicker
- Location permission state (FINE vs COARSE)
- `work_status` + `ongoing_orders`/`order_ids` — position vs task correlation

## Layer 5 — App integrity (per session / login)

- Play Integrity API / SafetyNet — verify the APK is official, not repacked,
  device not heavily rooted, not an emulator.
- Obfuscate sensitive strings (even a simple XOR defeats `strings`; serious
  keys belong in native `.so`).
- Never make the final decision on-device — the client only reports.
