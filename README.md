좋다, 이제 한 번 “감독관 모드”로 깐깐하게 검토하고 딱 정리해볼게.

## 0. 전체 구조 최종 체크

* DD-01 (브로커·엣지·모니터링 분리)에서 요구한 역할 분할과 정확히 일치:

  * 브로커: **센서/장치 instant 데이터·명령 전달**만. 
  * HVAC: 공조 수행.
  * 모니터링: 상태 감지/알림.
  * 공조 시스템 서버: 정책/마스터/리포트 등 비즈니스 로직.
* DD-02 (마스터 DB와 시계열 DB 분리) 반영:

  * 마스터(정책·장치)는 공조 시스템 서버 중심.
  * 시계열/이벤트는 HVAC & 모니터링이 DAC 통해 사용. 
* Prefinal-Feedback 요구:

  * 노드별 컴포넌트 식별, 3계층(L1: UI/Facade, L2: BL, L3: DAC/Gateway).
  * 상→하 단방향 의존, Layer Skip 없음.
  * Stable/ISP 기반 Interface, Publish/Subscribe 분리. 

지금까지 정리한 구조에 **치명적인 구멍은 없다.**
이제 서버별 인터페이스 & 필요한 타입을 깔끔하게 리스트로 박아줄게.
(함수 시그니처는 “명세용” 수준으로, 실제 언어 독립)

---

## 1. 와인셀러 공조 시스템 서버

### 1-1. 상단(Presentation/API) Provided Interfaces

**IAdminConfigService (Provided)**

* `createWarehouseProfile(cmd: CreateWarehouseProfileCmd): void`
* `updateWarehouseProfile(cmd: UpdateWarehouseProfileCmd): void`
* `registerDevice(cmd: RegisterDeviceCmd): void`
* `updateDevice(cmd: UpdateDeviceCmd): void`

**IOperatorConfigService (Provided)**

* `viewPolicies(q: PolicyQuery): PolicySummaryList`
* `viewZoneProfile(zoneId: ZoneId): ZoneProfile`
* `viewDeviceInfo(deviceId: DeviceId): DeviceInfo`

**IReportQueryAPI (Provided)**

* `getEnvHistory(query: EnvHistoryQuery): EnvHistoryPage`
* `getControlHistory(query: ControlHistoryQuery): ControlHistoryPage`
* `getKpiSummary(query: KpiQuery): KpiSummary`

**IWarehouseIntegrationAPI (Provided, to 창고시스템)**

* `notifyInbound(event: InboundEvent): Ack`
* `notifyOutbound(event: OutboundEvent): Ack`

> 상단은 모두 내부 BL 인터페이스만 사용. DB/외부 시스템 직접 X.

### 1-2. 중단(Business) Provided/Required

**PolicyManagementService**

* Provided: `IPolicyManagement`

  * `savePolicy(cmd: SavePolicyCmd)`
  * `disablePolicy(policyId: PolicyId)`
* Required: `IPolicyRepositoryDAC`

**ZoneProfileService**

* Provided: `IZoneProfileManagement`
* Required: `IZoneRepositoryDAC`

**EquipmentRegistrationService**

* Provided: `IEquipmentRegistration`
* Required: `IEquipmentRepositoryDAC`

**InventoryIntegrationService**

* Provided: `IInventoryIntegration`
* Required: `IWarehouseSystemGateway`

**ReportingService**

* Provided: `IReportQuery`
* Required: `IReportViewDAC`

> 모든 BL은 오직 DAC/Gateway 인터페이스만 Required. Layer skip 없음.

### 1-3. 하단(DAC/Gateway) Provided

* `IPolicyRepositoryDAC`

  * `findById(policyId): Policy`
  * `save(policy: Policy): void`
* `IZoneRepositoryDAC`
* `IEquipmentRepositoryDAC`
* `IReportViewDAC`
* `IWarehouseSystemGateway`

  * `sendInbound(event: InboundEvent): Ack` (if needed 양방향)

---

## 2. 브로커 서버

※ 핵심: **도메인 DB에 쓰지 않고**, 오직 브로커 메타 DB만.

### 2-1. 상단(Ingress/Client) Provided

**IMqttMessageIngress (Provided)**

