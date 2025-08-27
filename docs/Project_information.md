# Project Brief — Mini IVI

**Owner:** Huy Van Le  
**Repository:** `Project_Mini_IVI`  
**Created:** 8/2025  
**Document version:** v1.0

---

## 1) Elevator Pitch
Build a **mini IVI** on PC/WSL2 with the flow: **ECU Simulator (CSV replay)** → **VehicleService (QtDBus daemon)** → **HMI (Qt/QML)**.  
Focus: the **HMI updates in real time** when ECU data changes, via **D-Bus** (speed, RPM, warning).

---

## 2) Goals
- **Real-time HMI:** HMI (Qt/QML) subscribes to D-Bus signals and **updates UI immediately** (speed gauge, RPM gauge, warning indicator).
- **D-Bus middleware:** Expose methods/properties/signals: `GetSpeed`, `GetRpm`, `GetWarnings`, `Health`, and `SpeedChanged`, `RpmChanged`, `WarningChanged`.
- **Full V-model docs:** SRS, AD, UD, Test Specific, Traceability with clear acceptance criteria.
---

## 3) Output (v1)
- **Runnable code:**
  - `apps/ecu_sim`: console simulator reading `data/replay.csv` @ ~50 Hz.
  - `src/core`: **WarningEngine** (State pattern: Normal/Active/Faulted).
  - `src/ipc/dbus` + `apps/vehicle_service_dbus`: D-Bus daemon.
  - **`apps/hmi_qml`:** “Vehicle Info” screen (speed gauge, RPM gauge, warning icon)
- **Docs:** SRS, AD, UD, Test Specific, Traceability.
---

## 4) Success Criteria (HMI-centric)
- **UI end-to-end latency (P99):** `< 40 ms` from ECU tick → rendered frame (timestamp at source & sink).
- **Build/Docs:** Create CMake; V-model documents complete and consistent.

---

## 5) Scope & Context
- **ECU Simulator** parses CSV every 20 ms.
- **VehicleService (QtDBus)** caches state, emits signals, serves properties/methods.
- **HMI (Qt/QML)** **subscribes to signals** and binds properties → gauges/indicator update as data changes.
