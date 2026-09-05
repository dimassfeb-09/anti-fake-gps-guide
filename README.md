# Anti Fake-GPS Guide (for Developers)

How modern delivery-driver apps keep location reports honest — traced from a
production implementation — and how to build the same in your own app.

> Generic educational documentation. No brand names, no proprietary code —
> only the architecture patterns, payload schemas, constants, and checklists
> distilled from tracing a real driver app end-to-end (location pipeline,
> fraud SDK, transport). Use it to defend your own app.

## Docs

### Concepts

| # | Doc | What it covers |
|---|-----|----------------|
| 1 | [Why fake GPS matters](docs/01-why-it-matters.md) | The problem, the cost, why one flag is never enough |
| 2 | [Spoofing techniques](docs/02-spoofing-techniques.md) | 6 ways people fake location + the traces each leaves |
| 3 | [Use cases](docs/03-legitimate-usecases.md) | Legit apps that need honest location |

### Deep trace (from a production driver app)

| # | Doc | What it covers |
|---|-----|----------------|
| 8 | [Reference pipeline](docs/08-reference-pipeline.md) | End-to-end data flow: RN bridge → monitor service → provider racing → model → router → IPC → server, with real constants (20 s intervene, dual strategy) |
| 9 | [Payload schemas](docs/09-payload-schemas.md) | Exact JSON schemas: per-fix item, batch request, history request, order correlation, response envelope |
| 10 | [Fraud-SDK internals](docs/10-fraud-sdk-internals.md) | Full decoded check table (30 probes), independent mock listener, emulator/root/farm detection strings |
| 11 | [Transport & resilience](docs/11-transport-and-resilience.md) | REST endpoints, IPC to the push process, WSS heartbeat/reconnect tuning, offline buffering |

### Build it yourself

| # | Doc | What it covers |
|---|-----|----------------|
| 4 | [Defense in depth](docs/04-defense-in-depth.md) | 5 layers distilled into build guidance |
| 5 | [Android implementation](docs/05-android-implementation.md) | Report builder, provider racing, router, IPC reporter, Play Integrity — copy-ready Kotlin |
| 6 | [Server-side scoring](docs/06-server-side-scoring.md) | Weighted scoring model + enforcement thresholds + teleport math |
| 7 | [Limitations](docs/07-limitations.md) | What detection can't do, false positives, privacy |
| 12 | [Testing your implementation](docs/12-testing.md) | adb commands, per-layer test matrix, log fingerprints to grep |

**The one-line summary:** report from the client, decide on the server,
always keep a human in the loop for heavy sanctions.
