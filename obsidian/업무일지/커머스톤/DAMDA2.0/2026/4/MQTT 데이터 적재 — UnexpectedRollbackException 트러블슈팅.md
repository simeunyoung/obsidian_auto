
## 한줄 요약

> `@Transactional` + retry 루프 공존으로 인해, 재시도 성공 후에도 트랜잭션이 커밋되지 않는 구조적 버그. 실제 데이터 유실은 없었으나 에러 로그가 지속 발생하는 잠재적 위험 코드.

---

## 🗓 발생 일시

- 발견: 2026-04-06 16:15
- 수정: 2026-04-06

---

## 🔴 에러 로그

```
INFO  ResourceHistoryService - Resource history saved successfully for DeviceId: COMERSTONE-DRY-24587CDE8334
ERROR CurrentStatusService  - Device not found for ID: COMERSTONE-DRY-24587CDE8334
WARN  MqttMessageHandler    - [RETRY] Device Not Found (1/5) backoff=1000ms
WARN  MqttMessageHandler    - [RETRY] Device Not Found (2/5) backoff=2000ms
WARN  MqttMessageHandler    - [RETRY] Device Not Found (3/5) backoff=4000ms
INFO  CurrentStatusService  - CurrentStatus updated successfully
INFO  StatusHistoryService  - StatusHistory processed successfully
INFO  MqttMessageHandler    - <<< [END] Saved (History & Status)

ERROR SimpleAsyncUncaughtExceptionHandler - Unexpected exception occurred invoking async method
org.springframework.transaction.UnexpectedRollbackException:
    Transaction silently rolled back because it has been marked as rollback-only
```

> 💡 성공 로그가 찍히고 나서 바로 롤백 에러 발생 → 처음엔 데이터 유실로 의심

---

## 🔍 원인 분석

### 구조적 원인 — @Transactional + retry 루프 충돌

```
handleAsync()                  ← @Transactional → 외부 트랜잭션 T 시작
  └─ resourceHistoryIns()      ← REQUIRED → T에 참여
  └─ currentStatusModify()     ← REQUIRED → T에 참여
       예외 발생
       → T가 rollback-only 마킹 (되돌릴 수 없음)
  └─ [retry] currentStatusModify()
       성공해도 T는 rollback-only 상태
       → 커밋 시도 → UnexpectedRollbackException
```

**핵심:** Spring의 `REQUIRED` 전파에서 내부 예외 발생 시 외부 트랜잭션이 rollback-only로 마킹되고, 이는 이후 취소 불가. retry에서 성공해도 커밋 단계에서 에러 발생.

### 실제 발생 타임라인

|시각|동작|결과|
|---|---|---|
|16:15:20|MQTT 수신, 외부 트랜잭션 T 시작|-|
|16:15:20|`resourceHistoryIns()` 실행|성공 로그 출력 (커밋 전)|
|16:15:20|`currentStatusModify()` — 기기 미등록|예외 → **T rollback-only 마킹**|
|16:15:21~23|재시도 1~3회|계속 Device not found|
|(이 사이)|다른 경로로 기기 등록 완료|-|
|16:15:27|재시도 4회 — 기기 찾음|성공 로그 출력|
|16:15:27|커밋 시도|**UnexpectedRollbackException**|

### 부가 문제 — resourceHistoryIns() 중복 호출

`resourceHistoryIns()`가 retry 루프 안에 있어서 재시도마다 매번 호출됨. → 이번 케이스 기준 **4회 중복 저장 시도** 발생 (최종 롤백으로 중복 레코드는 없었음)

---

## 🗄 DB 확인 결과

에러 발생했음에도 데이터는 정상 존재했음.

|테이블|확인 결과|비고|
|---|---|---|
|`iot_device_resource_history`|register, register-notify 2건 저장|✅ 정상|
|`iot_device_current_status`|9개 항목 저장|✅ 정상|
|`iot_device_status_history`|없음|✅ 정상 (notify 미수신)|

**데이터가 있었던 이유:** 각 서비스가 자체 `@Transactional`을 보유하고 있어 독립 커밋이 완료된 상태였음. → 실제 데이터 유실 없음. 에러 로그만 발생하는 구조적 불일치 상태.

