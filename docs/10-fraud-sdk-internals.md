# 10. Fraud-Detection Module Internals (Second-Layer Checks)

A hardened driver app ships a fraud-detection module that re-checks location
honesty **independently of the app layer**. Sensitive probe strings are
XOR-encoded (`hex → XOR with short key → UTF-8`); the table below lists the
decoded set, grouped by probe area.

## 10.1 Independent mock listener

Besides the app-layer `isFromMockProvider()` flag, the SDK registers its own
`LocationListener` and, on every fix, rounds `lat/lon` through `BigDecimal`
(scale normalization — defeats naive float-compare bypasses) before calling
`isFromMockProvider()` again on its own copy. It also walks
`getLastKnownLocation()` across **all providers**, so a mock that was enabled
and then switched off still leaves a stale trace in another provider.

Relevant decoded anchors in the mock-check method:

- `"mock_location"`
- `"android.permission.ACCESS_FINE_LOCATION"`
- `"android.permission.ACCESS_COARSE_LOCATION"`
- `"location"`

Lesson: the second check must live in a **separate module** and read
**separate sources** (its own listener + all providers' last-known), or one
patch kills both layers.

## 10.2 Full decoded probe table (30 methods)

| Area | Decoded strings (what is probed) |
|---|---|
| Device identity | `advertising_id`, `ro.serialno` (via `android.os.SystemProperties` + `get`), `ro.build.fingerprint`, `ro.debuggable` |
| OS / radio state | `airplane_mode_on`, `location` service state, `input_method` |
| Network | `vpn`, `http.proxyHost`, `http.proxyPort`, `connectivity` (+ `ACCESS_NETWORK_STATE`), interface regex `(tun\|ppp\|pptp)[0-9]*$` |
| USB / ADB | `usb` |
| Emulator | `/sys/devices/system/cpu/`, `android.app.ActivityThread` + `currentProcessName` + `activity` |
| Farm / cloud phones | `stfagent`, `stfservice` |
| Package install monitor | `android.intent.action.PACKAGE_ADDED`, literal `package:com.example.test` (placeholder pattern for watchlist matching) |
| OEM | `oppo` |
| Init services | `getprop`, `[init.svc.` pattern `^\[init.svc.(?=.*[A-Z])[A-Za-z0-9]+]: \[stopped]$` |
| Sockets | `/proc/net/unix`, pattern `@(?=.*[A-Z])(?=.*[a-z])(?=.*[0-9])[A-Za-z0-9]+$` (abstract-socket scan, tool-agent hint) |
| Files | `stat ` output parsed for `Inode:` / `Links:` (tamper check on watched paths) |
| Permissions re-check | `checkSelfPermission` via `android.content.Context`; `ACCESS_FINE/COARSE_LOCATION`, `ACCESS_WIFI_STATE`, `READ_PHONE_STATE`; `camera`; `wifi` |
| Encoding / errors | `GBK`, `UTF-8`, `Permission denied`, `%.1f`, `S44,`, `SPSPDT_APPID` (vendor app-id tag) |

Notably **absent**: no `frida` / `xposed` / `magisk` / `su` string literals in
this SDK — root/tool detection here is behavioral (init services, sockets,
inodes, debuggable flag) rather than package-name blocklists, which are
trivially renamed. Prefer the same: check **properties**, not package names.

## 10.3 What to copy

1. **Separate listener, separate verdict.** Your fraud module must consume raw
   fixes itself — never trust the app layer's boolean.
2. **Normalize before comparing.** Round coordinates (`BigDecimal.setScale`)
   so float-representation tricks don't split your clustering.
3. **Probe properties, not names.** `ro.debuggable`, init-svc states, proxy
   settings, CPU info, and socket tables survive renames; package blocklists
   don't.
4. **Keep the string table encoded.** Even a one-byte XOR defeats casual
   `strings` triage and forces an attacker to trace code, not grep. Serious
   keys belong in native code, not in app bytecode at all.
5. **Report, don't enforce, on-device.** The SDK's job ends at producing a
   second verdict bit plus the environment snapshot — the score lives on the
   server ([scoring](06-server-side-scoring.md)).
