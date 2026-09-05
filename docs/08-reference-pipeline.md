# 8. Reference Pipeline (Traced End-to-End)

This is the complete location-reporting data flow as traced in a production
driver app, layer by layer. Class names are generalized; constants, intervals,
and schemas are the real traced values. Use it as the reference architecture
for [the builder's guide](04-defense-in-depth.md).

```
┌─ React Native ──────────────────────────────┐
│ DriverLocationModule  → map deeplinks only  │  (openGoogleMap / openMaps)
│ GALocation → getLatestLocation / getLocation│
│            / watchPosition / clearWatch     │
└──────────────────────┬──────────────────────┘
                       │ Promise bridge
┌─ Native monitor ─────▼──────────────────────┐
│ LocationMonitorService                      │
│   strategy = TIME_STRATEGY                  │  periodic fixes
│            | ORDER_TYPE_STRATEGY            │  fixes tied to job state
│   (selected by an int flag at runtime)      │
└──────────────────────┬──────────────────────┘
                       │
┌─ Provider racing ────▼──────────────────────┐
│ ProviderRacingEngine                        │
│   races gps / network / fused / passive,    │
│   picks winner per fix                      │
│   FUSED_INTERVENE event re-armed via        │
│   Handler.sendEmptyMessageDelayed(…, 20000) │  0x4e20 = 20 000 ms = 20 s
└──────────────────────┬──────────────────────┘
                       │ winning android.location.Location
┌─ Model builder ──────▼──────────────────────┐
│ LocationUtils                               │
│   Location.isFromMockProvider()  → Boolean  │  app-layer mock flag
│   Settings.Secure "location_mode" read      │  (2 call sites)
│   → LocationInfo{…, is_mockup, extra_info}  │
└──────────────────────┬──────────────────────┘
                       │ List<LocationInfo> + OrderInfo
┌─ Business / router ──▼──────────────────────┐
│ LocationBusiness                            │
│   builds V1 LocationRequest /               │
│          HistoryLocationRequest             │
│   (embeds app version string, e.g. 8.10.0)  │
│ V2LocationUtils converts V1 → V2            │
│   (drops app_version / driver_id /          │
│    extra_info — leaner wire format)         │
│ LocationReportRouter routes:                │
│   "History"   → POST /api/v1/location/history
│   "Real-time" → POST /api/v1/location       │
└──────────────────────┬──────────────────────┘
                       │ Retrofit (Accept: application/json)
┌─ Transport ──────────▼──────────────────────┐
│ Two redundant paths:                        │
│  A. Direct Retrofit from the app process    │
│     (base URL from a config holder; a       │
│      SharedPreferences "url_mock" override  │
│      exists for test environments)          │
│  B. IPC to the push process:                │
│     LocationIPCReporterClient → Bundle      │
│     {param_location_request_id,              │
│      param_location_json} →                  │
│     PushMessageDaemonService →              │
│     ILocationReporter.reportLocation() /    │
│     reportHistoryLocation() → Retrofit      │
│     (log tags: [processSendLocationCommand] │
│      / [processSendHistoryLocationCommand]) │
│ Push channel itself: OkHttp WebSocket,      │
│ heartbeat 20 s, connect timeout 5 s,        │
│ reconnect 15 s, peak/non-peak ping tuning   │
└─────────────────────────────────────────────┘
```

## Key observations for builders

1. **Location entry is native, not JS.** The RN bridge only exposes
   one-shot/watch reads and map links; continuous reporting lives in the
   native service. Keep your reporting loop out of the JS thread.
2. **Two strategies, one switch.** Time-based reporting (always on while
   online) vs job-state reporting (denser during an active job). A single int
   flag selects the collector — this lets the server tune battery vs
   freshness per cohort.
3. **Racing, not single-provider.** Subscribing to several providers and
   picking the winner per fix beats trusting `fused` alone — and gives you
   the per-provider last-known readings the fraud check needs (Layer 2).
4. **V1 builds, V2 sends.** Keep a rich internal model (with debug fields
   like `extra_info`), then convert to a lean wire model. The server never
   sees your debug scaffolding, and you can evolve the wire format without
   touching collectors.
5. **Two transport paths.** Direct HTTP plus an IPC fallback into the
   long-lived push process means reports survive activity kills and short
   network gaps. Buffer locally, flush on reconnect.

See [payload schemas](09-payload-schemas.md) for the exact JSON, and
[transport & resilience](11-transport-and-resilience.md) for the endpoint,
IPC, and heartbeat details.
