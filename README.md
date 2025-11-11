좋아, 이제 진짜 “콜백 패턴까지 포함해서 레이어 규칙 안 어긋나는 버전”으로 다시 정리해줄게.
요점은 두 가지야:

1. **정적 의존성(컴포넌트/인터페이스 화살표)은 항상 상 → 하**
2. 이벤트/타이머/Kafka 소비처럼 “실행 시점에 하위에서 상위로 올라오는 것처럼 보이는 것”은
   **상위가 콜백 인터페이스를 제공하고, 하위가 그걸 호출하는 Callback 패턴으로 표현**

아래에 서버별로:

* 컴포넌트 목록
* 각 인터페이스가 어느 레이어에 있고,
* 누가 누구를 (상→하) 참조하는지,
* 콜백은 어떻게 표현하는지

까지 한 번에 볼 수 있게 정리했다.

---

## 1. 와인 공조 운영 서버 (Wine Cellar Ops Server)

**역할:** 정책/장치/창고정보/리포트. **브로커 직접 사용 안 함.** MasterDB + TSDB만 사용.

### Presentation Layer

* `OpsApiFrontController`

  * 단순 라우팅 (1:1 매핑)
* Controllers (각각 1:1 API)

  * `EnvMonitoringController` : `IEnvMonitoringAPI`
  * `ControlPolicyController` : `IControlPolicyAPI`
  * `InventoryController` : `IInventoryAPI`
  * `AlarmPolicyController` : `IAlarmPolicyAPI`
  * `DeviceConfigController` : `IDeviceConfigAPI`

**상→하 관계**

* `OpsApiFrontController` → 각 `I*API`
* 각 Controller → 아래 Application 서비스 인터페이스

### Application Layer

* `EnvStatusQueryService` : `IEnvStatusQueryService`
* `ZoneProfileService` : `IZoneProfileService`
* `ControlPolicyService` : `IControlPolicyService`
* `InventoryEventService` : `IInventoryEventService`
* `AlarmRuleService` : `IAlarmRuleService`
* `DeviceConfigService` : `IDeviceConfigService`
  (장치/존 매핑, 제조사, 인터페이스 타입 관리)

**상→하 관계**

* `EnvMonitoringController` → `IEnvStatusQueryService`
* `ControlPolicyController` → `IZoneProfileService`, `IControlPolicyService`
* `InventoryController` → `IInventoryEventService`
* `AlarmPolicyController` → `IAlarmRuleService`
* `DeviceConfigController` → `IDeviceConfigService`

### Infrastructure Layer

* Master DB Repos

  * `MasterDBAccess` :

    * `IZoneProfileRepository`
    * `IWarehouseDeviceRepository`
    * `IInventoryRepository`
    * `IAlarmRuleRepository`
* TS DB Repos

  * `TimeseriesDBAccess` :

    * `IEnvTimeseriesRepository`
    * `IControlHistoryRepository`
    * `IDeviceStatusTimeseriesRepository`
* 외부 클라이언트

  * `WarehouseSystemClient` : `IWarehouseSystemClient`
  * `KEPCOClient` : `IKEPCOClient`
  * `PushClient` : `IPushClient`

**상→하 관계 (대표)**

* `EnvStatusQueryService` → `IEnvTimeseriesRepository`, `IWarehouseDeviceRepository`
* `ZoneProfileService` → `IZoneProfileRepository`
* `ControlPolicyService` → `IZoneProfileRepository`, `IEnvTimeseriesRepository`, `IControlHistoryRepository`, `IWarehouseDeviceRepository`, (필요시) `IKEPCOClient`
* `InventoryEventService` → `IInventoryRepository`, `IWarehouseSystemClient`
* `AlarmRuleService` → `IAlarmRuleRepository`, `IPushClient`
* `DeviceConfigService` → `IWarehouseDeviceRepository`

**콜백 필요 없음** (하위가 상위를 직접 부르는 흐름 없음)

---

## 2. HVAC 서버 (Kafka + Callback 적용 핵심)

**역할:** 센서/입출고 이벤트 기반 공조 제어, 제어 이력 기록.
브로커(Kafka) + MasterDB + TSDB + 장비 벤더 API 사용.

### Presentation Layer

* `HVACApiController`

  * `IHVACControlAPI`
  * `IHVACStatusAPI`

**상→하**

