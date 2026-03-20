
> [!info] 개요
> ESP32가 AWS IoT Custom Job을 수신하여 MCU 펌웨어를 업데이트하는 구조.
> MCU 전달 방식은 두 가지 중 확인 후 채택 예정.

---

## 전송 방식 비교 (확인 필요)

| | 방식 A - 청크 전달 | 방식 B - URL 전달 |
|---|---|---|
| 구조 | ESP32가 S3 다운로드 → UART로 MCU 전달 | JD에 URL 포함 → MCU가 직접 S3 다운로드 |
| MCU 요구사항 | 인터넷 연결 불필요 | WiFi/LTE 모듈 필요 |
| 안정성 | UART 오류 시 재전송 복잡 | HTTP 자체 재시도 가능 |
| 코드 복잡도 | ESP32/MCU 양쪽 개발 필요 | MCU만 개발 |
| 속도 | UART 속도에 제한됨 | 네트워크 속도 |
| 선호도 | MCU 인터넷 불가 시 유일한 선택 | 가능하면 이 방식이 정석 |

### 방식 A - 청크 전달 흐름

```
AWS IoT Custom Job
      ↓ MQTT
   ESP32 수신
      ↓ HTTPS
   S3에서 바이너리 다운로드
      ↓ UART (256바이트씩 청크)
   MCU 수신 → ACK → 다음 청크
      ↓
   MCU 플래시 기록 → 재부팅
```

ESP32 펌웨어에서 처리할 것:
```c
// 1. JD에서 url, size 파싱
// 2. HTTPS로 S3 바이너리 다운로드
// 3. size 비교로 다운로드 완료 확인
// 4. UART로 MCU에 256바이트씩 청크 전송
// 5. 청크마다 MCU ACK 확인
// 6. 완료 후 SUCCEEDED / FAILED 보고
```

### 방식 B - URL 전달 흐름

```
AWS IoT Custom Job
      ↓ MQTT
   ESP32 수신
      ↓ UART (URL 문자열만 전달)
   MCU 수신
      ↓ HTTPS
   MCU가 직접 S3 다운로드
      ↓
   MCU 플래시 기록 → 재부팅
      ↓ UART (완료 응답)
   ESP32 → SUCCEEDED / FAILED 보고
```

ESP32 펌웨어에서 처리할 것:
```c
// 1. JD에서 url 파싱
// 2. UART로 MCU에 URL 문자열 전달
// 3. MCU 완료 응답 대기
// 4. 완료 후 SUCCEEDED / FAILED 보고
```

> [!warning] 확인 필요
> MCU가 인터넷(HTTPS) 직접 연결 가능한지 여부에 따라 방식 결정.
> - 불가 → 방식 A (청크 전달)
> - 가능 → 방식 B (URL 전달) 권장

---

## 아키텍처

```
AWS IoT Custom Job
      ↓
  Lambda 1 (HandleJobCreate)
  → iot_model_firmware INSERT (job_type = MCU)
      ↓
  $aws/events/jobExecution (IoT Rule)
      ↓
  Lambda 2 (HandleJobStatus)
  → iot_device_firmware 상태 갱신
      ↓
  ESP32 (MQTT로 Job 수신)
  → [방식 A] S3 다운로드 → UART 청크 전송 → MCU
  → [방식 B] UART로 URL 전달 → MCU가 직접 S3 다운로드
```

---

## 기존 ESP32 OTA vs MCU Custom Job 비교

| 항목 | ESP32 OTA | MCU Custom Job |
|------|-----------|----------------|
| Job ID prefix | `AFR_OTA-` | `MCU-` |
| Job 타입 | FreeRTOS OTA | Custom Job |
| AWS Signer | 필요 | 불필요 |
| IoT Stream | 자동 생성 | 불필요 |
| 다운로드 주체 | FreeRTOS 라이브러리 | ESP32 직접 HTTPS |
| 무결성 검증 | ECDSA 서명 | CRC32 (추후 추가 예정) |
| IAM Role | OTA 전용 복잡한 Role | 기존 rule-dev-damda-ota 재사용 |

---