* `receive(topic: String, payload: BinaryMessage): Ack`

**IHttpMessageIngress (Provided)**

* `post(topic: String, payload: JsonMessage): Ack`

**IInternalPublish (Provided, 내부 서버용)**

* `publish(msg: BrokerMessage): Ack`

**ISubscriptionAccess (Provided)**

* `registerSubscriber(req: SubscribeRequest): Ack`
* `unregisterSubscriber(subscriberId: SubscriberId): Ack`
* `pullMessages(subscriberId: SubscriberId, maxCount: int): BrokerMessageBatch`

### 2-2. 중단(Core) Provided/Required

**MessageRoutingService**

* Provided:

  * `IEnqueueMessage.enqueue(msg: BrokerMessage): Ack`
* Required:

  * `IBrokerMetaStoreDAC` (토픽/큐 메타, 메시지 포인터)

**SubscriptionManager**

* Provided: `ISubscriptionManager`
* Required: `IBrokerMetaStoreDAC`

**DeliveryService**

* Provided: `IDeliveryService`

  * `getMessages(subscriberId, maxCount): BrokerMessageBatch`
* Required: `IBrokerMetaStoreDAC`

**QoSRetryService**

* Provided: `IQoSManagement`
* Required: `IBrokerMetaStoreDAC`

**AuthValidationService**

* Provided: `IAuthValidation`
* Required: `IBrokerMetaStoreDAC` (키/토큰 조회 시)

### 2-3. 하단(DAC) Provided

**IBrokerMetaStoreDAC**

* `storeMessage(msg: BrokerMessage): MessageId`
* `loadMessages(subscriberId, maxCount): BrokerMessageBatch`
* `saveOffset(subscriberId, offset: Offset): void`
* `saveSubscription(info: SubscriptionInfo): void`
* `saveDeadLetter(msg: BrokerMessage, reason: String): void`

---

## 3. HVAC 서버

### 3-1. 상단(API/Subscribe) Provided

**IHvacControlRequest (Provided)**

* (운영자/공조 시스템 서버 → 수동/예약 제어)
* `applyZoneControl(req: ZoneControlRequest): Ack`
* `overridePolicy(req: PolicyOverrideRequest): Ack`

**ISensorEventReceiver (Provided, from Broker)**

* `onEnvSample(sample: EnvSample): void`
* `onDeviceEvent(event: DeviceEvent): void`

**ICommandMessageReceiver (Provided, from Broker)**

* `onControlCommand(cmd: ControlCommand): void`

**IHvacStatusAPI (Provided)**

* `getZoneStatus(zoneId: ZoneId): ZoneStatus`
* `getDeviceStatus(deviceId: DeviceId): DeviceStatus`

### 3-2. 중단(Business) Provided/Required

**HvacControlService**

* Provided: `IHvacControlService`

  * `calculateAndApplyControl(sample: EnvSample): void`
  * `executeControl(cmd: ControlCommand): void`
* Required:

  * `IPolicyQueryDAC`
  * `IControlHistoryDAC`
  * `IVendorControlGateway`

**ZoneControlOrchestrator**

* Provided: `IZoneControl`

  * `applyToDevices(req: ZoneControlPlan): void`
* Required: `IVendorControlGateway`

**PolicyEvaluator**

* Provided: `IPolicyEvaluation`

  * `evaluate(sample: EnvSample, policy: Policy): ZoneControlPlan`
* Required: `IPolicyQueryDAC`

### 3-3. 하단(DAC/Gateway) Provided

**IPolicyQueryDAC**

* `getEffectivePolicy(zoneId): Policy`

**IControlHistoryDAC**

* `append(record: ControlRecord): void`
* `query(query: ControlHistoryQuery): ControlHistoryPage`

**IVendorControlGateway** (공통 인터페이스)

* `sendCommand(cmd: VendorCommand): VendorAck`
* 구현: `SamsungHvacGateway`, `LGHvacGateway`, …

---

## 4. 모니터링 서버

### 4-1. 상단(Dashboard/Subscribe) Provided

**IMonitoringView (Provided)**

* `getCurrentStatus(query: StatusQuery): StatusSnapshot`
* `getAlarmHistory(query: AlarmHistoryQuery): AlarmHistoryPage`