* `HVACApiController` → `IHVACControlService`, `IHVACStatusQueryService`

### Application Layer

#### 도메인/서비스 인터페이스

* `HVACControlService`

  * 구현: `IHVACControlService`
  * **이벤트 핸들러 역할도 겸함**

    * `IEnvSampleEventHandler`
    * `IInOutEventHandler`
* `SetpointCalculationService` : `ISetpointCalculationService`
* `DeviceCommandService` : `IDeviceCommandService`
* `HVACStatusQueryService` : `IHVACStatusQueryService`

**상→하 관계**

* `HVACApiController` → `IHVACControlService`, `IHVACStatusQueryService`
* `HVACControlService` → `ISetpointCalculationService`, `IDeviceCommandService`
* `HVACControlService` → (아래 인터페이스들로 **등록/조회**)

  * `IEnvSampleSubscription`
  * `IInOutEventSubscription`
  * `IHVACDeviceConfigRepository`
  * `IPolicyReferenceRepository`
* `DeviceCommandService` → `IDeviceCommandPublishing`, `ISamsungHVACClient`, `ILGHVACClient`, `IControlHistoryRepository_HVAC`
* `HVACStatusQueryService` → `IControlHistoryRepository_HVAC`, `IHVACDeviceConfigRepository`

#### Callback 인터페이스 (Application 레이어 정의)

* `IEnvSampleEventHandler`

  * `onEnvSample(zoneId, sample)`
* `IInOutEventHandler`

  * `onItemEvent(event)`

`HVACControlService` 가 둘 다 구현.

### Infrastructure Layer

* `BrokerGatewayForHVAC`

  * 제공 인터페이스(상위에서 사용):

    * `IEnvSampleSubscription`

      * `registerEnvSampleHandler(handler: IEnvSampleEventHandler)`
    * `IInOutEventSubscription`

      * `registerInOutEventHandler(handler: IInOutEventHandler)`
    * `IDeviceCommandPublishing`

      * `publishDeviceCommand(cmd)`
  * 내부에서 Kafka Consumer/Producer 처리.
  * **실행 시**: 이벤트 수신 시 등록된 `IEnvSampleEventHandler` / `IInOutEventHandler`를 호출 (Sequence에서 `<<callback>>`로 표기).

* Master DB

  * `HVACMasterDBAccess`

    * `IHVACDeviceConfigRepository`
    * `IPolicyReferenceRepository`

* TS DB

  * `HVACTimeseriesDBAccess`

    * `IEnvTimeseriesRepository_HVAC`
    * `IControlHistoryRepository_HVAC`

* Vendor Clients

  * `SamsungHVACClient` : `ISamsungHVACClient`
  * `LGHVACClient` : `ILGHVACClient`

**상→하 정적 관계**

* `HVACControlService` → `IEnvSampleSubscription`, `IInOutEventSubscription`,
  `IHVACDeviceConfigRepository`, `IPolicyReferenceRepository`
* `SetpointCalculationService` → `IEnvTimeseriesRepository_HVAC`
* `DeviceCommandService` → `IDeviceCommandPublishing`,
  `ISamsungHVACClient`, `ILGHVACClient`,
  `IControlHistoryRepository_HVAC`
* `HVACStatusQueryService` → `IControlHistoryRepository_HVAC`,
  `IHVACDeviceConfigRepository`

**콜백 요약**

* 등록: 상위(`HVACControlService`) → 하위(`BrokerGatewayForHVAC`)
  `registerEnvSampleHandler(this)`, `registerInOutEventHandler(this)`
* 호출: 하위가 **등록된 인터페이스**를 호출
  `BrokerGatewayForHVAC -> IEnvSampleEventHandler.onEnvSample()` (시퀀스에서 `<<callback>>`)
* 정적 의존성 관점: Application ↔ (공통 인터페이스) ↔ Infra 이고, **하위→상위 직접 참조 없음.**

---

## 3. 모니터링 서버 (Heartbeat + 1minTimer Callback)

**역할:** 삼성/LG 공조장치 heartbeat 주기 확인, 상태 TSDB 기록, 이상 알림.
센서 X, 브로커 X. Timer + Vendor API + DB.

### Presentation Layer

* `MonitoringApiController`

  * `IMonitoringQueryAPI`
  * `IAlarmQueryAPI`

**상→하**

* `MonitoringApiController` → `IMonitoringQueryService`, `IAlarmManagementService`

