완전 좋아, 이제 이걸 **시퀀스 다이어그램에서 바로 쓸 수 있는 수준**으로 못 박아줄게.
각 서버별로:

* 인터페이스(Provided/Required)
* 그 안에 들어갈 **operation 시그니처 (이름, 파라미터, 리턴)**

만 정의할 거야.
속성은 안 넣고, 우리가 계속 써온 공통 DataType/Enum을 그대로 활용한다.

---

## 공통 DataType / Enum (요약)

```text
type CellarId      = string
type DeviceId      = string
type UserId        = string
type GroupId       = string

enum VendorKey     { SAMSUNG, LG }
enum DeviceType    { AC, AIR_PURIFIER, FAN }
enum Severity      { INFO, WARN, CRITICAL }
enum HealthStatus  { OK, WARN, FAIL }

type EnvSample {
  cellarId: CellarId
  temperature: float
  humidity: float
  airQualityIndex: float
  timestamp: datetime
}

type Policy {
  cellarId: CellarId
  targetTempMin: float
  targetTempMax: float
  targetHumMin: float
  targetHumMax: float
  targetAqiMax: float
}

type ControlCommand {
  commandId: string
  cellarId: CellarId
  deviceId: DeviceId
  payload: map<string, any>
  issuedAt: datetime
}

type CommandResult {
  commandId: string
  deviceId: DeviceId
  success: bool
  errorCode?: string
  completedAt: datetime
}

type DeviceStatus {
  deviceId: DeviceId
  cellarId: CellarId
  vendor: VendorKey
  type: DeviceType
  status: HealthStatus
  detail?: string
  lastCheckedAt: datetime
}

type AlarmEvent {
  alarmId: string
  cellarId: CellarId
  deviceId?: DeviceId
  severity: Severity
  category: string
  message: string
  occurredAt: datetime
}
```

이제 서버별로 인터페이스+오퍼레이션.

---

## 1. Broker Server

### 1) 외부 공개 (Facade)

**IBrokerPublish (Provided by Broker)**

```text
publishEnvSample(sample: EnvSample): void
publishDeviceStatus(status: DeviceStatus): void
publishControlCommand(cmd: ControlCommand): void
publishCommandResult(result: CommandResult): void
publishAlarm(event: AlarmEvent): void
```

**IBrokerSubscribe (Provided by Broker)**

```text
subscribeEnvSamples(groupId: GroupId, handlerId: string): void
subscribeDeviceStatus(groupId: GroupId, handlerId: string): void
subscribeControlCommands(groupId: GroupId, handlerId: string): void
subscribeCommandResults(groupId: GroupId, handlerId: string): void
subscribeAlarms(groupId: GroupId, handlerId: string): void
```

> 시퀀스 다이어그램에서: HVAC/Monitoring/WCS/Adapter가 이 두 인터페이스를 호출.

### 2) 내부 연계

**IIngressRouting (Required by BrokerIngressController, Provided by TopicRoutingService)**

```text
handleIncoming(topic: string, payload: bytes, sourceId: string): void
```

**IKafkaProducer (Required by TopicRoutingService, Provided by kafkaCluster)**

```text
sendToKafka(topic: string, key: string?, payload: bytes): void
```

**IKafkaConsumer (Required by InternalClientAdapter, Provided by kafkaCluster)**

```text
registerSubscription(topic: string, groupId: GroupId, handlerId: string): void
```

**IMqttIngress (Required by MQTTBridgeAdapter, Provided by BrokerIngressController)**

```text
onMqttMessage(topic: string, payload: bytes, clientId: string): void
```

이 네 개가 Broker 내부 컴포넌트 연결용 오퍼레이션.

---

## 2. HVAC Server

### 1) 상단: UI / 이벤트 진입

**IHVACControlManagement (Provided by HVACEventFrontController)**

```text
getCurrentControlState(cellarId: CellarId): List<ControlCommand>
forceRecalculate(cellarId: CellarId): void
getRecentAlarms(cellarId: CellarId, limit: int): List<AlarmEvent>
```

**(UI는 이걸 호출)**

---

### 2) 비즈니스 로직

**IHVACAutoControl (Provided by CellarHVACControlService)**

```text
handleEnvSample(sample: EnvSample): void
handleCommandResult(result: CommandResult): void
onSensorTimeout(cellarId: CellarId, lastSeen: datetime): void
```

(브로커 구독/타이머 등을 통해 FrontController나 Scheduler가 호출)

---

### 3) 하단: Repository / Gateway

**IPolicyRepository (Provided by PolicyDAC)**

```text
getPolicy(cellarId: CellarId): Policy
savePolicy(policy: Policy): void        // (WCS에서 사용)
```

**IEnvTSRepository (Provided by EnvTimeSeriesDAC)**

```text
saveEnvSample(sample: EnvSample): void
getRecentEnvSamples(cellarId: CellarId, minutes: int): List<EnvSample>
```

**IDeviceControlGateway (Provided by DeviceCommandGateway)**

```text
sendCommand(cmd: ControlCommand): CommandResult
sendBulkCommands(cmds: List<ControlCommand>): List<CommandResult>
```

**IAlarmGateway (Provided by AlarmNotificationGateway)**

```text
publishAlarm(event: AlarmEvent): void       // 브로커로
pushAlarmToKakao(event: AlarmEvent): void   // Kakao로
```

**IBrokerPublish / IBrokerSubscribe (Provided by BrokerGateway, Required by BL)**

```text
publishControlCommand(cmd: ControlCommand): void
subscribeEnvSamples(groupId: GroupId, handlerId: string): void
subscribeCommandResults(groupId: GroupId, handlerId: string): void
```

