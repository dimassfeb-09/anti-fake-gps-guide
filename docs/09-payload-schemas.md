# 9. Payload Schemas (Exact Field Names)

Schemas below use the exact JSON keys traced in the reference implementation.
Types are JSON types. `Double`/`Float`/`Integer`/`Boolean` follow the
Gson annotations on the wire classes.

## 9.1 Per-fix item (V2 wire format — what the server receives)

One element of the `locations` array, per GPS fix:

```json
{
  "longitude": -6.123456,
  "latitude": 106.123456,
  "speed": 8.5,
  "heading": 174.0,
  "accuracy": 12.3,
  "provider": 2,
  "ongoing_orders": 1,
  "work_status": 3,
  "order_ids": ["123456789"],
  "client_time": "2026-09-06 07:55:12",
  "altitude": 8.0,
  "is_mockup": false
}
```

Notes:

- `provider` is an **integer code** on the wire (`gps` / `fused` / `network` /
  `passive` normalized at build time) — not the Android provider string.
- `heading` is the wire name; the internal model calls it `head`
  (from `Location.getBearing()`).
- `client_time` is a **string** (`"yyyy-MM-dd HH:mm:ss"`), while the internal
  model keeps `c_time` as epoch millis. Format on the client, parse on the
  server — and cross-check the two for teleport detection.
- `order_ids` may be empty when idle; `ongoing_orders` + `work_status` let the
  server correlate "driver claims to be delivering but hasn't moved in
  30 minutes".

## 9.2 Batch request (V2)

```json
{
  "device_info": "<opaque device string>",
  "locations": [ { "…fix…", "…fix…" } ],
  "app_state": 1,
  "user_agent": "<app user agent>"
}
```

The history variant has the same shape minus `app_state`:

```json
{
  "device_info": "<opaque device string>",
  "locations": [ { "…fix…" } ],
  "user_agent": "<app user agent>"
}
```

## 9.3 Internal model (V1 — richer, never sent raw)

The collector builds a richer object first; a converter strips it down to V2
before upload. Internal-only keys:

| Key | Type | Meaning |
|---|---|---|
| `c_time` | Long (epoch ms) | fix time, pre-format |
| `extra_info` | object | GPS health snapshot (see 9.4) |
| `app_version` | String | e.g. `"8.10.0"` — dropped on the wire |
| `driver_id` | String | dropped on the wire (identity comes from auth headers) |

Keeping debug fields internal and converting to a lean wire model means you
can evolve what you log without changing the API contract.

## 9.4 `extra_info` (GPS health snapshot, internal)

```json
"extra_info": {
  "is_enable_gps": true,
  "gps_accuracy": 12.3,
  "last_gps_client_time": "2026-09-06 07:55:12",
  "prev_update_status": {
    "is_success": true,
    "error_message": ""
  }
}
```

Server cross-checks (each is cheap, together they are strong):

- `is_enable_gps == false` while fixes keep arriving → spoof or stale cache.
- `gps_accuracy` vs top-level `accuracy` divergence → mixed-source spoof.
- `last_gps_client_time` vs `client_time` vs distance → teleport math.
- `prev_update_status.is_success == true` with jumping positions → fabricated
  history (a real radio that jumps that much would report errors).

## 9.5 Order correlation

Active jobs travel alongside fixes so the server can join position × task:

```json
{ "order_id": "123456789",
  "origin":      { "lat": -6.11, "lon": 106.11 },
  "destination": { "lat": -6.22, "lon": 106.22 } }
```

## 9.6 Response envelope

```http
POST /api/v1/location          HTTP/1.1
Accept: application/json
Content-Type: application/json
```

```http
POST /api/v1/location/history  HTTP/1.1
Accept: application/json
Content-Type: application/json
```

Both return a generic envelope (`BaseResponse<Object>`-shaped:
`{code, message, data}` pattern) — the client logs `"Request failed"` /
`"Response body is empty"` / `"Response data filed required nonNull but null"`
on the failure branches, which are useful grep anchors when testing
(see [testing](12-testing.md)).
