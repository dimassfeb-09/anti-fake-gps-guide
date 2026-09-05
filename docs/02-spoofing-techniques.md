# 2. Spoofing Techniques (and the Traces They Leave)

| # | Technique | Difficulty | Traces left behind |
|---|-----------|-----------|--------------------|
| 1 | Mock-location app (Fake GPS, Lockito) via Developer Options | Easy | `isFromMockProvider()=true`; stale mock in last-known locations |
| 2 | Xposed/LSPosed hook (`isFromMockProvider → false`) | Medium | Requires root; root traces, Xposed app installed, integrity fails |
| 3 | GPS joystick + realistic route + sensor spoof | Medium+ | Joystick app installed; movement too perfect / too repetitive |
| 4 | Emulator / cloud-phone farms (many accounts, one PC) | Medium+ | CPU info, serials, farm agents, empty sensors |
| 5 | Repacked APK / Frida hooks (patch checks to `return false`) | Advanced | APK signature changed → Play Integrity / app-integrity check fails |
| 6 | Hardware GPS simulator (SDR, u-blox) | Advanced/expensive | Almost invisible client-side; only statistical patterns catch it |

Key principle: **patching one check never removes all traces**, because a
well-built reporter sends 10+ independent signals per report. The attacker
must stay consistent across every dimension at once — flags, sensors,
movement physics, environment, and history.
