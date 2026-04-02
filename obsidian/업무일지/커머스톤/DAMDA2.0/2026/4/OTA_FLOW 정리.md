# OTA Flow 정리

## 1. 개요

현재 OTA는 **APP → DCM API → MQTT write/json → Device → AWS IoT Jobs → 상태 반영 → DB 동기화** 구조로 동작한다.

핵심은 다음과 같다.

- APP은 AWS IoT Jobs를 직접 호출하지 않는다.
- DCM은 디바이스에 `write/json` 메시지를 보내 OTA 시작 트리거만 준다.
- 실제 AWS IoT Jobs의 `start-next`, `update` 처리는 디바이스가 수행한다.
- 서버 DB의 `iot_device_firmware.update_status`는 `QUEUED / IN_PROGRESS / COMPLETED / FAILED` 상태를 가진다.

---

## 2. 주요 구성요소

### APP
- OTA 가능 여부 조회
- OTA 시작 요청

### DCM
- 디바이스/모델/펌웨어 조회
- 디바이스용 MQTT 메시지 생성
- AWS IoT Core로 `write/json` 발행

### Device
- DCM의 `write/json` 수신
- AWS IoT Jobs reserved topic 구독
- `start-next`로 작업 시작
- OTA 다운로드 / 검증 / 설치
- `jobs/{jobId}/update`로 성공/실패 보고

### AWS IoT Core / Jobs
- Job 등록 및 pending execution 관리
- `notify`, `notify-next` 알림 발행
- `start-next/accepted`, `update/accepted` 응답 처리

### DB
- `iot_model_firmware`: 모델 기준 배포 단위
- `iot_device_firmware`: 디바이스별 OTA 상태 관리

---

## 3. DB 구조

### 3.1 iot_model_firmware
모델 기준 배포 단위 테이블

주요 컬럼:
- `id`
- `device_model_id`
- `job_id`
- `version_number`
- `job_type` (`MCU` 또는 `ESP32`)
- `completed`
- `created_time`

### 3.2 iot_device_firmware
디바이스별 OTA 상태 관리 테이블

주요 컬럼:
- `id`
- `device_id`
- `model_firmware_id`
- `update_status` (`QUEUED`, `IN_PROGRESS`, `COMPLETED`, `FAILED`)
- `created_time`

---

## 4. OTA 체크 플로우

APP이 특정 디바이스에 OTA 가능한 펌웨어가 있는지 확인한다.

```text
APP
  → POST /api/1.0/device/ota/check
DCM
  → iot_device_firmware에서 device_id + job_type별 QUEUED 존재 여부 조회
APP
  ← { mcu: true/false, esp32: true/false }
```

조회 조건 개념:
- `iot_device_firmware.update_status = 'QUEUED'`
- `iot_model_firmware.job_type = 'MCU' 또는 'ESP32'`

즉, 이 API는 **AWS IoT Jobs를 직접 조회하는 것이 아니라 DB 기준으로 대기 중 OTA가 있는지 확인**하는 용도다.

---

## 5. OTA 시작 플로우

### 5.1 APP 요청

APP이 OTA 시작 API를 호출한다.

예시:

```json
{
  "account": { },
  "did": "manufacture-model-serial",
  "job_type": "MCU"
}
```

> 현재 DTO 기준 요청 필드명은 `job_type` 이다.

---

### 5.2 DCM 내부 처리

`executeOta()` 기준 처리 순서는 다음과 같다.

1. `did` 분해 (`manufacture`, `model`, `serial`)
2. `IotDeviceSerial` 조회 후 `uuid` 확보
3. `clientId` 생성
4. `IotDeviceModel` 조회
5. `job_type` 기준 활성 펌웨어 조회
6. MQTT payload 생성
7. `{deviceType}/sync/iot-server/{clientId}/write/json` 로 publish

코드상 핵심 조회:

```java
IotModelFirmware iotModelFirmware = iotModelFirmwareRepository
    .findTopByDeviceModelIdAndCompletedAndJobTypeOrderByIdDesc(
        iotDeviceModel.getId(), false, otaStartRequestDto.getJob_type())
    .orElse(null);
```

