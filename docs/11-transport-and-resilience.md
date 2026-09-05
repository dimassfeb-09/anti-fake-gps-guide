# 11. Transport & Resilience (Endpoints, IPC, Heartbeats)

How reports get to the server without loss — endpoints, fallback paths,
and channel tuning that works in production.

## 11.1 REST endpoints

| Purpose | Method | Path | Body | Response |
|---|---|---|---|---|
| Real-time batch | POST | `/api/v1/location` | `LocationRequestV2` (§9.2) | generic envelope |
| History backfill | POST | `/api/v1/location/history` | `HistoryLocationRequestV2` (§9.2) | generic envelope |

- Header: `Accept: application/json`, `Content-Type: application/json`.
- Auth rides on the shared network stack (token interceptor), not in the body —
  note the wire model drops `driver_id` for exactly this reason.
- Base URL comes from a central config holder. A `SharedPreferences` key
  `url_mock` can override it — a test-environment escape hatch you should
  replicate (and gate behind your own build flags, never ship it open).

## 11.2 IPC fallback into the push process

The reporting client exists twice:

1. **In-process Retrofit** (normal path, same process as the UI/service).
2. **IPC reporter client** (`LocationIPCReporterClient`) that ships the already-
   serialized JSON to the long-lived push process:

```
Bundle { param_location_request_id, param_location_json }
  → PushMessageDaemonService
  → ILocationReporter.reportLocation(json, callback)
  → ILocationReporter.reportHistoryLocation(json, callback)
```

Log anchors: `[processSendLocationCommand]`, `[processSendHistoryLocationCommand]`,
`"locationJson is null or empty"`, `"locationReporter is null"`.

Why two paths: the UI process can be killed while the push daemon survives.
Location generated just before a kill is forwarded over IPC instead of lost.

## 11.3 Push channel tuning (field-tested values)

The daemon holds its own WebSocket (OkHttp) with server-driven tuning:

| Setting | Value |
|---|---|
| Heartbeat interval | `0x4e20` = **20 000 ms = 20 s** |
| Connect timeout | `0x5` = **5 s** |
| Reconnect interval | `0x3a98` = **15 000 ms = 15 s** |
| Ping policy | per-profile: `PingSettings{maxFailureCount, pingIntervalSec}`, with separate `NonPeakSettings` and `PeakHourConfig{start, end, pingSettings}` plus a `greyscale` rollout percent |

Copy the shape: heartbeat + reconnect + peak/non-peak ping profiles, all
remotely tunable (`WssConfig{peak_hours, non_peak_settings, greyscale}`),
so you can trade battery vs freshness per cohort without shipping an update.

## 11.4 Offline buffering

The business layer funnels through suspend functions named like
`reportLocation() >>> insertAllWithLimit FAILED` on the failure branch —
i.e. fixes are **inserted into a local store with a cap** first, then
uploaded. On reconnect, the history endpoint backfills the gap.

Builder checklist:

- [ ] Cap the buffer (e.g. last N fixes or M hours) — unbounded queues OOM.
- [ ] Upload order: oldest first, idempotent server-side (dedupe by
      `driver + client_time`).
- [ ] Separate real-time vs history endpoints so backfill never blocks live fixes.
- [ ] Log every failure branch distinctly (`Request failed` vs
      `Response body is empty` vs `data … null`) — see testing.