## Job ID 네이밍 규칙

```
MCU-{MANUFACTURER}-{MODEL}-{VERSION}

예시: MCU-COMERSTONE-DRY-v3
```

> [!info] 버전 규칙
> 버전은 `v1`, `v2`, `v3` 단순 순번으로 관리.
> 단일 숫자라 `.` 없이 Job ID 제약 조건 충족.

| 항목 | 형식 | 예시 |
|------|------|------|
| Job ID | `MCU-{제조사}-{모델}-v{N}` | `MCU-COMERSTONE-DRY-v3` |
| S3 경로 | `mcu/{제조사}/{모델}/v{N}.bin` | `mcu/comerstone/dry/v3.bin` |
| JD version | `v{N}` | `v3` |

> [!tip] Job ID 재사용 불가
> AWS IoT Job ID는 한번 사용하면 삭제해도 영구 재사용 불가.
> 재배포 시 suffix 추가: `MCU-COMERSTONE-DRY-v3-2`

---

## S3 버킷 구조

```
s3-dev-damda-ota/
├── esp32/
│   └── comerstone/
│       └── dry/
│           └── v3.bin
└── mcu/
    └── comerstone/
        └── dry/
            ├── v3.bin     ← 펌웨어 바이너리
            └── v3.json    ← Job Document
```

> [!tip] 파일명 규칙
> 빌드 결과물(`damda_esp32_project.bin`)은 업로드 시 버전명으로 rename
> ```bash
> aws s3 cp damda_esp32_project.bin s3://s3-dev-damda-ota/mcu/comerstone/dry/v3.bin
> ```

---

## Job Document (JD)

### 방식 A - 청크 전달용 JD

```json
{
  "operation": "mcu_firmware_update",
  "transfer_mode": "chunk",
  "firmware": {
    "manufacturer": "COMERSTONE",
    "model": "DRY",
    "version": "v3",
    "url": "${aws:iot:s3-presigned-url:https://s3.ap-northeast-2.amazonaws.com/s3-dev-damda-ota/mcu/comerstone/dry/v3.bin}",
    "size": 997568
  }
}
```

### 방식 B - URL 전달용 JD

```json
{
  "operation": "mcu_firmware_update",
  "transfer_mode": "url",
  "firmware": {
    "manufacturer": "COMERSTONE",
    "model": "DRY",
    "version": "v3",
    "url": "${aws:iot:s3-presigned-url:https://s3.ap-northeast-2.amazonaws.com/s3-dev-damda-ota/mcu/comerstone/dry/v3.bin}",
    "size": 997568
  }
}
```

> [!info] transfer_mode 필드
> ESP32 펌웨어에서 `transfer_mode` 값으로 청크/URL 방식을 분기 처리.
> 방식 결정 후 해당 값으로 고정하거나 필드 제거 가능.

> [!info] Presigned URL 플레이스홀더 방식 (베스트 프랙티스)
> URL을 직접 넣지 않고 `${aws:iot:s3-presigned-url:...}` 플레이스홀더 사용.
> 디바이스가 Job을 요청할 때마다 AWS가 자동으로 신선한 URL 발급 → 만료 걱정 없음.

---

## Job 생성 순서

```bash
# 1. 바이너리 S3 업로드
aws s3 cp damda_esp32_project.bin \
  s3://s3-dev-damda-ota/mcu/comerstone/dry/v3.bin \
  --region ap-northeast-2

# 2. 파일 크기 확인 → JD size 필드에 반영
ls -l damda_esp32_project.bin  # 바이트 수 확인

# 3. JD 작성 후 S3 업로드
aws s3 cp v3.json \
  s3://s3-dev-damda-ota/mcu/comerstone/dry/v3.json \
  --region ap-northeast-2

# 4. Job 생성 (CLI 권장 - presignedUrlConfig 설정 필요)
aws iot create-job \
  --job-id "MCU-COMERSTONE-DRY-v3" \
  --targets "arn:aws:iot:ap-northeast-2:554518830846:thinggroup/COMERSTONE-DRY-group" \
  --document-source "s3://s3-dev-damda-ota/mcu/comerstone/dry/v3.json" \
  --presigned-url-config '{
    "roleArn": "arn:aws:iam::554518830846:role/rule-dev-damda-ota",
    "expiresInSec": 3600
  }' \
  --target-selection SNAPSHOT \
  --region ap-northeast-2
```

