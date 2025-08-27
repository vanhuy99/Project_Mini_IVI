# Architecture Design (AD)

## 1) Architecture Overview

**Goal:** The HMI updates **in real time** when ECU data changes via **D-Bus**.

**Flow:**  
ECU Simulator (CSV @20 ms) → **VehicleService (QtDBus daemon)** → **HMI (Qt/QML)**

**Clean dependency direction:**  
`apps → services → ipc/adapters → core` (one-way).  
`src/core` is **pure C++** (no Qt dependency).

---

## 2) Containers & Processes

### 2.1 ECU Simulator (`apps/ecu_sim`)
- **Purpose:** Produce rows from `data/replay.csv` every ~20 ms (~50 Hz).
- **Responsibilities:** Parse CSV → `Row{ts_ms,kph,rpm,fault,active}`; log stats; act as the data source for the service.
- **Non-functional:** stable cadence, low jitter, low CPU.

### 2.2 VehicleService (`apps/vehicle_service_dbus`)
- **Purpose:** Publish a stable **D-Bus API** for Speed/RPM/Warning.
- **Responsibilities:**
  - Maintain **StateCache** (latest snapshot).
  - Run **WarningEngine** (Normal/Active/Faulted).
  - Expose **methods/properties**; **emit signals** on cadence/change.
- **IPC:** QtDBus on **session bus** (WSLg).
- **Data errors:** malformed CSV rows → skip + log; emit `WarningChanged` only on state transitions.

### 2.3 HMI (`apps/hmi_qml`)
- **Purpose:** Render the “Vehicle Info” screen in real time.
- **Responsibilities:** QtDBus **client proxy** subscribes to `*Changed`; **ViewModel** (Q_PROPERTY) exposes speed, rpm, warning; QML binds and renders; **offline banner** if >5000 ms without events.

---

## 3) Component View

### 3.1 Core (pure C++)
- `src/core/warning_engine.hpp/.cpp`
  - `IWarningState`, `Normal`, `Active`, `Faulted`
  - `WarningEngine::step(fault,active)`, `state() -> std::string`

### 3.2 IPC / D-Bus glue (Qt)
- `src/ipc/dbus/DbusServer.*`
  - Owns StateCache, exports methods/properties, emits signals.
  - Registers bus `org.example.Vehicle`, path `/org/example/Vehicle`, interface `org.example.Vehicle.V1`.
- `src/ipc/dbus/DbusClient.*` (HMI side)
  - Subscribes to signals; calls `Get*` for **snapshot**; emits Qt notify signals for the ViewModel.

### 3.3 Services (thin façade)
- `src/services/vehicle/VehicleService.*` (optional)
  - Wraps domain & D-Bus server; manages start/stop lifecycle.

### 3.4 Apps
- `apps/vehicle_service_dbus/main.cpp`  
  Boot: parse args → init Qt → construct `DbusServer` → event loop.
- `apps/hmi_qml/main.cpp`  
  Init QtQuick → create `VehicleProxy` + `VehicleViewModel` → load `Main.qml`.
- `apps/ecu_sim/main.cpp`  
  Read CSV; print progress; (in v1 the service reads CSV; simulator acts as a validation tool).

---

## 4) Public Contract (D-Bus API)

**Bus:** `org.example.Vehicle`  
**Path:** `/org/example/Vehicle`  
**Interface:** `org.example.Vehicle.V1` (versioned, **additive in V1**)

```xml
<!-- docs/dbus/vehicle_v1.xml -->
<node>
  <interface name="org.example.Vehicle.V1">
    <method name="GetSpeed"><arg direction="out" type="d"/><arg direction="out" type="t"/></method>
    <method name="GetRpm"><arg direction="out" type="d"/><arg direction="out" type="t"/></method>
    <method name="GetWarnings"><arg direction="out" type="s"/><arg direction="out" type="t"/></method>
    <method name="Health"/>
    <signal name="SpeedChanged"><arg type="d"/><arg type="t"/></signal>
    <signal name="RpmChanged"><arg type="d"/><arg type="t"/></signal>
    <signal name="WarningChanged"><arg type="s"/><arg type="t"/></signal>
    <property name="Speed" type="d" access="read"/>
    <property name="Rpm" type="d" access="read"/>
    <property name="WarningState" type="s" access="read"/>
  </interface>
</node>
```


