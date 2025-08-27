# Software Requirements Specification (SRS)

## 1. Purpose & Scope
This SRS defines functional and non-functional requirements for a mini IVI stack whose primary goal is to demonstrate a **real-time HMI** reacting to **ECU Simulator** data through a **D-Bus** middleware service. The system runs on **Ubuntu (WSL2)** with **WSLg** for GUI.

In-scope (v1):
- ECU Simulator (CSV replay, ~50 Hz).
- VehicleService (QtDBus daemon) exposing speed, rpm, warning state via methods, properties, and signals.
- HMI (Qt/QML) that subscribes to D-Bus, renders gauges and warning indicator in real time.
- Documentation (V-model) is created clearly.

---

## 2. Definitions & Acronyms
- **HMI**: Human-Machine Interface (Qt/QML app).
- **D-Bus**: Linux IPC bus (session bus in v1).
- **WSL2 / WSLg**: Windows Subsystem for Linux v2 / GUI layer for WSL.
- **P99 latency**: 99th-percentile end-to-end latency (source tick → HMI render).
- **CSV tick**: One row of replay data (kph, rpm, flags, ts).

---

## 3. System Overview
**Flow:** ECU Simulator (CSV 20ms tick) → VehicleService (QtDBus) → HMI (Qt/QML).  
**Clean architecture dependency:** `apps → services → ipc/adapters → core` (one-way).  

---

## 4. Stakeholders & User Classes
- **End user personal (HMI):** Driver observe speed/RPM/warning in real time.

---

## 5. Operating Environment
- **OS:** Ubuntu (WSL2) on Windows 11; GUI via **WSLg**.
- **Toolchain:** g++/clang, CMake, Ninja, Qt6 + QtDBus, dbus.
- **Runtime:** Session D-Bus; local PC-only.

---

## 6. Functional Requirements (SR-xx)

### 6.1 VehicleService API (D-Bus)
- **SR-01** — `GetSpeed()`  
  Returns `(kph: double, ts_ms: uint64)` — latest cached values.  
  *Acceptance:* Correct types and monotonic `ts_ms`. *Verify:* TC-HMI-02.

- **SR-02** — `GetRpm()`  
  Returns `(rpm: double, ts_ms: uint64)`; see SR-01 criteria. *Verify:* TC-HMI-02.

- **SR-03** — `GetWarnings()`  
  Returns `(state: string in {"Normal","Active","Faulted"}, ts_ms: uint64)`.  
  *Verify:* TC-HMI-03, TC-WARN-01.

- **SR-04** — Signals **50–100 Hz** (or event-driven for warning):  
  `SpeedChanged(double kph, uint64 ts_ms)`,  
  `RpmChanged(double rpm, uint64 ts_ms)`,  
  `WarningChanged(string state, uint64 ts_ms)` (on change).  
  *Verify:* TC-HMI-01 (receive rate ≥ 40 ev/s for 50 Hz; tolerance), TC-HMI-03.

- **SR-05** — Read-only properties: `Speed`, `Rpm`, `WarningState`.  
  *Verify:* TC-HMI-02 (snapshot/binding).

- **SR-06** — `Health()` indicates service readiness; returns OK on success.  
  *Verify:* Integration smoke.

### 6.2 ECU Simulator (CSV Input)
- **SR-07** — Ingest CSV rows at **~20 ms** intervals (configurable).  
  *Verify:* TC-HMI-01 (observed event cadence).

- **SR-08** — CSV schema:  
  `ts_ms(uint64), kph(double), rpm(double), fault(0/1), active(0/1)`; lines starting with `#` ignored.  
  Invalid rows are skipped with a warning log.  
  *Verify:* TC-CSV-01/02/03.

### 6.3 Domain Logic (Warning State)
- **SR-09** — WarningEngine implements state machine  
  `Normal ↔ Active ↔ Faulted` per UD transition table.  
  *Verify:* TC-WARN-01 (unit).

### 6.4 HMI (Qt/QML, mandatory)
- **SR-10** — HMI **subscribes** to all 3 signals and renders **immediately** on events.  
  *Verify:* TC-HMI-01 (rate), TC-UI-LAT (latency).

- **SR-11** — HMI screen “Vehicle Info”:  
  **Speed gauge**, **RPM gauge**, **Warning indicator** (Normal/Active/Faulted).  
  *Verify:* TC-HMI-03 (visual/logic).

- **SR-12** — **Offline indicator** when **no events > 5000 ms**.  
  *Verify:* TC-UI-OFFLINE.

- **SR-13** — HMI **Snapshot** action on start/reconnect: calls `Get*` to sync baseline.  
  *Verify:* TC-HMI-02.

