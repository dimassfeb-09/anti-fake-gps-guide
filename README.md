# Anti Fake-GPS Guide (for Developers)

How modern delivery-driver apps keep location reports honest — and how to
implement the same in your own app.

> Generic educational documentation — common industry patterns (mock-provider
> flags, sensor cross-checks, device fingerprinting, server-side scoring).
> Not a guide to tampering with any third-party app.

## Docs

| # | Doc | What it covers |
|---|-----|----------------|
| 1 | [Why fake GPS matters](docs/01-why-it-matters.md) | The problem, the cost, why one flag is never enough |
| 2 | [Spoofing techniques](docs/02-spoofing-techniques.md) | 6 ways people fake location + the traces each leaves |
| 3 | [Use cases](docs/03-legitimate-usecases.md) | Legit apps that need honest location |
| 4 | [Defense in depth](docs/04-defense-in-depth.md) | 5 layers: OS flag → independent check → sensors → fingerprint → integrity |
| 5 | [Android implementation](docs/05-android-implementation.md) | Report schema, Kotlin snippets, Play Integrity |
| 6 | [Server-side scoring](docs/06-server-side-scoring.md) | Weighted scoring model + enforcement thresholds |
| 7 | [Limitations](docs/07-limitations.md) | What detection can't do, false positives, privacy |

**The one-line summary:** report from the client, decide on the server,
always keep a human in the loop for heavy sanctions.
