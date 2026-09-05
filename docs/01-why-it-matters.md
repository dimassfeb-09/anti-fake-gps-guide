# 1. Why Fake GPS Matters

Android ships an official mock-location API (`addTestProvider()`, mock-location
apps) meant for developers testing location features. The same API is reused
for abuse:

- **Ride-hailing / delivery drivers** pinning themselves next to busy
  restaurants so dispatch prioritizes them.
- **Fake attendance** — clocking in from home on a GPS-based check-in app.
- **Fake delivery proof** — marking orders delivered without going there.
- **Location-based promo farming** — claiming region-locked rewards.

## Why one flag is never enough

`Location.isFromMockProvider()` exists and works — but it is a single boolean
produced on-device, where the attacker has full control (root, hooks, repacks).
Every bypass closes one hole and opens a trace somewhere else.

The robust posture is therefore:

1. Collect **many independent signals** (flags, sensors, environment, history).
2. Send them all to the server on every report.
3. Let the **server score** honesty instead of trusting any single client flag.
4. Keep a **human review** before heavy sanctions (false positives are real:
   tall buildings, cheap phones, GPS drift).