### Application Layer

#### 콜백/스케줄링

* `DeviceHealthCheckScheduler`

  * `I1MinTickListener` 구현
  * 1분마다 호출되면 `DeviceHealthCheckService` 실행
* `DeviceHealthCheckService` : `IDeviceHealthCheckService`
* `AlarmManagementService` : `IAlarmManagementService`
* `MonitoringQueryService` : `IMonitoringQueryService`

**상→하 관계**

* `MonitoringApiController` → `IMonitoringQueryService`, `IAlarmManagementService`
* `DeviceHealthCheckScheduler` → `IDeviceHealthCheckService`
* `DeviceHealthCheckService` →

  * `IMonDeviceConfigRepository`
  * `ISamsungHVACStatusClient`
  * `ILGHVACStatusClient`
  * `IDeviceHeartbeatRepository`
* `AlarmManagementService` → `IPushClient`, `IDeviceStatusHistoryRepository`
* `MonitoringQueryService` → `IDeviceHeartbeatRepository`, `IDeviceStatusHistoryRepository`

#### Timer Callback 인터페이스

* `I1MinTickListener` (Application에 정의)

  * `on1MinTick()`

`DeviceHealthCheckScheduler` 가 구현.

### Infrastructure Layer

* `TimerGateway`

  * `I1MinTimerRegistration`

    * `register1MinTickListener(listener: I1MinTickListener)`
  * 실제 OS/스케줄러 기반 1분 타이머.
  * 1분마다 `listener.on1MinTick()` 호출 (`<<callback>>`)

* Master DB

  * `MonMasterDBAccess` : `IMonDeviceConfigRepository`

* TS DB

  * `MonTimeseriesDBAccess` :

    * `IDeviceHeartbeatRepository`
    * `IDeviceStatusHistoryRepository`

* Vendor Status Clients

  * `SamsungHVACStatusClient` : `ISamsungHVACStatusClient`
  * `LGHVACStatusClient` : `ILGHVACStatusClient`

* `PushClient` : `IPushClient`

**상→하 정적 관계**

* `DeviceHealthCheckScheduler` → `I1MinTimerRegistration` (등록)
* `DeviceHealthCheckService` → `IMonDeviceConfigRepository`,
  `ISamsungHVACStatusClient`, `ILGHVACStatusClient`,
  `IDeviceHeartbeatRepository`
* `AlarmManagementService` → `IPushClient`, `IDeviceStatusHistoryRepository`
* `MonitoringQueryService` → `IDeviceHeartbeatRepository`,
  `IDeviceStatusHistoryRepository`

**콜백 요약**

* 상위(App) `DeviceHealthCheckScheduler` 가 `TimerGateway.register1MinTickListener(this)` 호출.
* 하위(Infra) Timer가 1분마다 `I1MinTickListener.on1MinTick()` 호출 (`<<callback>>`).
* 정적 의존성은 상→하만 존재.

---

이렇게 정리하면:

* 세 서버 모두 **레이어 규칙 + Callback 사용 가이드**에 정확히 들어맞고,
* 각 인터페이스가 “누가 제공 / 누가 사용 / 어떤 방향”인지 명확해서,
* 이후 UC 시퀀스 다이어그램 그릴 때 그대로 가져다 써도 구조적 모순이 생기지 않아.

다음 단계에서 원하면,
각 서버별 대표 UC(예: “센서값 수신→HVAC 제어”, “1분 heartbeat 체크→알람 발송”, “Ops에서 정책 변경→HVAC 반영”)를 이 인터페이스 이름 그대로 넣어서 시퀀스 다이어그램으로 바로 풀어줄게.

==========================================================================================

좋다, 이제 이걸 “진짜 코드 짤 수 있을 만큼” 인터페이스/데이터타입 정의까지 내려보자.
요구한 것처럼:

* 공통 DataType/Enum 먼저 정의
* 각 서버별로 **Provided / Required 인터페이스를 레이어 기준으로 분리**
* 각 오퍼레이션의 파라미터/리턴 타입을 전부 명시

구체 언어는 안 박고, 타입 이름만 딱 보면 설계 리뷰에서 읽히도록 써둘게.

---

## 0. 공통 DataType / Enum 정의

### ID 계열

