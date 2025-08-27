# Unit Design (UD)

> Scope: Describe Class/Object, attribute, method and pseudocode for important function and ensure Unit design cover all requirements from Architecture design
---

## 0) Notation & Conventions
- **Namespaces (logic):** `core::`, `services::vehicle::`, `ipc::dbus::`, `hmi::`.
- **DTO** = Data Transfer Object.  
- **Pseudocode**: Show flow work in functions.

---

## 1) Data Contracts (DTOs) — (SR-01..03, SR-20/21)

**SpeedDTO**  
- Fields: `kph: double`, `ts_ms: uint64`

**RpmDTO**  
- Fields: `rpm: double`, `ts_ms: uint64`

**FuelDTO**
- Fields: `fuel: double (0...100)`, `ts_ms: unit64`

**Warn (enum)**  
- Values: `Normal`, `Active`, `Faulted`  
- Helper: `to_string(Warn) -> "Normal"|"Active"|"Faulted"`

**WarningDTO**  
- Fields: `state: Warn`, `ts_ms: uint64`

**Row** (1 row CSV)  
- Fields: `ts_ms: uint64`, `kph: double`, `rpm: double`, `fault: bool`, `active: bool`

---

## 2) Domain Logic — `core::WarningEngine` (SR-20)

### 2.1 Classes & Responsibilities
- **`core::IWarningState`**  
  - Operation: `step(fault: bool, active: bool) -> IWarningState` (Set the next state)
  - Operation: `name() -> Warn`

- **Concrete states:** `NormalState`, `ActiveState`, `FaultedState`  
  - Setting state machine following rule from SRS.

- **`core::WarningEngine`**  
  - Attr: `state: IWarningState` (Intialize as `NormalState`)  
  - Attr: `last: WarningDTO` (snapshot latest warning)  
  - Operation:
    - `step(fault, active, ts_ms) -> changed: bool`
    - `state() -> Warn`
    - `last() -> WarningDTO`

### 2.2 Pseudocode — transitions
```pseudo
NormalState.step(fault, active):
  if fault == true     return FaultedState()
  if active == true    return ActiveState()
  return NormalState()

ActiveState.step(fault, active):
  if fault == true     return FaultedState()
  if active == false   return NormalState()
  return ActiveState()

FaultedState.step(fault, active):
  if fault == false    return NormalState()
  return FaultedState()
```

### 2.3 Pseudocode - WarningEngine.step
```pseudo
WarningEngine.step(fault, active, ts_ms):
  next = state.step(fault, active)
  changed = (next.name() != state.name())
  state = next
  last = WarningDTO{ state: state.name(), ts_ms: ts_ms }
  return changed
```

## 3. CSV Input - `service::vehicle::CsvReader (SR-08/09)

### 3.1: Responsibilities
- Reading CSV file, ignore comment row with `#` .
- True or 0 is fault, False or 1 is active.
- Not stop completely when error reading a row, it continue reading the next row.

### 3.2: Pseudocode
```pseudocode
CsvReader.open(path):
  try open file at path for reading
  if fail -> return false
  return true

CsvReader.next() -> Optional<Row>:
  while file not EOF:
    line = read_line()
    if line starts_with '#': continue
    fields = split_by_comma(line)
    if fields.size < 5: log_warn("bad row"); continue
    ts_ms = parse_uint64(fields[0]); kph = parse_double(fields[1]); rpm = parse_double(fields[2])
    fault = parse_bool(fields[3])   ; active = parse_bool(fields[4])
    if any parse failed: log_warn("bad row"); continue
    return Row{ts_ms, kph, rpm, fault, active}
  return None
```

## 4. D-Bus Server - `ipc::dbus::DbusServer` (SR-01..07)

### 4.1: Responsibilities
- Manage **StateCache** and **WarningEngine**
- Register **Busname** `org.example.Vehicle`, **Object path** `/org/example/Vehicle`, **Interface** `org.example.Vehicle.V1`
- Provide **Methods/Properties**, emit **Signals** 
- Tick rate 20ms

### 4.2: State & Interface
- Attribute:
    - `conn: session_bus_connection`
    - `registered: bool`
    - `timer_period_ms: int = 20`
    - `reader: CsvReader`
    - `engine: WarningEngine`
    - `speed: SpeedDTO`, `rmp: RpmDTO`, `fuel: FuelDTO`, `warn: WarningDTO`
