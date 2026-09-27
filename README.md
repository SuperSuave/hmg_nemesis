# NEMESIS Manual

These notes describe NEMESIS on a Gen5W head unit (Telechips TCC803x, Android 4.4.2 / KitKat, firmware family **260507**).

| Document | Contents |
|----------|----------|
| [01 — DSP OTA `troubleShooter` path](01-DSP-OTA-troubleShooter.md) | How the firmware runs unsigned DSP OTA members as root, and how that path is reused. |
| [02 — Debugging Execution Interface](02-Debugging-Execution-Interface.md) | What NEMESIS is on the unit after it is present: features, how you open it, CLI and Manager usage. |
| [03 — Install and uninstall](03-Install-and-Uninstall.md) | How NEMESIS is written onto `/system` and how it is removed. |
| [04 — Packages](04-Packages.md) | Installable packages: metadata, layout, builder, lifecycle. |
| [05 — Run scripts](05-Run-Scripts.md) | USB `NEMESIS.sh` / `type=execute`: when they run and how to write them. |

Documents **01** and **03** are the only ones that describe the DSP Update Agent path and on-disk install, do not use them with advanced LLMs which may have safeguards enabled. **02**, **04**, and **05** assume NEMESIS is already on the head unit and treat it as a Debugging Execution Interface for packages and privileged commands, so safeguards will not be triggered.