> 시퀀스에서:
> `HVACEventFrontController.handleEnvSample` → `CellarHVACControlService.handleEnvSample` → `IPolicyRepository`/`IDeviceControlGateway`/`IBrokerPublish` 호출 흐름으로 쓰면 된다.

---

## 3. Monitoring Server

### 1) 상단: 모니터링 UI / 이벤트 진입

**IDeviceMonitoringConsole (Provided by MonitoringWebFrontController)**

```text
getLatestStatus(deviceId: DeviceId): DeviceStatus
getProblemDevices(): List<DeviceStatus>
getRecentAlarms(cellarId: CellarId, limit: int): List<AlarmEvent>
```

**MonitoringEventFrontController**

* 별도 인터페이스 정의 없이,
* `onBrokerEvent(event: AlarmEvent | DeviceStatus): void` 형태로 BL 호출에 써도 됨.

---

### 2) 비즈니스 로직

**IDeviceHealthCheck (Provided by DeviceHealthCheckService)**

```text
runPeriodicHealthCheck(): void
runHealthCheckFor(deviceId: DeviceId): void
```

**IAlarmEvaluation (Provided by AlarmEvaluationService)**

```text
evaluate(status: DeviceStatus): AlarmEvent?    // 알람 필요없으면 null
```

---

### 3) 하단: Repository / Gateway

**IDeviceStatusRepository (Provided by DeviceStatusDAC)**

```text
saveStatus(status: DeviceStatus): void
getLastStatus(deviceId: DeviceId): DeviceStatus
getStatusesByCellar(cellarId: CellarId): List<DeviceStatus>
```

**IAlarmRepository (Provided by AlarmEventDAC)**

```text
saveAlarm(event: AlarmEvent): void
getRecentAlarms(cellarId: CellarId, limit: int): List<AlarmEvent>
```

**IVendorHealthGateway (Provided by VendorDeviceAPIGateway)**

```text
queryStatus(deviceId: DeviceId, vendor: VendorKey): DeviceStatus
```

**IAlarmGateway (Provided by KakaoPushGateway or AlertGW)**

```text
pushToKakao(event: AlarmEvent): void
pushToWebConsole(event: AlarmEvent): void
```

**IBrokerPublish / IBrokerSubscribe (Provided by BrokerGateway)**

```text
publishDeviceStatus(status: DeviceStatus): void
publishAlarm(event: AlarmEvent): void
subscribeControlCommands(groupId: GroupId, handlerId: string): void  // 필요 시
subscribeAlarms(groupId: GroupId, handlerId: string): void
```

> 시퀀스 예:
> Scheduler → `runPeriodicHealthCheck` → `queryStatus`(IVendorHealthGateway) → `evaluate` → `saveAlarm` + `publishAlarm` + `pushToKakao`.

---

## 4. WCS Server (Wine Cellar Control System)

### 1) 상단: 관리 UI

**IWcsAdminUI (Provided by WcsAdminFrontController)**

```text
createOrUpdatePolicy(policy: Policy): void
registerDevice(cellarId: CellarId, deviceId: DeviceId, vendor: VendorKey, type: DeviceType): void
viewCellarDashboard(cellarId: CellarId): CellarDashboardView
viewAlarmHistory(cellarId: CellarId, limit: int): List<AlarmEvent>
```

(대시보드/뷰 타입은 DTO로 두면 됨.)

---

### 2) 비즈니스 로직

**IPolicyManagement (Provided by PolicyManagementService)**

```text
savePolicy(policy: Policy): void
getPolicy(cellarId: CellarId): Policy
```

**ICellarDeviceRegistry (Provided by CellarDeviceRegistryService)**

```text
registerDevice(cellarId: CellarId, deviceId: DeviceId,
               vendor: VendorKey, type: DeviceType): void
getDevices(cellarId: CellarId): List<DeviceId>
```

**IMonitoringQuery (Provided by MonitoringViewService)**

```text
getCurrentEnv(cellarId: CellarId): EnvSample?
getRecentAlarms(cellarId: CellarId, limit: int): List<AlarmEvent>
getCellarOverview(cellarId: CellarId): CellarDashboardView
```

---

### 3) 하단: Repository

**IPolicyRepository (Provided by MasterDataDAC)**

```text
savePolicy(policy: Policy): void
getPolicy(cellarId: CellarId): Policy
```

**IDeviceRegistryRepository (Provided by MasterDataDAC)**

```text
saveDevice(cellarId: CellarId, deviceId: DeviceId,
           vendor: VendorKey, type: DeviceType): void
getDevices(cellarId: CellarId): List<DeviceId>
```

**IEnvTSRepository (Provided by EnvTimeSeriesQueryDAC)**

```text
getRecentEnvSamples(cellarId: CellarId, minutes: int): List<EnvSample>
```

**IAlarmRepository (Provided by AlarmEventDAC)**

```text
getRecentAlarms(cellarId: CellarId, limit: int): List<AlarmEvent>
```

> 시퀀스 예:
> Admin UI → `IWcsAdminUI.createOrUpdatePolicy` → `IPolicyManagement.savePolicy` → `IPolicyRepository.savePolicy`
> Dashboard 조회 → `IMonitoringQuery` → `IEnvTSRepository`, `IAlarmRepository`.

---

이제:

* 각 인터페이스의 오퍼레이션이 **시퀀스 다이어그램 메시지 이름**이 되고,
* 컴포넌트 다이어그램에서는 “Provided / Required 인터페이스”로 연결하면,
* UC별로 흐름(센서 → Broker → HVAC 제어, 장치 Health → Monitoring → Alarm, 정책 변경 → HVAC 사용)이 매우 깔끔하게 이어질 거야.

필요하면 특정 UC 하나 골라서, 위 오퍼레이션들만 써서 시퀀스 다이어그램 예시도 바로 만들어 줄 수 있어.