> [!tip] 콘솔에서 생성 시
> - 작업 문서 → 파일에서 → S3 URL: `s3://s3-dev-damda-ota/mcu/comerstone/dry/v3.json`
> - 리소스 URL 사전 서명 구성 체크 → 역할: `rule-dev-damda-ota` → 제한 시간: 60분

---

## 전송 방식 상세

### 방식 A - 청크(Chunk) 전송

펌웨어 바이너리를 작은 조각으로 잘라서 UART로 순서대로 전송하는 방식.

```
S3 바이너리 (997KB)
      ↓ HTTPS 다운로드 (ESP32)
ESP32 메모리에 적재
      ↓ UART (256바이트씩)
   MCU 수신 → ACK
      ↓
   다음 청크 전송
      ↓ (반복)
   MCU 플래시에 기록 → 재부팅
```

UART 버퍼 제한으로 통째로 전송 불가 → 청크 단위로 나눠서 전송 필요.

### 방식 B - URL 전달

ESP32가 URL 문자열만 MCU에 전달하고, MCU가 직접 S3에서 다운로드하는 방식.

```
S3 바이너리 (997KB)
      ↓ MQTT (JD 수신)
ESP32 → URL 파싱
      ↓ UART (URL 문자열만)
   MCU 수신
      ↓ HTTPS 직접 다운로드
   MCU 플래시에 기록 → 재부팅
      ↓ UART (완료 응답)
ESP32 → SUCCEEDED / FAILED 보고
```

MCU가 HTTPS 통신 가능한 경우 ESP32 코드가 단순해지고 더 안정적.

---

## DB 스키마 변경

```sql
-- iot_model_firmware에 job_type 컬럼 추가
ALTER TABLE iot_model_firmware
ADD COLUMN job_type VARCHAR(10) NOT NULL DEFAULT 'ESP32'
COMMENT 'ESP32 | MCU'
AFTER version_number;
```

변경 후 컬럼 순서:
```
id → device_model_id → job_id → version_number → job_type → created_time → completed
```

---

## Lambda 수정 사항

### Lambda 1 (HandleJobCreate)

```python
# job_type 판별 함수 추가
def extract_job_type(job_id):
    if job_id.startswith("AFR_OTA-"):
        return "ESP32"
    elif job_id.startswith("MCU-"):
        return "MCU"
    else:
        raise ValueError(f"Unknown job_id prefix: {job_id}")

# extract_manufacture_model 수정 (MCU prefix 처리 추가)
def extract_manufacture_model(job_id):
    if job_id.startswith("AFR_OTA-"):
        job_id = job_id[len("AFR_OTA-"):]
    elif job_id.startswith("MCU-"):
        job_id = job_id[len("MCU-"):]

    parts = job_id.split("-")
    if len(parts) != 3:
        raise ValueError(f"Invalid job_id format: {job_id}")
    manufacture, model, version = parts[0], parts[1], parts[2]
    return manufacture, model, version

# INSERT 시 job_type 추가
cursor.execute("""
    INSERT INTO iot_model_firmware
        (device_model_id, job_id, version_number, job_type)
    VALUES (%s, %s, %s, %s)
""", (device_model_id, job_id, version, job_type))
```

### Lambda 2 (HandleJobStatus)

```python
# job_type 조회 함수 추가
def get_job_type(conn, job_id):
    with conn.cursor() as cursor:
        cursor.execute("""
            SELECT job_type FROM iot_model_firmware
            WHERE job_id = %s LIMIT 1
        """, (job_id,))
        row = cursor.fetchone()
    return row[0] if row else "ESP32"

# upsert 후 분기 로그 추가
upsert_device_firmware(conn, device_id, model_firmware_id, status)
conn.commit()

job_type = get_job_type(conn, job_id)
if job_type == "MCU":
    logger.info("[MCU][OK] job_id=%s device_id=%s status=%s", job_id, device_id, status)
else:
    logger.info("[ESP32][OK] job_id=%s device_id=%s status=%s", job_id, device_id, status)
```