### 6.5 Build & Run
- **SR-14** — Build with CMake; out-of-source build supported.  
  *Verify:* CI build.

- **SR-15** — Scripts or README instructions to run 3 processes:  
  ECU simulator, VehicleService (D-Bus), HMI.  
  *Verify:* Smoke run.

---

## 7. Non-Functional Requirements (NFR-xx)
- **NFR-01 Performance** — **UI P99 end-to-end latency < 40 ms**, measured as  
  `ui_latency_ms = (HMI_render_ts_ms − source_ts_ms)` over ≥ 60s.  
  *Verify:* TC-UI-LAT.

- **NFR-02 Smoothness** — **FPS ≥ 60**.  
  *Verify:* TC-UI-FPS.

- **NFR-03 Stream Reliability** — Drops **< 1%** over 5 minutes.  
  *Verify:* System test.

- **NFR-04 Resource Usage** — RSS < 150 MB per process, CPU < 15% on reference laptop.  
  *Verify:* Perf profiling.

- **NFR-05 Documentation (M)** — Project Information, SRS, AD, UD, Test Specification, Traceability kept consistent and up-to-date for v1 baseline.  
  *Verify:* Doc review.

---

## 8. External Interface Requirements
### 8.1 D-Bus Contract
- **Bus name:** `org.example.Vehicle`  
- **Object path:** `/org/example/Vehicle`  
- **Interface:** `org.example.Vehicle.V1`

**Methods**
- `GetSpeed() -> (double kph, uint64 ts_ms)`
- `GetRpm() -> (double rpm, uint64 ts_ms)`
- `GetWarnings() -> (string state, uint64 ts_ms)`
- `Health() -> ()` (no args; returns OK or raises D-Bus error on failure)

**Signals**
- `SpeedChanged(double kph, uint64 ts_ms)`
- `RpmChanged(double rpm, uint64 ts_ms)`
- `WarningChanged(string state, uint64 ts_ms)` (only when state changes)

**Properties (read-only)**
- `Speed: double`, `Rpm: double`, `WarningState: string`


### 8.2 File Interface (CSV)
- Path: `data/replay.csv`  
- Schema (comma separated):  
  `ts_ms(uint64), kph(double), rpm(double), fault(0|1|true|false), active(0|1|true|false)`  
- Lines starting with `#` are comments; invalid rows are skipped (logged).

---

## 9. Use Cases
- **UC-01 Live Display**  
  *Primary:* HMI shows live speed/RPM and warning; updates on each event.  
  *Basic flow:* Start all processes → HMI subscribes → ECU ticks → VehicleService emits → HMI renders gauges/indicator.  
  *Alternate:* If no events for >5000 ms → offline indicator.

- **UC-02 Snapshot Sync**  
  *Primary:* On HMI start or reconnect, it calls `Get*` to sync baseline values.  
  *Alternate:* If call fails → retry with backoff; show offline indicator.

- **UC-03 Fault Transition**  
  *Primary:* CSV sets `fault=1` or `active=1` → WarningEngine transitions; HMI indicator changes.  
  *Alternate:* Fault clears → back to Normal.

- **UC-04 Service Interruption**  
  *Primary:* VehicleService crashes/restarts → HMI shows offline; on recovery, HMI resyncs via snapshot.

---

## 10. Data Models
- **Data Transfer Objects:**  
  `Speed { double kph; uint64 ts_ms; }`  
  `Rpm   { double rpm; uint64 ts_ms; }`  
  `WarningState includes {"Normal","Active","Faulted"}`

- **State Machine (WarningEngine):**

| current | fault | active | next    |
|--------:|:-----:|:------:|:-------:|
| Normal  |   0   |   0    | Normal  |
| Normal  |   0   |   1    | Active  |
| Normal  |   1   |   *    | Faulted |
| Active  |   0   |   1    | Active  |
| Active  |   0   |   0    | Normal  |
| Active  |   1   |   *    | Faulted |
| Faulted |   0   |   *    | Normal  |
| Faulted |   1   |   *    | Faulted |

---

## 11. Verification & Traceability (high level)
| Req | Verification |
|-----|--------------|
| SR-01..06, SR-07 | Introspection + integration tests (TC-HMI-02, smoke) |
| SR-08..09 | CSV unit/integration tests (TC-CSV-01/02/03) |
| SR-10 | Unit tests (TC-WARN-01) |
| SR-11..15 | HMI integration (TC-HMI-01/02/03), offline (TC-UI-OFFLINE) |
| NFR-01..03 | Perf/System tests (TC-UI-LAT, TC-UI-FPS, drop measurement) |
| NFR-04..09 | Perf profiling, build graph, environment smoke, doc review |

A full traceability matrix is maintained in `docs/TRACEABILITY.md`.