## 5) Runtime Scenarios (Sequence)

### 5.1 Normal streaming (20 ms/tick)

#### 5.1.1: ECU reads one CSV row → Row{ts,kph,rpm,fault,active}.

#### 5.1.2: VehicleService:
- WarningEngine.step(fault,active) → update StateCache.
- Emit:
    - SpeedChanged(kph,ts)
    - RpmChanged(rpm,ts)
    - WarningChanged(state,ts) only on state change.

#### 5.1.3: HMI:
- Client slot fires → updates ViewModel.
- QML rebinds → gauges/indicator repaint.
- If nowMs - lastEventMs > 500 → show offline banner.

## 5.2 HMI startup/reconnect (snapshot)
- HMI calls GetSpeed/GetRpm/GetWarnings to sync baseline; then subscribes to signals.

## 5.3 Service interruption
- Service dies: proxy disconnects → HMI marks offline.
- Service restarts: HMI reconnects + snapshot() to resync.

## 6) Threading & Concurrency
- VehicleService
    - Main thread (Qt event loop): D-Bus dispatch + emitting signals.
    - QTimer @20 ms for CSV reading (or a worker thread if parsing is heavy).
    - Rule: keep handlers fast; avoid blocking the event loop.
- HMI
    - UI thread = render + D-Bus callbacks.
    - Rule: keep callbacks light; update ViewModel and return.

## 7) Performance Budgets (aligned with SRS)
- P99 UI latency < 40 ms (tick → rendered frame).
Suggested budget: parse <1 ms → emit <0.5 ms → HMI slot <2 ms → render <16 ms.
- FPS ≥ 60, jank < 5% over 60 s.
- Drops < 1% over 5 minutes @ 50–100 Hz.
- Per-process resources: RSS < 150 MB, CPU < 15% (reference laptop).

Techniques  
- Latest-value rendering (drop stale events when overloaded).
- Avoid hot-path allocations; reuse buffers.

## 8) Configuration & Packaging
- Inputs (v1): data/replay.csv (CLI arg for the service).
- Output:
    - core_warning_engine (static lib)
    - vehicle_service_dbus (daemon)
    - hmi_qml (UI)
    - ecu_sim (tool)

## 9) Observability & Diagnostics
- Logging: per module (core, dbus, hmi), include source ts_ms.
- Counters: tick rate, dropped events, age of last event.
- Manual probes:

```bash
qdbus org.example.Vehicle /org/example/Vehicle org.example.Vehicle.V1.Health
dbus-monitor "type='signal',interface='org.example.Vehicle.V1'"
```
- Perf tools: `pidstat`, `perf record/report`, Qt Quick Profiler.


10) Failure Modes & Handling

| Failure        | Detection         | Handling        |
|:---------------|:------------------|:----------------|
| CSV missing/invalid | open/parse error  | log error; exit with non-zero; smoke test fails fast   |
| D-Bus registration fails| register failure              | retry with bounded backoff; if still failing → exit non-zero             |
| HMI disconnects/crash   | proxy disconnection signals   | show offline banner; retry connection; call `snapshot()` on reconnect    |
| High CPU / jank         | perf counters, profiler       | latest-value policy; reduce heavy QML bindings; precompute in ViewModel  |
| Contract drift          | CI introspection diff         | contract-first XML; versioned `…V1`; **additive-only** changes           |

## 12) Build & Deployment (WSL2 + WSLg)

```bash
# Configure & build
cmake -S . -B build -GNinja -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build

# Run (3 terminals)
./build/apps/ecu_sim/ecu_sim ./data/replay.csv
./build/apps/vehicle_service_dbus/vehicle_service_dbus ./data/replay.csv
./build/apps/hmi_qml/hmi_qml
```