주의사항:
- 요청의 `job_type` 값이 DB 값과 일치하지 않으면 조회 실패한다.
- DB에 `ESP32`로 저장되어 있는데 요청이 `esp32`로 오면 환경에 따라 조회가 실패할 수 있으므로 서버에서 대문자 정규화가 안전하다.

권장 예:

```java
String jobType = otaStartRequestDto.getJob_type();
if (jobType != null) {
    jobType = jobType.toUpperCase();
}
```

---

### 5.3 DCM → Device MQTT 메시지

발행 토픽:

```text
{deviceType}/sync/iot-server/{clientId}/write/json
```

예시 페이로드:

```json
{
  "sid": "DCM_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "did": "acme-model01-S00001",
  "cid": "acme-model01-abc123uuid",
  "msg": {
    "o": "w",
    "e": [
      {
        "n": "/3/0/3",
        "sv": "1.2.3"
      }
    ]
  }
}
```

의미:
- `sid`: 요청 식별용 세션 ID
- `did`: 디바이스 ID
- `cid`: MQTT client ID
- `msg.o = w`: write 요청
- `msg.e[0].n = /3/0/3`: 버전 관련 리소스 URI
- `msg.e[0].sv`: 배포할 펌웨어 버전

즉, DCM은 디바이스에 **"이 버전으로 OTA 진행해라"** 라는 트리거를 주는 역할이다.

---

## 6. 디바이스 기준 AWS IoT Jobs 플로우

이제부터는 디바이스 내부 로직이다.

### 6.1 전제 상태

AWS IoT Core에 Job 생성이 끝난 상태라고 가정한다.

이 시점의 기본 상태:

```text
JobExecution = QUEUED
```

---

### 6.2 디바이스 구독 토픽

디바이스는 보통 아래 Jobs reserved topic을 구독한다.

```text
$aws/things/{thingName}/jobs/notify
$aws/things/{thingName}/jobs/notify-next
$aws/things/{thingName}/jobs/start-next/accepted
$aws/things/{thingName}/jobs/start-next/rejected
$aws/things/{thingName}/jobs/+/update/accepted
$aws/things/{thingName}/jobs/+/update/rejected
```

설명:
- `notify`: pending job 목록 변경 알림
- `notify-next`: 다음에 처리할 job 1건 알림
- `start-next/accepted`: 다음 작업 시작 요청 성공 응답
- `update/accepted`: 작업 상태 업데이트 성공 응답

실무적으로는 `notify`와 `notify-next`를 둘 다 구독하는 것이 안전하다.

---

### 6.3 Job 알림 수신

AWS가 작업을 할당하면 디바이스는 다음 알림을 받을 수 있다.

#### notify 예시

```json
{
  "jobs": {
    "QUEUED": ["job1", "job2"]
  }
}
```

의미:
- "할 일 생김"

#### notify-next 예시

```json
{
  "execution": {
    "jobId": "job1",
    "status": "QUEUED"
  }
}
```

의미:
- "다음에 처리할 작업은 이거"

---

### 6.4 디바이스가 작업 시작

디바이스는 다음 작업을 시작하기 위해 아래 topic으로 publish 한다.

```text
$aws/things/{thingName}/jobs/start-next
```

예시 payload:

```json
{
  "stepTimeoutInMinutes": 10
}
```

이 요청의 의미:
- "다음 pending 작업을 가져와서 시작하겠다"

응답 topic:

```text
$aws/things/{thingName}/jobs/start-next/accepted
$aws/things/{thingName}/jobs/start-next/rejected
```

`accepted` 예시:

```json
{
  "execution": {
    "jobId": "job1",
    "status": "IN_PROGRESS",
    "jobDocument": {
      "jobType": "MCU_OTA"
    }
  }
}
```

이 시점 상태 변화:

```text
QUEUED → IN_PROGRESS
```

중요:
- `start-next/accepted`는 **완료가 아니라 시작 성공 응답**이다.
- 완료/실패 처리는 별도의 `jobs/{jobId}/update` 로 한다.