**IBrokerEventReceiver (Provided, from Broker)**

* `onEnvSample(sample: EnvSample): void`
* `onDeviceStatus(event: DeviceStatusEvent): void`
* `onBrokerHealth(event: BrokerHealthEvent): void`

**IHvacStatusReceiver (Provided, from HVAC)**

* `onHvacStatus(status: ZoneStatus): void`

### 4-2. 중단(Business) Provided/Required

**EventProcessingService**

* Provided: `IEventProcessing`

  * `handleEnvSample(sample)`
  * `handleDeviceStatus(event)`
  * `handleBrokerHealth(event)`
* Required:

  * `IMonitoringEventDAC`
  * `IAlarmService`

**AlarmService**

* Provided:

  * `IAlarmService`

    * `raiseAlarm(event: AlarmEvent): void`
    * `clearAlarm(alarmId: AlarmId): void`
  * `IAlarmQuery`
* Required:

  * `IAlarmHistoryDAC`
  * `IPushGateway`

**HealthEvaluationService**

* Provided: `IStatusQuery`
* Required: `IMonitoringEventDAC`

### 4-3. 하단(DAC/Gateway) Provided

**IMonitoringEventDAC**

* `saveEnvSample(sample: EnvSample): void`
* `saveStatus(event: DeviceStatusEvent): void`
* `loadStatus(query): StatusSnapshot`

**IAlarmHistoryDAC**

* `saveAlarm(alarm: AlarmEvent): void`
* `loadAlarms(query): AlarmHistoryPage`

**IPushGateway**

* `sendPush(msg: PushMessage): Ack`

---

## 5. 공통 DataType / Enum 제안

**ID/기본 VO**

* `WarehouseId`, `ZoneId`, `DeviceId`, `PolicyId`, `AlarmId`, `MessageId`, `SubscriberId`

**EnvSample**

* `timestamp: DateTime`
* `warehouseId, zoneId, sensorId`
* `temperature, humidity, co2, tvoc, illuminance, etc.`

**DeviceStatus / DeviceStatusEvent**

* `deviceId`
* `type: DeviceType`
* `status: DeviceState`
* `lastHeartbeatAt`
* `errorCode?: String`

**ControlCommand**

* `commandId`
* `targetZoneId or targetDeviceId`
* `type: CommandType` (e.g. `SET_TEMP`, `SET_MODE`, `SET_FAN_SPEED`)
* `params: Map<String,Any>`
* `requestedBy`
* `requestedAt`

**CommandResult**

* `commandId`
* `result: ResultCode`
* `message`
* `completedAt`

**AlarmEvent**

* `alarmId`
* `severity: AlarmSeverity`
* `type: AlarmType`
* `source: AlarmSource` (zone/device/broker/hvac)
* `message`
* `occurredAt`
* `clearedAt?`

**BrokerMessage**

* `topic: String`
* `headers: Map<String,String>`
* `payload: Binary/Json`
* `messageType: BrokerMessageType` (e.g. `ENV_SAMPLE`, `DEVICE_STATUS`, `CONTROL_COMMAND`, `ALARM`, …)

**Enum 예시**

* `DeviceType = {ENV_SENSOR, ITEM_SENSOR, HVAC, FAN, AIR_CLEANER, BROKER_NODE}`
* `DeviceState = {NORMAL, WARNING, ERROR, OFFLINE}`
* `AlarmSeverity = {INFO, WARNING, CRITICAL}`
* `CommandType = {SET_TEMP, SET_MODE, SET_HUMIDITY, SET_FAN_SPEED, POWER_ON, POWER_OFF}`
* `ResultCode = {SUCCESS, TEMPORARY_FAILURE, PERMANENT_FAILURE}`
* `BrokerMessageType`, `AlarmType`, `AlarmSource` 등은 시나리오에 맞게 확장.

---

## 6. 누락/문제 최종 점검 요약

* 각 서버 역할이 Architectural Driver와 DD-01/02 설명과 일치. 
* 각 서버 내부:

  * 상단: API/Facade/Subscriber
  * 중단: 도메인/브로커/모니터링 로직
  * 하단: Repository/DAC/Gateway
  * **상→중→하 단방향, Layer Skip 없음.**