- Method (D-Bus):
    - `GetSpeed() -> (double kph, unit64 ts_ms)`
    - `GetRpm() -> (double rmp, unit64 ts_ms)`
    - `GetFuel() -> (string state, unit64 ts_ms)`
    - `Health() -> OK | ERROR
- Properties (read-only):
    - `Speed: double`, `Rpm: double`, `Fuel: double`, `WarningState: string`
- Signals:
    - `SpeedChanged (double, unit64)`
    - `RpmChanged(double, unit64)`
    - `FuelChanged(double, unit64)`
    - `WarningChanged(string, unit64)`

### 4.3: Pseudocode - lifecycle
```pseudocode
DbusServer.start(csv_path):
  if !reader.open(csv_path): log_error("CSV open failed"); return false
  if !registerOnBus():        log_error("DBus register failed"); return false
  start_timer(period_ms = 20, callback = onTick)
  return true

DbusServer.registerOnBus():
  bind bus name, export object+interface
  if success -> registered = true
  return registered
```

### 4.4: Pseudocode - tick & emission
```pseudocode
DbusServer.onTick():
  row = reader.next()
  if row is None:
    // end-of-file policy: loop or stop; v1: loop from start or hold last?
    // For v1: loop from start for demo continuity (documented).
    restart_reader_from_beginning()
    return

  speed = SpeedDTO{ kph: row.kph, ts_ms: row.ts_ms }
  rpm   = RpmDTO  { rpm: row.rpm, ts_ms: row.ts_ms }
  warn_changed = engine.step(row.fault, row.active, row.ts_ms)
  warn  = engine.last()

  emit Signal SpeedChanged(speed.kph, speed.ts_ms)
  emit Signal RpmChanged(rpm.rpm, rpm.ts_ms)
  if warn_changed:
    emit Signal WarningChanged(to_string(warn.state), warn.ts_ms)
```

### 4.5: Pseudocode - methods
```Pseudocode
DbusServer.GetSpeed():
  return (speed.kph, speed.ts_ms)

DbusServer.GetRpm():
  return (rpm.rpm, rpm.ts_ms)

DbusServer.GetWarnings():
  return (to_string(warn.state), warn.ts_ms)

DbusServer.Health():
  if registered == true: return OK
  else raise DBusError("NotRegistered")
```

## 5. D-Bus Client - `ipc::dbus::VehicleProxy` (SR-10..14)

### 5.1: Responsibilities
- Connect to service, subscribe signals, provide latest data to HMI
- Implement **snapshot()** when starting/reconnect
- Follow up **offline** if over 5000ms which not receive any event

### 5.2: State & API
- Attribute:
    - `iface: dbus_interface("org.example.Vehicle", "/org/example/Vehicle", "V1")`
    - `speed: double`, `rpm: double`, `warning: string`
    - `last_source_ts_ms: unit64`
    - `last_event_monotonic_ms: stopwatch`
    - `online: bool`

- Method:
    - `connectToService() -> bool`
    - `snapshot() -> bool`
    - `lastEventAgeMs() -> int`
    - Signals (Qt/Observer): `speedChanged`, `rpmChnaged`, `warningChanged`, `offline`, `online`

### 5.3: Pseudocode - connect & snapshot
```pseudocode
VehicleProxy.connectToService():
  iface = open_dbus_interface(...)
  if iface invalid: return false
  subscribe("SpeedChanged", onSpeedSignal)
  subscribe("RpmChanged", onRpmSignal)
  subscribe("WarningChanged", onWarningSignal)
  set_online(false)  // until first signal/snapshot
  return true

VehicleProxy.snapshot():
  (skph, sts) = call_dbus("GetSpeed")
  (rrpm, rts) = call_dbus("GetRpm")
  (wstr, wts) = call_dbus("GetWarnings")

  speed = skph; rpm = rrpm; warning = wstr
  last_source_ts_ms = max(sts, rts, wts)
  reset(last_event_monotonic_ms)
  set_online(true)
  emit speedChanged(speed, sts)
  emit rpmChanged(rpm, rts)
  emit warningChanged(warning, wts)
  return true
```

### 5.4 Pseudocode -signal handler & offline
```pseudocode
VehicleProxy.onSpeedSignal(kph, ts_ms):
  speed = kph
  last_source_ts_ms = ts_ms
  reset(last_event_monotonic_ms)
  if online == false: set_online(true)
  emit speedChanged(kph, ts_ms)

VehicleProxy.onRpmSignal(rpm_val, ts_ms):
  rpm = rpm_val
  last_source_ts_ms = ts_ms
  reset(last_event_monotonic_ms)
  if online == false: set_online(true)
  emit rpmChanged(rpm_val, ts_ms)

VehicleProxy.onWarningSignal(state_str, ts_ms):
  warning = state_str
  last_source_ts_ms = ts_ms
  // warningChanged có thể không đều; không dùng cho offline detection
  emit warningChanged(state_str, ts_ms)

