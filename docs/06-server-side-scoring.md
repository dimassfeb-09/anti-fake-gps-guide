# 6. Server-Side Scoring

Never ban on one signal. Use a weighted score (0–100) per session or per time
window.

| Signal | Points |
|---|---|
| `is_mockup = true` on either layer | +60 |
| Teleport (distance / time impossible) | +40 |
| Synthetic `speed`/`accuracy`/`altitude` pattern | +20 |
| GPS disabled / VPN / proxy / USB-ADB attached | +10 each |
| Emulator / farm agent present | +50 |
| Known mock-related package installed (plus install timing) | +25 |
| Repetitive pattern (same point & hour every day) | +15 |

Suggested enforcement:

- **Score ≥ 70** → hold incentives, require re-verification (photo + liveness,
  or supervised check-in).
- **Score ≥ 90 or repeated high scores** → suspend after **human review**.
- **Always** require a human reviewer before any permanent action — GPS has
  real false positives.

Persist per-device history so you can see trends, not just snapshots. A single
noisy fix is expected; a perfect, identical point every lunch hour is not.