```text
type SensorId        = string
type WarehouseId     = string
type ZoneId          = string       // 창고 내 존/셀러 단위
type DeviceId        = string       // 공조 장치, 센서 장치 등
type ManufacturerId  = string       // "SAMSUNG", "LG", ...
type AlarmId         = string
type CommandId       = string
```

### 환경 측정 관련

```text
type Timestamp       = datetime

enum EnvMetricType { TEMP, HUMID, CO2, PM25, VOC }

struct EnvSample {
  sensorId: SensorId
  warehouseId?: WarehouseId  // 매핑 후 사용할 수 있음
  zoneId?: ZoneId
  measuredAt: Timestamp
  temperature?: float
  humidity?: float
  co2?: float
  pm25?: float
  voc?: float
}

struct EnvSampleWithLocation {
  warehouseId: WarehouseId
  zoneId: ZoneId
  measuredAt: Timestamp
  temperature?: float
  humidity?: float
  co2?: float
  pm25?: float
  voc?: float
}
```

### 정책/프로파일/제어

```text
struct ZoneProfile {
  warehouseId: WarehouseId
  zoneId: ZoneId
  targetTempRange: (float min, float max)
  targetHumidityRange: (float min, float max)
  targetCo2Max?: float
  targetPm25Max?: float
}

struct ControlPolicyRange {
  warehouseId: WarehouseId
  zoneId: ZoneId
  tempMin: float
  tempMax: float
  humidityMin: float
  humidityMax: float
  co2Max?: float
  pm25Max?: float
}

enum DeviceType { HVAC, AIR_PURIFIER, VENTILATION, HUMIDIFIER, DEHUMIDIFIER }

enum ManufacturerType { SAMSUNG, LG }

struct WarehouseDevice {
  deviceId: DeviceId
  warehouseId: WarehouseId
  zoneId: ZoneId
  manufacturer: ManufacturerType
  deviceType: DeviceType
  address: string           // API endpoint or logical address
}

struct TargetControlCommand {
  commandId: CommandId
  warehouseId: WarehouseId
  zoneId: ZoneId
  deviceId: DeviceId
  mode: string              // 예: "COOL", "HEAT", ...
  setTemp?: float
  setHumidity?: float
  fanSpeed?: int
  etcOptions: map<string, string>
}
```

### 장치 상태/모니터링

```text
enum HealthState { NORMAL, WARN, DOWN, UNKNOWN }

struct DeviceStatus {
  deviceId: DeviceId
  warehouseId: WarehouseId
  checkedAt: Timestamp
  health: HealthState
  detailMessage?: string
}

struct HeartbeatRecord {
  deviceId: DeviceId
  warehouseId: WarehouseId
  checkedAt: Timestamp
  success: bool
  responseTimeMs?: int
}
```

### 알람 / 결과 공통

```text
enum AlarmSeverity { INFO, WARN, ERROR, CRITICAL }

enum AlarmType {
  UNKNOWN_SENSOR,
  ENV_STORE_FAILURE,
  HVAC_CONTROL_ERROR,
  DEVICE_DOWN,
  PUSH_SERVER_ERROR
}

struct AlarmEvent {
  alarmId: AlarmId
  type: AlarmType
  severity: AlarmSeverity
  occurredAt: Timestamp
  warehouseId?: WarehouseId
  deviceId?: DeviceId
  message: string
}

enum PushStatus { SUCCESS, TEMP_FAILURE, PERM_FAILURE }

struct OperationResult {
  success: bool
  errorCode?: string
  message?: string
}
```

이걸 기반으로 서버별 인터페이스를 깔끔히 나눈다.

---

## 1. Wine Cellar Ops Server

Ops는 **Master/TS DB 기반 비즈니스/조회 서버**. 브로커/센서 직접 X.

### 1-1. Presentation Layer – Provided APIs

(외부/운영자/다른 서버에서 호출)

**IEnvMonitoringAPI (Provided by EnvMonitoringController)**

```text
EnvStatusView getCurrentEnvStatus(warehouseId: WarehouseId, zoneId?: ZoneId)

EnvHistoryView getEnvHistory(
  warehouseId: WarehouseId,
  zoneId: ZoneId,
  from: Timestamp,
  to: Timestamp
)
```

**IControlPolicyAPI (Provided by ControlPolicyController)**

```text
OperationResult defineZoneProfile(profile: ZoneProfile)

ZoneProfile getZoneProfile(warehouseId: WarehouseId, zoneId: ZoneId)
```