---

### 6.5 실제 OTA 수행

디바이스는 `jobDocument`를 기반으로 아래 절차를 수행한다.

1. 작업 종류 판별
2. 다운로드 URL 또는 파일 정보 확인
3. 펌웨어 다운로드
4. 해시/서명 검증
5. 플래시 쓰기
6. 재부팅 또는 적용
7. 성공/실패 판단

---

### 6.6 완료 / 실패 상태 반영

디바이스는 최종 결과를 아래 topic으로 publish 한다.

```text
$aws/things/{thingName}/jobs/{jobId}/update
```

성공 예시:

```json
{
  "status": "SUCCEEDED"
}
```

실패 예시:

```json
{
  "status": "FAILED",
  "statusDetails": {
    "reason": "checksum error"
  }
}
```

응답 topic:

```text
$aws/things/{thingName}/jobs/{jobId}/update/accepted
$aws/things/{thingName}/jobs/{jobId}/update/rejected
```

중요:
- **완료/실패 처리의 진짜 상태 반영 API는 `jobs/{jobId}/update`** 이다.
- `start-next/accepted`로 완료 처리하는 것은 아니다.

---

## 7. 서버 DB 상태 반영

서버 DB의 상태는 보통 아래 방식으로 반영한다.

1. AWS IoT Jobs 이벤트
2. IoT Rule
3. Lambda
4. 별도 조회 로직

즉, 일반적인 구조는:

```text
Device가 AWS Jobs 상태 변경
  → AWS 이벤트 발생
  → Lambda 또는 서버 동기화 로직 실행
  → iot_device_firmware.update_status 갱신
```

예시 상태 매핑:
- AWS `QUEUED` → DB `QUEUED`
- AWS `IN_PROGRESS` → DB `IN_PROGRESS`
- AWS `SUCCEEDED` → DB `COMPLETED`
- AWS `FAILED` → DB `FAILED`

---

## 8. 전체 시퀀스

### 8.1 APP / DCM / Device / AWS / DB 전체 흐름

```text
[사전 준비]
디바이스 등록
  → 서버가 활성 펌웨어 있으면 iot_device_firmware에 QUEUED 생성

[1] APP 조회
APP
  → /api/1.0/device/ota/check
DCM
  → DB에서 MCU / ESP32 대기 여부 확인
APP
  ← {mcu, esp32}

[2] APP 시작
APP
  → /api/1.0/device/ota/start {did, job_type}
DCM
  → did 분해, serial/uuid/model/firmware 조회
  → write/json MQTT 발행
Device
  ← OTA 시작 트리거 수신

[3] 디바이스 Jobs 처리
Device
  → jobs/notify, jobs/notify-next 구독
AWS Jobs
  → 작업 알림 전송
Device
  → jobs/start-next publish
AWS Jobs
  ← start-next/accepted 응답
  → 상태: QUEUED → IN_PROGRESS

[4] OTA 수행
Device
  → 다운로드 / 검증 / 설치 / 재부팅

[5] 결과 반영
Device
  → jobs/{jobId}/update publish
AWS Jobs
  ← update/accepted 응답
  → 상태: SUCCEEDED 또는 FAILED

[6] DB 동기화
AWS 이벤트 / Lambda / 서버 조회
  → iot_device_firmware 상태 갱신
```

---

## 9. 두 개 이상의 Job이 동시에 있을 때

예: RTOS 작업 + 사용자정의 MCU 작업

주의사항:
- 디바이스는 한 번에 하나의 Job만 처리하는 것이 안전하다.
- `start-next`는 pending 작업 중 다음 것을 반환한다.
- 작업 종류는 AWS가 자동 분기해주지 않으므로 디바이스가 `jobDocument`를 보고 판단해야 한다.

권장 방식:
1. 같은 디바이스에 동시에 여러 OTA Job을 넣지 않기
2. 꼭 동시에 존재해야 하면 디바이스가 `jobDocument.jobType`으로 분기하기
3. 가능하면 하나의 상위 Job으로 통합하기