> ⚠️ 단, 트랜잭션 전파 설정 변경이나 서비스 구조 변경 시 실제 롤백이 발생할 수 있는 잠재적 위험 구조

---

## 🛠 수정 내용

### 수정 전 코드 (문제)

java

```java
@Async("mqttTaskExecutor")
@Transactional(rollbackFor = Exception.class, timeout = 10)
public void handleAsync(MqttPayloadDto dto) {
    while (true) {
        try {
            resourceHistoryService.resourceHistoryIns(dto); // ← 루프 안 (중복 호출)
            if (!isControlTopic) {
                currentStatusService.currentStatusModify(dto);
                statusHistoryService.statusHistoryIns(dto);
            }
            break;
        } catch (InvalidBodyFormatException e) {
            if (e.getMessage().contains("Device not found") && attempt++ < maxRetries) {
                Thread.sleep(backoffMs);
            } else { break; }
        }
    }
}
```

### 수정 후 코드

java

```java
@Async("mqttTaskExecutor")
// @Transactional 제거 — retry 루프와 공존 불가
public void handleAsync(MqttPayloadDto dto) {

    // 1. 원본 로그 저장 — retry와 무관하게 1회만
    resourceHistoryService.resourceHistoryIns(dto);

    // 2. 제어 토픽이면 종료
    if (isControlTopic) {
        log.info("<<< [END] Saved (History Only) - Status Update Skipped");
        return;
    }

    // 3. CurrentStatus + StatusHistory만 retry 대상
    while (true) {
        try {
            currentStatusService.currentStatusModify(dto);
            statusHistoryService.statusHistoryIns(dto);
            log.info("<<< [END] Saved (History & Status)");
            break;
        } catch (InvalidBodyFormatException e) {
            if (e.getMessage().contains("Device not found") && attempt++ < maxRetries) {
                Thread.sleep(backoffMs);
            } else {
                log.error("[FAIL] Invalid Format. CID: {}", correlationId, e);
                break;
            }
        }
    }
}
```

### 변경 요약

|항목|수정 전|수정 후|
|---|---|---|
|`@Transactional`|`handleAsync()`에 선언|제거|
|`resourceHistoryIns()` 위치|retry 루프 안 (최대 5회)|retry 루프 밖 (1회)|
|retry 대상|전체 서비스 포함|`currentStatusModify` + `statusHistoryIns`|
|`isControlTopic` 분기|루프 안|루프 진입 전|
|재시도 성공 시 결과|`UnexpectedRollbackException`|정상 커밋|

---

## ✅ 검증 항목

- [ ]  기기 등록 직후 MQTT 수신 시나리오 — 3개 테이블 정상 저장 확인
- [ ]  `UnexpectedRollbackException` 에러 로그 미발생 확인
- [ ]  `resource_history` 중복 레코드 미생성 확인
- [ ]  Deadlock 재시도 시나리오 정상 동작 확인

---

## 💡 교훈

### @Transactional + retry 루프는 함께 쓰면 안 된다

retry 로직의 전제는 "예외가 발생해도 다시 시도할 수 있다"는 것. 하지만 `@Transactional`이 걸린 메서드 안에서 예외가 발생하면 트랜잭션이 rollback-only로 마킹되고, 이 상태는 되돌릴 수 없다.

### 해결 방법 두 가지

**방법 1 — retry 메서드에서 @Transactional 제거** (이번에 적용) 각 서비스의 개별 트랜잭션에 위임. 서비스 간 원자성이 필요 없을 때 적합.

**방법 2 — 내부 서비스에 REQUIRES_NEW 전파 레벨 사용**

java

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void currentStatusModify(MqttPayloadDto dto) { ... }
```

예외 발생 시 내부 트랜잭션만 롤백, 외부 트랜잭션은 오염되지 않음. 서비스 간 원자성 보장이 필요한 경우 사용.

### 로그를 맹신하지 말 것

성공 로그가 찍혀도 외부 트랜잭션 커밋 단계에서 실패할 수 있다. DB 직접 조회로 실제 저장 여부를 반드시 확인해야 한다.