**IInventoryAPI (Provided by InventoryController)**

```text
OperationResult registerInOutEvent(event: InOutEvent)

InOutHistoryView getInOutHistory(warehouseId: WarehouseId, period: TimeRange)
```

**IAlarmPolicyAPI (Provided by AlarmPolicyController)**

```text
OperationResult defineAlarmRule(rule: AlarmRule)

AlarmRule getAlarmRule(ruleId: string)
```

**IDeviceConfigAPI (Provided by DeviceConfigController)**

```text
OperationResult registerWarehouseDevice(device: WarehouseDevice)

WarehouseDevice getWarehouseDevice(deviceId: DeviceId)
List<WarehouseDevice> listWarehouseDevices(warehouseId: WarehouseId)
```

### 1-2. Application Layer – Provided Services / Required Infra

#### IEnvStatusQueryService (Provided by EnvStatusQueryService)

```text
EnvStatusView getCurrentEnvStatus(warehouseId: WarehouseId, zoneId?: ZoneId)

EnvHistoryView getEnvHistory(
  warehouseId: WarehouseId,
  zoneId: ZoneId,
  from: Timestamp,
  to: Timestamp
)
```

**Requires (downwards):**

* `IEnvTimeseriesRepository`
* `IWarehouseDeviceRepository`

#### IZoneProfileService (Provided by ZoneProfileService)

```text
OperationResult defineZoneProfile(profile: ZoneProfile)
ZoneProfile getZoneProfile(warehouseId: WarehouseId, zoneId: ZoneId)
```

**Requires:** `IZoneProfileRepository`

#### IControlPolicyService (Provided by ControlPolicyService)

(Ops에서 정책 계산·시뮬레이션 용도)

```text
ControlPolicyRange getPolicyRange(warehouseId: WarehouseId, zoneId: ZoneId)

ControlSimulationResult simulateControl(
  warehouseId: WarehouseId,
  zoneId: ZoneId,
  currentEnv: EnvSampleWithLocation
)
```

**Requires:**

* `IZoneProfileRepository`
* `IEnvTimeseriesRepository`
* `IControlHistoryRepository`
* `IWarehouseDeviceRepository`
* (옵션) `IKEPCOClient`

#### IInventoryEventService

```text
OperationResult applyInOutEvent(event: InOutEvent)
InOutHistoryView getInOutHistory(warehouseId: WarehouseId, range: TimeRange)
```

**Requires:** `IInventoryRepository`, `IWarehouseSystemClient`

#### IAlarmRuleService

```text
OperationResult defineAlarmRule(rule: AlarmRule)
AlarmRule getAlarmRule(ruleId: string)
```

**Requires:** `IAlarmRuleRepository`

#### IDeviceConfigService

```text
OperationResult registerWarehouseDevice(device: WarehouseDevice)
WarehouseDevice getWarehouseDevice(deviceId: DeviceId)
List<WarehouseDevice> listDevices(warehouseId: WarehouseId)
```

**Requires:** `IWarehouseDeviceRepository`

### 1-3. Infrastructure Layer – Provided / Required

**Repositories (Provided)**

```text
IZoneProfileRepository
  +save(profile: ZoneProfile): OperationResult
  +find(warehouseId, zoneId): ZoneProfile?

IWarehouseDeviceRepository
  +save(device: WarehouseDevice): OperationResult
  +find(deviceId): WarehouseDevice?
  +listByWarehouse(warehouseId): List<WarehouseDevice>

IInventoryRepository
  +save(event: InOutEvent): OperationResult
  +list(warehouseId, range): List<InOutEvent>

IAlarmRuleRepository
  +save(rule: AlarmRule): OperationResult
  +find(ruleId): AlarmRule?
```

**TimeSeries (Provided)**

```text
IEnvTimeseriesRepository
  +save(sample: EnvSampleWithLocation): OperationResult
  +query(warehouseId, zoneId, range): List<EnvSampleWithLocation>

IControlHistoryRepository
  +save(history: ControlHistoryRecord): OperationResult
  +list(warehouseId, range): List<ControlHistoryRecord>

IDeviceStatusTimeseriesRepository
  +save(status: DeviceStatus): OperationResult
  +list(warehouseId, range): List<DeviceStatus>
```

**External Clients (Provided)**