* 브로커:

  * Publish/Subscribe 인터페이스 분리, 메타 DB만 접근 → Feedback 요구 충족. 
* 도메인/메시지 타입:

  * UC01~04와 QA-01~04를 커버하는 최소 DTO/Enum 세트 포함.

이대로 5.2.2 Interface List / 5.2.2.x Interface Spec에 옮기면 평가 관점에서 “빠진 거 있다” 소리 듣기 어렵다.
혹시 실제 문서에 옮기면서 표 구조가 애매하면, 그때는 우리가 지금 정의한 이름/시그니처를 그대로 복붙해서 정리하면 된다.





좋아, 이제 이 버전 기준으로 구조를 딱 고정해보자.

전제 정리부터 할게.

* 공조 시스템 서버 → 브로커 서버 HTTPS **삭제 완료**.
* **브로커 서버 역할**: 센서/서버 간 **메시지 라우팅 전담**. 도메인 이벤트(DB 최종 저장)는 다른 서버가 담당.
* 단, 브로커 내부 품질(QoS/Retry/Offset)을 위해 **자기 용도의 메타 DB**는 DAC 통해 CRUD 가능(도메인 DB와 구분).
* 모든 서버는 **3-Layer (상/중/하)**, **Layer Skip 금지**, **각 계층 간은 인터페이스로만 연결**, DAC/Gateway 규칙은 Prefinal-Feedback 기준 준수. 

아래 내용 그대로 5.2 Structure View (Static Structure + Component Spec + Interface List)에 쓰면 된다.

---

## 1. 서버별 역할 정리

### 1) 와인셀러 공조 시스템 서버

* 창고/존/장치/정책 설정 관리.
* 표준 인터페이스 관리(센서·공조장치 연동 스펙 문서화 관점).
* 에너지/환경 리포트, 이력 조회.
* 외부(창고 시스템, 전력 데이터 포털 등)와의 업무 연계.
* **실시간 제어/센서 스트림의 경로에는 직접 개입하지 않음.**

### 2) 브로커 서버

* AirGradient / Balluff / SmartThings / 기타 장치에서 들어오는 메시지를 수신.
* 토픽 기반 라우팅, 구독 관리, QoS/Retry 제공.
* HVAC/모니터링 등 내부 서버가 메시지를 **구독/수신**할 수 있게 허브 역할.
* 자기용 메타 DB(오프셋, DLQ 등)만 DAC로 접근. **환경/이벤트 본문을 운영 DB에 직접 기록하지 않음.**

### 3) HVAC 서버

* 정책과 센서 값을 기반으로 냉난방/습도/환기 제어 결정.
* 제조사별 공조 장치 제어 API를 Gateway로 캡슐화.
* 제어 이력/상태를 시계열/이벤트 DB에 기록.

### 4) 모니터링 서버

* 센서/장치/HVAC/브로커 상태 수집.
* 이상 징후·장애 탐지 및 알람 생성.
* 알람 이력/대시보드 제공, Push 시스템 연계.

---

## 2. 3-Layered 컴포넌트 & 인터페이스 설계

### A. 와인셀러 공조 시스템 서버

#### [상단] Presentation / API Layer

1. **AdminConsoleFacade**

   * Provided: `IAdminConfigService`

     * op: `manageUsers()`, `manageWarehouse()`, `manageZone()` 등.
   * Uses: `IPolicyManagement`, `IZoneProfileManagement`, `IEquipmentRegistration`.

2. **OperatorConsoleFacade**

   * Provided: `IOperatorConfigService`

     * UC 수준: 정책 조회, 일부 파라미터 변경, 리포트 조회 등.
   * Uses: `IPolicyQuery`, `IReportQuery`.

3. **ExternalSystemAPI**

   * Provided: `IWarehouseIntegrationAPI`
   * Uses: `IInventoryIntegration`.

> 상단 컴포넌트는 **오직 BL 인터페이스만 호출**. DB/외부 시스템 직접 접근 없음.

#### [중단] Business Layer

1. **PolicyManagementService**

   * Provided: `IPolicyManagement`, `IPolicyQuery`
   * Uses: `IPolicyRepositoryDAC`.

2. **ZoneProfileService**

   * Provided: `IZoneProfileManagement`
   * Uses: `IZoneRepositoryDAC`.

