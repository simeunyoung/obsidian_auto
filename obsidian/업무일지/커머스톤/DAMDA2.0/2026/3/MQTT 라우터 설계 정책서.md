## 1. 개요

SmartThings 디바이스 동기화를 위한 MQTT 라우터 서비스 설계 정책

- MQTT 토픽 구독 → 디바이스 ID DB 확인 → 다른 서버로 전송
- 구독 토픽: `DRY/sync/iot-server/COMERSTONE-DRY-c51fdae9/execute/json`

---

## 2. 전체 아키텍처

```
MQTT Broker
    ↓
EC2 x2 (Spring Boot, 이중화) ← 공유 구독으로 로드밸런싱
    ↓
ThreadPoolTaskExecutor (내부 큐 포함)
    ↓
HikariCP (커넥션 풀)
    ↓
RDS (같은 VPC)
    ↓
다른 서버로 HTTP 전송
```

---

## 3. 인프라 스펙

### EC2

|환경|인스턴스|vCPU|RAM|구성|
|---|---|---|---|---|
|개발|t2.large|2코어|8GB|단일|
|운영|m5.xlarge|4코어|16GB|이중화 x2|

> **t2 주의사항**: CPU 버스트 크레딧 방식 → 지속 고부하 시 크레딧 소진으로 CPU 30% 제한 가능

### RDS

|환경|인스턴스|RAM|max_connections|
|---|---|---|---|
|개발|db.m5d.large|8GB|약 682개|
|운영|db.r6g.large|16GB|약 1,365개|

---

## 4. 스레드풀 설계

### I/O Bound 스레드 계산 공식

```
최적 스레드 수 = vCPU 수 × (1 + 대기시간 / CPU시간)

작업 시간 구성
  JSON 파싱   : 1ms  (CPU 사용)
  DB 조회     : 5ms  (I/O 대기, 같은 VPC)
  HTTP 전송   : 20ms (I/O 대기)
  ─────────────────────────────
  CPU 시간    : 1ms
  대기 시간   : 25ms

개발 (2코어) = 2 × (1 + 25/1) = 52개 → 50개
운영 (4코어) = 4 × (1 + 25/1) = 104개 → 100개
```

### 처리량 계산

```
스레드 1개 초당 처리량 = 1 / 0.026초 = 약 38건/초

개발 단일  : 50스레드  × 38 = 1,900건/초
운영 1대   : 100스레드 × 38 = 3,800건/초
운영 이중화 : 3,800 × 2    = 7,600건/초
```

### 스레드풀 설정값

```java
// 개발 (t2.large, 2코어)
corePoolSize  = 20
maxPoolSize   = 40
queueCapacity = 500

// 운영 (m5.xlarge, 4코어)
corePoolSize  = 50
maxPoolSize   = 80
queueCapacity = 1000
```

### 동작 방식

```
메시지 900개 유입
    ↓
corePool 스레드 즉시 처리
    ↓
남은 메시지 → 내부 큐 적재
    ↓
큐 가득 찼을 때 → maxPool까지 스레드 증가
    ↓
그래도 초과 시 → CallerRunsPolicy (자연스러운 속도 조절)
```

> **큐는 외부(SQS)가 아닌 JVM 메모리 내부 큐 사용** 서버 다운 시 큐 데이터 유실 가능하나 SmartThings 동기화 특성상 다음 메시지로 복구되므로 허용

---

## 5. HikariCP 커넥션 풀 설계

### 전체 서비스 커넥션 배분 (운영 기준)

```
db.r6g.large max_connections : 1,365개
여유분 20% 제외               : 실사용 1,092개
서비스 8개 × 이중화 2대       : 16개 풀
```

|서비스|운영 (대당)|개발 (단일)|비고|
|---|---|---|---|
|디바이스 컨트롤|100|50|핵심 서비스|
|스마트싱스|100|50|핵심 서비스|
|멤버십|80|40||
|이벤트|80|40||
|어스 DB 적재|80|40|배치성|
|Auth|80|40||
|알림|50|25||
|**MQTT 라우터**|**30**|**15**|SELECT만|
|**합계**|**600**|**300**||

```
운영 총합 600개 << 한도 1,092개 ✅
개발 총합 300개 << 한도 545개  ✅
```

### MQTT 라우터 HikariCP 설정

```yaml
# 운영
spring:
  datasource:
    hikari:
      maximum-pool-size: 30
      minimum-idle: 5
      connection-timeout: 3000
      idle-timeout: 60000

# 개발
spring:
  datasource:
    hikari:
      maximum-pool-size: 15
      minimum-idle: 3
      connection-timeout: 3000
      idle-timeout: 60000
```

---

## 6. MQTT 설정

```yaml
mqtt:
  broker-url: tcp://broker-url:1883
  client-id: dry-sync-${random.uuid}   # 인스턴스마다 고유값 필수
  topic: $share/dry-group/DRY/sync/iot-server/COMERSTONE-DRY-c51fdae9/execute/json
  qos: 1                               # 최소 1번 전달 보장
  clean-session: false                 # 재연결 시 유실 방지
  automatic-reconnect: true            # 자동 재연결
  keep-alive: 60
```

