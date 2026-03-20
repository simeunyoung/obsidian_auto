[!bug] 이슈 요약 AWS IoT OTA 진행 중, IoT Rule이 트리거되지 않아 Lambda가 실행되지 않고 DB 상태값이 업데이트되지 않는 문제

---
## 📋 환경

| 항목  | 내용                                 |
| --- | ---------------------------------- |
| 서비스 | AWS IoT Core + Lambda + RDS(MySQL) |
| 리전  | ap-northeast-2 (서울)                |
| 현상  | 개발 계정 정상 / **운영 계정에서만 미작동**        |

---

## 🔍 증상

- OTA Job 자체는 디바이스에서 **성공적으로 완료**됨
- IoT Rule이 트리거되지 않아 **Lambda 미실행**
- DB의 `iot_device_firmware.update_status` 컬럼이 **업데이트되지 않음**
- CloudWatch Lambda 로그에 **아무 기록 없음**

---

## 🧩 IoT Rule SQL (정상 작동 기준)

sql

```sql
SELECT
  topic(4) AS job_id,
  topic(5) AS operation,
  thingArn AS thing_arn,
  status,
  timestamp() AS received_ts,
  timestamp AS event_ts
FROM '$aws/events/jobExecution/+/+'
WHERE status <> 'REMOVED'
```

> [!warning] 핵심 포인트 `$aws/events/jobExecution/+/+` 토픽은 AWS IoT에서 **Job Execution 이벤트를 명시적으로 활성화**해야만 메시지가 발행된다.  
> 이 설정이 꺼져 있으면 OTA가 성공해도 해당 토픽으로 **아무 메시지도 오지 않는다.**

---

## ✅ 체크한 항목들 (모두 정상이었음)

- [x]  IoT Rule 존재 여부 확인
- [x]  IoT Rule ENABLED 상태 확인
- [x]  Rule Action IAM Role - `lambda:InvokeFunction` 권한 확인
- [x]  Lambda Resource Policy - `iot.amazonaws.com` 허용 확인
- [x]  Lambda 함수 리전/계정 ARN 확인
- [x]  CloudWatch IoT 로그 레벨 설정 확인

---

## 🎯 원인

> [!danger] 근본 원인 **AWS IoT Core → Settings → Event-based messages** 에서  
> `JOB_EXECUTION` 이벤트가 **비활성화** 상태였음
> 
> 개발 계정에는 활성화되어 있었지만, 운영 계정 세팅 시 누락됨

---

## 🛠️ 해결 방법

### 방법 A: AWS 콘솔

```
IoT Core
→ Settings
→ Event-based messages
→ Manage events
→ Job Execution 관련 항목 체크 후 저장
```