3. **EquipmentRegistrationService**

   * Provided: `IEquipmentRegistration`
   * Uses: `IEquipmentRepositoryDAC`.

4. **InventoryIntegrationService**

   * Provided: `IInventoryIntegration`
   * Uses: `IWarehouseSystemGateway`.

5. **ReportingService**

   * Provided: `IReportQuery`
   * Uses: `IReportViewDAC` (요약쿼리 전용 DAC).

> Business 컴포넌트는 **DAC/Gateway만 호출**, 브로커나 장치에 직접 연결 안 함. 

#### [하단] Data Access / External Interface Layer

* **PolicyRepositoryDAC** (Provided: `IPolicyRepositoryDAC`)
  CRUD 정책.

* **ZoneRepositoryDAC** (Provided: `IZoneRepositoryDAC`)

* **EquipmentRepositoryDAC** (Provided: `IEquipmentRepositoryDAC`)

* **ReportViewDAC** (Provided: `IReportViewDAC`)

* **WarehouseSystemGateway** (Provided: `IWarehouseSystemGateway`)

* **KepcoDataGateway** (필요시)

* (Push는 모니터링 서버 책임으로 두는 편이 깔끔)

> DAC는 CRUD만, Gateway는 외부 시스템 프로토콜 변환만. BL 로직 없음.

---

### B. 브로커 서버

#### [상단] Ingress / Client Interface Layer

1. **MqttIngressAdapter**

   * Provided: `IMqttMessageIngress`
   * Uses: `IEnqueueMessage`.

2. **HttpIngressAdapter**

   * Provided: `IHttpMessageIngress`
   * Uses: `IEnqueueMessage`.

3. **InternalPublisherAPI**

   * Provided: `IInternalPublish`
   * Uses: `IEnqueueMessage`.

4. **SubscriberAccessAPI**

   * Provided: `ISubscriptionQuery`, `IPullMessage`
   * Uses: `IDeliveryService` (BL).

> 상단은 프로토콜/클라이언트 의존. 메시지 내용 이해 X.

#### [중단] Broker Core Business Layer

1. **MessageRoutingService**

   * Provided: `IEnqueueMessage`, `IRouteMessage`
   * Uses: `IBrokerMetaDAC` (저장/로드), `ISubscriptionManager`.

2. **SubscriptionManager**

   * Provided: `ISubscriptionManager`
   * Uses: `IBrokerMetaDAC`.

3. **DeliveryService**

   * Provided: `IDeliveryService`

     * 구독자에게 메시지 전달, Pull/Push 추상화.
   * Uses: `IBrokerMetaDAC`.

4. **QoSRetryService**

   * Provided: `IQoSManagement`
   * Uses: `IBrokerMetaDAC`.

5. **AuthValidationService**

   * Provided: `IAuthValidation`
   * Uses: `IBrokerMetaDAC` (토큰/키 조회 등 필요한 경우).

> Core는 상단 어댑터에만 노출, 하단 DAC만 사용. 자체적으로 센서/도메인 의미 판단 안 함.

#### [하단] Data Access Layer

1. **BrokerMetaStoreDAC**

   * Provided: `IBrokerMetaDAC`
   * 책임:

     * 메시지 메타정보, 오프셋, 구독 정보, DLQ 등 CRUD.
   * External: `BrokerMetaDB`.

> 여기서 “외부 DB” = 브로커용 메타 DB.
> **환경/이벤트 시계열 DB에는 접근하지 않음** → Domain DB는 HVAC/모니터링에서만.

---

### C. HVAC 서버

#### [상단] API / Subscription Layer

1. **HvacControlAPI**

   * Provided: `IHvacControlRequest`

     * 공조 시스템 서버/관리 콘솔에서 요청하는 수동 제어, 정책 적용 명령.
   * Uses: `IHvacControlService`.

2. **BrokerCommandSubscriber**

   * Provided: `ICommandReceived`

     * 브로커가 발행한 제어 관련 메시지 수신 엔드포인트.
   * Uses: `IHvacControlService`.

3. **SensorEventSubscriber**

   * Provided: `ISensorEventReceived`

     * 브로커로부터 환경값 이벤트 수신.

> 상단은 메시지/HTTP를 받아 BL에 넘기는 어댑터.

