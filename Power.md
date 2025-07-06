* **Boot Settings**:

  * UEFI configured to **Power On** automatically when AC power returns (BIOS: "AC Power Loss" → "Power On").
  * `/etc/systemd/logind.conf`: set `HandleLidSwitch=ignore` and `HandleLidSwitchDocked=ignore` to keep the system running with the lid closed.

## 2. Power Management

* **Auto‑Power‑On After Outage**:

  1. Mains out → laptop runs on battery → battery dies → machine powers off.
  2. Mains returns → BIOS automatically powers on.
* **Graceful Low‑Battery Shutdown**:

  * `/etc/UPower/UPower.conf`:

    ```ini
    PercentageLow=10
    PercentageCritical=3
    PercentageAction=2
    CriticalPowerAction=PowerOff
    ```
  * UPower warns at 10% and initiates a clean shutdown at the action threshold.