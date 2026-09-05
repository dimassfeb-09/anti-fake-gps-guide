# 7. Limitations

Be honest about what detection can't do.

- **No client-side check is 100%**. A hardware GPS simulator (SDR, u-blox)
  feeds real-looking GPS RF and is almost invisible on-device. Only
  server-side statistical analysis of long trajectories has a chance.

- **Every check has a cost**: battery (polling), permissions, and false
  positives. Typical causes of false positives: high-rise urban canyons,
  cheap phones with weak GPS chips, sudden GPS drift.

- **Privacy**: request only what you need, explain why, and avoid shipping
  raw device dumps. Aggregate and hash where possible.

- **Operational rule**: the client *reports*, the *server* decides, and a
  **human reviews** heavy sanctions. Automate friction (extra checks), not
  final punishment.

- **Maintainability**: treat this as a living control — retune thresholds
  from real incident data, and log everything so you can learn which signals
  actually predicted abuse in your population. What works for one city or
  device fleet may be noisy for another.