```text
IWarehouseSystemClient
  +notifyInOutSynced(event: InOutEvent): OperationResult

IKEPCOClient
  +getPowerUsage(warehouseId, range): PowerUsageView

IPushClient
  +send(message: PushMessage): PushStatus
```

**Required by Repos/Clients:** DB connection, HTTP 등 infra 디테일 (여기선 생략)

---

## 2. HVAC 서버

핵심: 브로커 이벤트 → `HVACControlService` 콜백, 제조사별 제어.

### 2-1. Presentation – Provided

**IHVACControlAPI (HVACApiController)**

```text
OperationResult requestImmediateControl(warehouseId: WarehouseId)

HVACControlStatusView getControlStatus(warehouseId: WarehouseId, zoneId?: ZoneId)
```

**IHVACStatusAPI**

```text
HVACControlStatusView getLastControlResult(commandId: CommandId)
```

### 2-2. Application – Provided / Required

#### IHVACControlService (Provided by HVACControlService)

```text
OperationResult requestHVACControl(warehouseId: WarehouseId)

/* 콜백: 브로커에서 직접 호출, Provided interface */
onEnvSample(sensorId: SensorId, envData: EnvSample)         // from IEnvSampleEventHandler
onItemEvent(event: InOutEvent)                              // from IInOutEventHandler
```

**Requires (downwards):**

* `IEnvSampleSubscription` (for register)
* `IInOutEventSubscription`
* `IHVACDeviceConfigRepository`
* `IPolicyReferenceRepository`
* `ISetpointCalculationService`
* `IDeviceCommandService`

#### ISetpointCalculationService

```text
TargetControlCommandSet calculate(
  env: EnvSampleWithLocation,
  policy: ControlPolicyRange,
  devices: List<WarehouseDevice>
)
```

**Requires:** `IEnvTimeseriesRepository_HVAC` (history 기반 계산 시)

#### IDeviceCommandService

```text
OperationResult executeCommands(commands: TargetControlCommandSet)
```

**Requires:**

* `IDeviceCommandPublishing` (if via broker)
* `ISamsungHVACClient`, `ILGHVACClient`
* `IControlHistoryRepository_HVAC`

#### IHVACStatusQueryService

```text
HVACControlStatusView getStatus(warehouseId: WarehouseId, zoneId?: ZoneId)
HVACControlStatusView getByCommandId(commandId: CommandId)
```

**Requires:** `IControlHistoryRepository_HVAC`, `IHVACDeviceConfigRepository`

### 2-3. Infrastructure – Provided / Callback

**Kafka Gateway (Provided)**

```text
IEnvSampleSubscription
  +registerEnvSampleHandler(h: IEnvSampleEventHandler): void

IInOutEventSubscription
  +registerInOutEventHandler(h: IInOutEventHandler): void

IDeviceCommandPublishing
  +publishDeviceCommand(cmd: TargetControlCommand): OperationResult
```

**HVAC DB (Provided)**

```text
IHVACDeviceConfigRepository
  +findByWarehouse(warehouseId): List<WarehouseDevice>
  +findById(deviceId): WarehouseDevice?

IPolicyReferenceRepository
  +getPolicyRange(warehouseId, zoneId): ControlPolicyRange?

IEnvTimeseriesRepository_HVAC
  +getLatestSample(warehouseId, zoneId): EnvSampleWithLocation?

IControlHistoryRepository_HVAC
  +save(record: ControlHistoryRecord): OperationResult
  +findByCommandId(commandId): ControlHistoryRecord?
  +list(warehouseId, range): List<ControlHistoryRecord>
```

**Vendor Clients (Provided)**

```text
ISamsungHVACClient
  +sendCommands(commands: TargetControlCommandSet): OperationResult

ILGHVACClient
  +sendCommands(commands: TargetControlCommandSet): OperationResult
```

**Callback 사용**

* `BrokerGatewayForHVAC` 는 실제 이벤트 시:

  * `IEnvSampleEventHandler.onEnvSample(...)`
  * `IInOutEventHandler.onItemEvent(...)`
    호출 (상위 Provided 인터페이스).

---

## 3. Monitoring 서버

Timer + Vendor API로 공조 장치 Heartbeat만 관리.

### 3-1. Presentation – Provided

**IMonitoringQueryAPI**

```text
List<DeviceStatus> getDeviceStatuses(warehouseId: WarehouseId)

List<HeartbeatRecord> getHeartbeatHistory(
  warehouseId: WarehouseId,
  deviceId: DeviceId,
  range: TimeRange
)
```