### 공유 구독 (Shared Subscription)

```
이중화 2대가 같은 토픽 구독 시 중복 처리 방지

$share/dry-group/토픽명

EC2 #1 ──▶ 메시지 A, C, E ...
EC2 #2 ──▶ 메시지 B, D, F ...
→ 브로커가 알아서 분배 (로드밸런싱) ✅
→ 중복 처리 없음 ✅
```

---

## 7. 메시지 처리 플로우

```
MQTT 메시지 수신 (메인 스레드)
    ↓
@ServiceActivator → 즉시 스레드풀에 던짐
    ↓
@Async("mqttTaskExecutor")
    ↓
try {
  1. JSON 파싱 → deviceId 추출
  2. RDS 조회 → 디바이스 존재 확인 (SELECT EXISTS)
       존재 X → return (처리 종료)
       존재 O → 다음 단계
  3. 다른 서버로 HTTP 전송 (WebClient)
} catch (Exception e) {
  log.error() → 해당 메시지만 실패
               다른 메시지 영향 없음 ✅
}
```

---

## 8. 코드 구조

### AsyncConfig

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "mqttTaskExecutor")
    public Executor mqttTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // 개발/운영 환경별 값 주입
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setThreadNamePrefix("mqtt-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
        executor.initialize();
        return executor;
    }
}
```

### MqttConfig

```java
@Configuration
public class MqttConfig {

    @Bean
    public MqttPahoClientFactory mqttClientFactory() {
        DefaultMqttPahoClientFactory factory = new DefaultMqttPahoClientFactory();
        MqttConnectOptions options = new MqttConnectOptions();
        options.setCleanSession(false);
        options.setAutomaticReconnect(true);
        options.setKeepAliveInterval(60);
        factory.setConnectionOptions(options);
        return factory;
    }

    @Bean
    public MessageChannel mqttInputChannel() {
        return new DirectChannel();
    }

    @Bean
    public MessageProducer mqttInbound() {
        MqttPahoMessageDrivenChannelAdapter adapter =
            new MqttPahoMessageDrivenChannelAdapter(
                clientId, mqttClientFactory(), topic
            );
        adapter.setQos(1);
        adapter.setOutputChannel(mqttInputChannel());
        return adapter;
    }
}
```

### MqttMessageHandler

```java
@Component
public class MqttMessageHandler {

    @ServiceActivator(inputChannel = "mqttInputChannel")
    public void handleMessage(Message<?> message) {
        // 스레드풀에 던지고 즉시 리턴
        mqttMessageService.processMessage(message.getPayload().toString());
    }
}
```

### MqttMessageService

```java
@Service
@Slf4j
public class MqttMessageService {

    @Async("mqttTaskExecutor")
    public void processMessage(String payload) {
        try {
            String deviceId = parseDeviceId(payload);
            if (deviceId == null) return;

            boolean exists = deviceRepository.existsById(deviceId);
            if (!exists) return;

            forwardService.send(payload);

        } catch (Exception e) {
            // 메시지 단위 예외 처리 → 다른 메시지 영향 없음
            log.error("메시지 처리 실패: {}", e.getMessage());
        }
    }
}
```

---

## 9. 리소스 특성 요약

```
I/O Bound 작업
  - DB 조회 대기   (5ms, 같은 VPC)
  - HTTP 전송 대기 (20ms)

CPU     : 낮음  (JSON 파싱 정도)
메모리   : 중간  (스레드풀 + 큐 유지)
네트워크 : 중요  (같은 VPC로 지연 최소화)
```

---

## 10. 개발 체크리스트

|순서|항목|내용|중요도|
|---|---|---|---|
|1|의존성 추가|spring-integration-mqtt, webflux|🔴 필수|
|2|AsyncConfig|@EnableAsync + ThreadPoolTaskExecutor|🔴 필수|
|3|MqttConfig|브로커 연결, 공유구독 토픽 설정|🔴 필수|
|4|HikariCP|application.yml 커넥션 풀 설정|🔴 필수|
|5|MessageHandler|@ServiceActivator 수신 후 즉시 비동기|🔴 필수|
|6|MessageService|@Async + try/catch 메시지 단위 처리|🔴 필수|
|7|DeviceRepository|existsById 단순 조회|🔴 필수|
|8|ForwardService|WebClient HTTP 전송|🔴 필수|
|9|client-id 고유값|${random.uuid} 설정|🔴 필수|
|10|공유구독 토픽 확인|MQTT 브로커 지원 여부 확인|🟠 중요|
|11|스레드풀 모니터링|activeCount, queue size 확인|🟡 권장|
|12|HikariCP 튜닝|운영 후 커넥션 대기시간 모니터링|🟡 권장|

---

## 11. 핵심 원칙 3가지

```
1. 메시지 독립성
   → 한 건 실패가 전체에 영향 없도록
   → 메시지 단위 try/catch 필수

2. 커넥션 재사용
   → HikariCP로 DB 커넥션 풀 관리
   → 전체 서비스 커넥션 합산 한도 초과 금지

3. 이중화 중복 방지
   → 공유 구독으로 브로커 레벨에서 분배
   → client-id 인스턴스별 고유값 필수
```