예시 `jobDocument`:

```json
{
  "jobType": "MCU_OTA",
  "fwTarget": "mcu",
  "version": "2.0.1"
}
```

또는

```json
{
  "jobType": "RTOS_OTA",
  "fwTarget": "esp32",
  "version": "1.2.3"
}
```

---

## 10. notify만 구독해도 되는가

가능은 하지만 권장되지는 않는다.

### notify만 구독하는 경우
- pending job 목록 변화는 감지 가능
- 하지만 실제로 어떤 작업을 할지 추가 조회가 필요함

### 권장 방식
- `notify`와 `notify-next`를 둘 다 구독
- 연결 직후 pending job 목록 또는 `$next`를 직접 조회

즉, 실무 권장 플로우는 다음과 같다.

```text
디바이스 연결
  → jobs/notify 구독
  → jobs/notify-next 구독
  → 연결 직후 pending job 조회 또는 next job 조회
  → 있으면 start-next 또는 직접 처리 시작
```

---

## 11. notify 메시지가 안 올 때 확인할 것

### 11.1 실제 pending job execution이 있는지
- Job이 실제 생성되었는지
- 대상 thing 또는 thing group이 맞는지
- 해당 디바이스에 execution이 `QUEUED` 상태인지

### 11.2 topic의 thingName이 정확한지
정확한 topic 예:

```text
$aws/things/{thingName}/jobs/notify
```

- clientId와 thingName 혼동 여부 확인

### 11.3 IoT Policy 권한 확인
필요 권한 예:
- `iot:Connect`
- `iot:Subscribe`
- `iot:Receive`
- `iot:Publish`

Jobs reserved topic에 대한 권한이 포함되어야 한다.

### 11.4 알림을 놓친 경우
디바이스가 offline 상태였다가 나중에 접속하면 알림을 놓칠 수 있다.
그래서 연결 직후 `GetPendingJobExecutions`, `$next`, `start-next` 등의 직접 조회/요청이 필요하다.

---

## 12. API/DTO 관련 주의사항

### 12.1 요청 필드명
현재 DTO:

```java
public class OtaStartRequestDto {
    AccountRequestDto account;
    String did;
    String job_type;
}
```

따라서 현재 JSON 요청은 다음처럼 보내야 한다.

```json
{
  "did": "manufacture-model-serial",
  "job_type": "MCU"
}
```

즉, 현재 구조에서는 **`job_type` 스네이크 케이스**가 맞다.

### 12.2 job_type 대소문자
DB 값이 `MCU`, `ESP32` 라면 요청도 동일하게 맞추는 것이 안전하다.
권장 처리:

```java
jobType = jobType.toUpperCase();
```

### 12.3 DB collation 확인
대소문자 구분 여부는 DB collation에 따라 달라질 수 있다.
확인 예:

```sql
SHOW VARIABLES LIKE 'collation%';
SHOW FULL COLUMNS FROM iot_model_firmware;
SELECT * FROM iot_model_firmware WHERE job_type = 'esp32';
```

현재 `*_ci` 계열 collation이면 일반적으로 대소문자 구분은 하지 않는다.
그래도 애플리케이션 레벨에서 정규화하는 것이 안전하다.

---

## 13. 핵심 요약

### 서버 역할
- OTA 시작 API 수신
- 활성 펌웨어 조회
- 디바이스에 `write/json` 발행

### 디바이스 역할
- `write/json` 수신
- AWS Jobs 알림 수신
- `start-next`로 실제 작업 시작
- OTA 수행
- `jobs/{jobId}/update`로 완료/실패 반영

### 중요한 구분
- `notify`: 작업 있음 알림
- `notify-next`: 다음 작업 알림
- `start-next`: 다음 작업 시작 요청
- `start-next/accepted`: 시작 성공 응답
- `jobs/{jobId}/update`: 완료/실패 처리

### 가장 중요한 한 줄

**DCM은 OTA 시작 트리거만 주고, 실제 AWS IoT Jobs lifecycle은 디바이스가 처리한다.**