**IAlarmQueryAPI**

```text
List<AlarmEvent> getActiveAlarms(warehouseId?: WarehouseId)
List<AlarmEvent> getAlarmHistory(warehouseId?: WarehouseId, range?: TimeRange)
```

### 3-2. Application – Provided / Required

#### I1MinTickListener (Provided by DeviceHealthCheckScheduler)

```text
on1MinTick(): void
```

#### IDeviceHealthCheckService

```text
checkSamsungDevices(): void
checkLGDevices(): void
```

(실제 구현에선 제조사 파라미터 1개로 합쳐도 됨)

**Requires:**

* `IMonDeviceConfigRepository`
* `ISamsungHVACStatusClient` / `ILGHVACStatusClient`
* `IDeviceHeartbeatRepository`
* `IAlarmManagementService` (DOWN or FAIL 시 알람 요청)

#### IAlarmManagementService

```text
raiseDeviceDownAlarm(device: WarehouseDevice): AlarmEvent
closeDeviceAlarmIfRecovered(device: WarehouseDevice): OperationResult
List<AlarmEvent> getActiveAlarms(warehouseId?: WarehouseId)
List<AlarmEvent> getAlarmHistory(warehouseId?: WarehouseId, range?: TimeRange)
```

**Requires:**

* `IPushClient`
* `IDeviceStatusHistoryRepository`

#### IMonitoringQueryService

```text
List<DeviceStatus> getDeviceStatuses(warehouseId: WarehouseId)
List<HeartbeatRecord> getHeartbeatHistory(
  warehouseId: WarehouseId,
  deviceId: DeviceId,
  range: TimeRange
)
```

**Requires:**

* `IDeviceHeartbeatRepository`
* `IDeviceStatusHistoryRepository`

#### DeviceHealthCheckScheduler (Provided / Required)

* **Provided:** `I1MinTickListener.on1MinTick()`
* **Requires:** `I1MinTimerRegistration` (타이머 등록), `IDeviceHealthCheckService`

### 3-3. Infrastructure – Provided / Callback

**Timer**

```text
I1MinTimerRegistration
  +register(listener: I1MinTickListener): void
```

Timer 엔진은 1분마다 `listener.on1MinTick()` 호출 (callback).

**Repositories**

```text
IMonDeviceConfigRepository
  +listSamsungDevices(): List<WarehouseDevice>
  +listLGDevices(): List<WarehouseDevice>

IDeviceHeartbeatRepository
  +save(record: HeartbeatRecord): OperationResult
  +list(warehouseId, deviceId, range): List<HeartbeatRecord>

IDeviceStatusHistoryRepository
  +save(status: DeviceStatus): OperationResult
  +list(warehouseId?: WarehouseId, range?: TimeRange): List<DeviceStatus>
```

**Vendor Status Clients**

```text
ISamsungHVACStatusClient
  +getStatus(device: WarehouseDevice): DeviceStatus

ILGHVACStatusClient
  +getStatus(device: WarehouseDevice): DeviceStatus
```

**Push**

```text
IPushClient
  +send(message: PushMessage): PushStatus
```

---

## 정리 포인트 (검토용)

* 각 인터페이스는 **어느 레이어에서 제공(Provided)**, **위 레이어/서비스에서 무엇을 Require하는지**가 명확하게 갈린 상태.
* UC 4.1.4.2.3 / 4.1.5.5 / 4.1.6.5의 플로우는:

  * UC-01: `IEnvSampleSubscription` + `IEnvSampleEventHandler` / TS repo / PushClient 로 직접 매핑.
  * UC-02: `IHVACControlService` + 정책/장치 repo + VendorClient.
  * UC-03/04: `I1MinTimerRegistration` + `I1MinTickListener` + VendorStatusClient + TS repo.
* 하위→상위 직접 의존은 전부 **콜백 인터페이스(상위 제공) 사용**으로 정리되어 있어, 피드백의 layered 규칙과도 충돌 없음.

이 정도면 설계 리뷰 때 “각 UC가 어떤 컴포넌트/인터페이스를 통해 흘러가는지”를 코드 레벨까지 바로 보일 수 있을 거야.
다음 단계에서 원하면, 이 정의들을 표 형태(Provided/Required matrix)로도 정리해 줄게.