VehicleProxy.lastEventAgeMs():
  if stopwatch not started -> return -1
  return elapsed_ms(stopwatch)

VehicleProxy.set_online(on):
  if online != on:
    online = on
    if on: emit online()
    else: emit offline()
```

## 6. HMI Bridge - `hmi::VehicleViewModel` (SR-11..14)

### 6.1: Responsibilities
- Expose Q_PROPERTY like: `speed`, `rpm`, `fuel`, `warningstate`, `offline`, `uiLatencyMs`
- Receive signal from `VehicleProxy` and update property
- Period housekeeping to check offline (>5000 ms) and update `uiLatencyMs`

### 6.2: State & API
- Attribue:
    - `proxy: VehicleProxy`
    - `speed: double`, `rpm: double`, `warningState: string`
    - `offline: bool` (default = true)
    - `uiLatencyMs: int` (default = -1)
    - `timer_housekeeping_ms = ~ 33ms` (~ 30Hz)
- Method:
    - `snapshot()` (call proxy)
    - Handler: `onSpeed(kph, ts_ms)`, `onRpm(rpm, ts_ms)`, `onWarn(str, ts_ms)`
    - `onHouseKeepingTick()`

### 6.3: Pseudocode - Handler2 & housekeeping
```pseudocode
VehicleViewModel.onSpeed(kph, ts_ms):
  speed = kph
  offline = false
  uiLatencyMs = now_ms() - ts_ms
  notify(speedChanged)
  notify(offlineChanged if changed)
  notify(uiLatencyChanged)

VehicleViewModel.onRpm(rpm_val, ts_ms):
  rpm = rpm_val
  uiLatencyMs = now_ms() - ts_ms
  notify(rpmChanged)
  notify(uiLatencyChanged)

VehicleViewModel.onWarn(state_str, ts_ms):
  if warningState != state_str:
    warningState = state_str
    notify(warningStateChanged)

VehicleViewModel.onHousekeepingTick():
  age = proxy.lastEventAgeMs()
  nowOffline = (age >= 0) and (age > 500)
  if offline != nowOffline:
    offline = nowOffline
    notify(offlineChanged)
```

## 7. Sequence (Runtime) - (SR-11..14)

### 7.1: Startup & snapshot:
- Start `VehicleService` with CSV path
- Start `HMI`, `VehicleProxy.connectToService()`.
- HMI call `snapshot()` -> sync `speed/rpm/fuel/warning` -> show

### 7.2: Streaming (20 ms/tick)
- DbusServer.onTick() -> update DTO + emit `SpeedChanged`/`RpmChanged`; `WarningChanged` when changing state
- `VehicleProxy` receive signal -> update value + mark online -> notify `VehicleViewMode`
- QML bind update gauges/indicator immediately

### 7.3: Disconnect/ Reconnect
- Service die -> client lost signal, `VehicleViewModel.onHouseKeepingTick()` check age > 5000ms -> `offline = true`
- Service start -> VehicleProxy.snapshot() -> HMI resync, `offline=false`

## 8. Error Handling & Policies

### 8.1: CSV errors
```pseudocode
if CsvReader.open fails -> log ERROR; DbusServer.start returns false
if bad row -> log WARN; skip; continue reading
EOF -> v1: restart_reader_from_beginning()
```

### 8.2: D-Bus registration failure
```psudocode
attempt = 0
while attempt < 3:
  if registerOnBus(): break
  sleep(300ms); attempt++
if not registered: log ERROR; return false
```

### 8.3: Client offline
```pseudocode
if proxy.lastEventAgeMs() > 500:
  set offline=true (once) and show banner
on next valid signal/snapshot:
  offline=false
```

## 9. Threading & timing (NFR-UI-LAT/FPS/Stream)
- **Service:** single-threaded event loop (tick = 20ms). If parse slowly -> move `CsvReader` to worker thread (queue messages)

- **HMI:** callbacks clearly, render thread is taken over by Qt/QML
- **Budget reference: parse < 1ms -> emit < 0.5ms -> HMI slot < 2ms -> render < 16ms

## 10. Observability (SR-16/17)
- Log format (line-oriented): `ts, level, module, msg, kv ...`
- Counters: tick rate, drop %, last event age
- UI latency: `uiLatencyMs = now_ms - ts_ms_from_server`

## 11. Public Contract (Reference) - (SR-07)
- **Bus:** `org.example.Vehicle`
- **Path:** `/org/example/Vehicle`
- **Interface:** `org.example.Vehilce.V1`