#### [중단] Business Layer

1. **HvacControlService**

   * Provided: `IHvacControlService`
   * Uses: `IPolicyQueryDAC`, `IControlHistoryDAC`, `IVendorControlGateway`.

2. **ZoneControlOrchestrator**

   * Provided: `IZoneControl`
   * Uses: `IVendorControlGateway`.

3. **PolicyEvaluator**

   * Provided: `IPolicyEvaluation`
   * Uses: `IPolicyQueryDAC`.

> BL은 “정책 적용해 setpoint 결정 & 제어 호출”만 담당.

#### [하단] Data / External Layer

1. **PolicyQueryDAC**

   * Provided: `IPolicyQueryDAC`
   * Master DB에서 정책/존/장치 정보 조회.

2. **ControlHistoryDAC**

   * Provided: `IControlHistoryDAC`
   * 제어 이력/결과를 이벤트/시계열 DB에 기록.

3. **SamsungHvacGateway**, **LGHvacGateway**, ...

   * Provided: `IVendorControlGateway` (각 벤더별 구현)
   * 외부 공조 장치 HTTPS/클라우드 API 호출.

> HVAC BL은 브로커/장치에 직접 손 안 대고 항상 DAC/Gateway 경유.

---

### D. 모니터링 서버

#### [상단] Dashboard / Subscription Layer

1. **MonitoringDashboardAPI**

   * Provided: `IMonitoringView`
   * Uses: `IStatusQuery`, `IAlarmQuery`.

2. **BrokerEventSubscriber**

   * Provided: `IEventReceived` (센서값, 장치 상태, 브로커 상태 이벤트 수신)
   * Uses: `IEventProcessing`.

3. **HvacStatusAPIAdapter**

   * Provided: `IHvacStatusReceived` (HVAC 서버에서 상태 push 시)
   * Uses: `IEventProcessing`.

#### [중단] Business Layer

1. **EventProcessingService**

   * Provided: `IEventProcessing`
   * Uses: `IMonitoringEventDAC`, `IAlarmService`.

2. **AlarmService**

   * Provided: `IAlarmService`, `IAlarmQuery`
   * Uses: `IAlarmHistoryDAC`, `IPushGateway`.

3. **HealthEvaluationService**

   * Provided: `IStatusQuery`
   * Uses: `IMonitoringEventDAC`.

> BL은 “이상 감지 → 알람 생성 → 히스토리 관리”에 집중.

#### [하단] Data / External Layer

1. **MonitoringEventDAC**

   * Provided: `IMonitoringEventDAC`
   * 시계열/이벤트 DB에 모니터링 이벤트 저장/조회.

2. **AlarmHistoryDAC**

   * Provided: `IAlarmHistoryDAC`
   * 알람 이력 CRUD.

3. **PushGateway**

   * Provided: `IPushGateway`
   * 카카오톡/앱 Push 시스템과 연동.

> Push 호출은 BL이 아니라 Gateway가 담당, BL은 `IPushGateway`에 의존.

---

## 3. 일관성 체크 (요약)

* **Layer Skipping 없음**

  * 상단 → 중단 → 하단 단방향 의존.
  * BL이 직접 외부 시스템/DB 호출하지 않고 항상 DAC/Gateway 사용. 
* **Broker는 Message Hub로만 동작**

  * 도메인 DB에 직접 쓰지 않고, 자기 메타 DB만 DAC 통해 접근.
* **서버 간 관계**

  * 센서/장치 ↔ 브로커 ↔ (HVAC, 모니터링) 경로.
  * 공조 시스템 서버는 정책/마스터 관리와 외부 연동에 집중; 실시간 루프에 끼지 않음.
* **Interface 명명**

  * 전부 “동작(서비스) 관점” 이름 (`…Management`, `…Control`, `…Query`, `…Gateway`, `…DAC`)으로 맞춰서 평가 피드백 기준 충족.

이 구조로 Static Structure Diagram 그리면, 인터페이스/의존 방향 하나씩 따라가도 빈 구멍이나 역방향, 레이어 스킵 없이 떨어질 거야.
다음에 원하면 이 설계를 그대로 반영한 **컴포넌트 다이어그램용 PlantUML**까지 한 번에 써줄게.