---

## ESP32 펌웨어 수정 사항 (펌웨어 개발자 전달)

```c
// Job Document 수신 후 operation 분기 처리 필요
if (strcmp(operation, "mcu_firmware_update") == 0) {

    if (strcmp(transfer_mode, "chunk") == 0) {
        // [방식 A - 청크 전달]
        // 1. JD에서 url, size 파싱
        // 2. HTTPS로 S3에서 바이너리 다운로드
        // 3. size 비교로 다운로드 완료 확인
        // 4. UART로 MCU에 256바이트씩 청크 전송
        // 5. 청크마다 MCU ACK 확인
        // 6. 완료 후 SUCCEEDED / FAILED 보고

    } else if (strcmp(transfer_mode, "url") == 0) {
        // [방식 B - URL 전달]
        // 1. JD에서 url 파싱
        // 2. UART로 MCU에 URL 문자열 전달
        // 3. MCU 다운로드 완료 응답 대기
        // 4. 완료 후 SUCCEEDED / FAILED 보고
    }

} else {
    // ESP32 OTA 기존 로직 (AFR OTA)
}
```

---

## 전체 작업 순서

- [x] IoT Rule Job Execution 이벤트 활성화 확인
- [ ] DB ALTER TABLE 실행
- [ ] Lambda 1 수정 + 배포
- [ ] Lambda 2 수정 + 배포
- [ ] ESP32 펌웨어 수정 (펌웨어 개발자)
- [ ] 테스트용 MCU Job 생성 및 동작 확인
- [ ] 체크섬(CRC32) 추가 (추후)

---

## 성공 후 추가해야 할 것들

### 1순위 - 무결성 검증 (CRC32)

JD에 체크섬 추가. ESP32 다운로드 직후 + MCU 수신 완료 후 두 번 검증.

```json
// JD firmware 필드에 추가
"checksum": {
  "algorithm": "CRC32",
  "value": "A1B2C3D4"
}
```

```bash
# Job 생성 전 CRC32 값 계산
python3 -c "
import binascii
with open('damda_esp32_project.bin', 'rb') as f:
    data = f.read()
crc = binascii.crc32(data) & 0xFFFFFFFF
print(f'{crc:08X}')
"
```

### 2순위 - 롤백

MCU 업데이트 실패 시 이전 버전으로 자동 복구.

```json
// JD에 추가
"rollback": {
  "version": "v2",
  "url": "${aws:iot:s3-presigned-url:https://s3.ap-northeast-2.amazonaws.com/s3-dev-damda-ota/mcu/comerstone/dry/v2.bin}"
}
```

### 3순위 - 진행률 리포팅

ESP32에서 청크 전송 중 중간 상태를 JobExecution에 보고.

```json
{
  "status": "IN_PROGRESS",
  "statusDetails": {
    "progress": "45%",
    "transferred_bytes": "448512"
  }
}
```

### 4순위 - 단계적 롤아웃

디바이스가 많을 때 한번에 전체 배포 대신 분당 N대씩 순차 배포.

```bash
# Job 생성 시 옵션 추가
--job-executions-rollout-config '{
  "maximumPerMinute": 10
}'
```

### 5순위 - 자동 중단 조건

실패율이 기준 초과 시 Job 자동 취소.

```bash
# Job 생성 시 옵션 추가
--abort-config '{
  "criteriaList": [{
    "action": "CANCEL",
    "failureType": "FAILED",
    "minNumberOfExecutedThings": 10,
    "thresholdPercentage": 20
  }]
}'
# 10대 이상 실행 후 20% 이상 실패 시 자동 취소
```

---

## 관련 링크

- [[AWS IoT OTA Job Execution Event Troubleshooting]]
- [[Lambda HandleJobCreate]]
- [[Lambda HandleJobStatus]]

---

## 태그

#AWS #IoT #OTA #MCU #ESP32 #CustomJob #펌웨어업데이트
