 
> IoT 디바이스와 모바일 앱을 연결하는 AWS IoT Core 기반 디바이스 제어 관리 서버  
  
---  
  
## 기술 스택  
  
| 분류 | 기술 |  
|------|------|  
| Language | Java 17 |  
| Framework | Spring Boot 3.2.5 |  
| ORM | Spring Data JPA, QueryDSL 5.0.0 |  
| HTTP Client | Spring WebFlux (WebClient) |  
| Database | MariaDB (AWS RDS), HikariCP |  
| IoT | AWS IoT Core SDK 2.22.0, Eclipse Paho MQTT 1.2.5 |  
| AWS | IoT Core, S3, STS, Signer |  
| Security | Bouncy Castle 1.70 (X.509 인증서, PEM) |  
| Docs | SpringDoc OpenAPI 2.0.2 (Swagger) |  
| Etc | Spring Retry, Lombok, Jackson |  
  
---  
  
## 프로젝트 구조 특징  
  
### 이중 데이터베이스 아키텍처  
비즈니스 데이터와 IoT 텔레메트리 데이터를 물리적으로 분리 관리  
  
- **IoT DB** — 센서 상태, 제어 이력, 상태 이력, MQTT 프로파일, 펌웨어 정보  
- **SVC DB** — 사용자, 그룹, 디바이스 메타데이터, 서비스 관계 데이터  
- `ChainedTransactionManager`로 두 DB에 걸친 트랜잭션 원자성 보장  
  
### 레이어 구조  
  
```  
Controller  
    └── DeviceServiceImpl (오케스트레이터)  
            ├── SvcDeviceServiceImpl    → SVC DB            ├── IotDeviceServiceImpl    → IoT DB            └── IotCoreDeviceServiceImpl → AWS IoT Core                    ├── ThingServiceImpl       (Thing 생성/삭제)  
                    ├── CertificateServiceImpl (인증서 발급/삭제)  
                    ├── PolicyServiceImpl      (정책 관리)  
                    └── MqttServiceImpl        (MQTT 통신)  
```  
  
---  
  
## 구현 기능  
  
### 디바이스 생명주기 관리  
  
#### 등록 프로세스 (3단계 원자적 처리)  
1. **SVC DB** — 사용자/그룹 관계 메타데이터 저장  
2. **AWS IoT Core** — Thing 생성 + 인증서 발급 + 정책 연결 + Thing Group 추가  
3. **IoT DB** — 디바이스 가등록 기록 저장  
- 중간 실패 시 보상 트랜잭션으로 AWS 리소스 자동 롤백 (Spring Retry 3회)  
  
#### 등록 확정 (Validate)  
- 디바이스 부팅 후 MQTT로 전달된 Register/Notify 메시지와 DB 데이터 정합성 검증  
- LWM2M 리소스 URI 검증으로 최종 등록 확정  
  
#### 삭제  
- AWS IoT Core Thing, Certificate, Policy 정리  
- SVC/IoT DB 데이터 삭제  
- MQTT로 디바이스에 삭제 명령 전송  
  
### 디바이스 제어 (Execute)  
- 사용자/그룹 권한 검증  
- LWM2M 형식 JSON 메시지 구성  
- MQTT Publish로 제어 명령 전달  
- `IotDeviceControlStatus`에 제어 이력 기록  
  
### 디바이스 상태 모니터링  
- MQTT Subscribe로 센서 데이터 수신 → `IotDeviceCurrentStatus` 실시간 갱신  
- `IotDeviceStatusHistory`에 시계열 이력 저장  
- 차트 API로 앱 대시보드용 시계열 데이터 제공  
  
### OTA 펌웨어 업데이트  
- MCU / ESP32 펌웨어 업데이트 여부 각각 확인 (`job_type` 기반 구분)  
- 최신 펌웨어 버전을 MQTT로 디바이스에 전달 (LWM2M `/3/0/3` 리소스)  
- 디바이스가 수신 후 자체 다운로드 및 설치  
  
### AWS IoT Core 리소스 관리 (Portal)  
- Thing Type 생성/삭제  
- Thing Group 계층 구조 생성/삭제/수정 (제조사 → 모델 → 디바이스)  
  
---  
  
## API 목록  
  
### Device API (`/api/1.0/device`)  
  
| Method | Endpoint | 설명 | 방향 |  
|--------|----------|------|------|  
| POST | `/register` | 디바이스 가등록 | App → DCM |  
| POST | `/http/register` | HTTP 디바이스 등록 | App → DCM |  
| POST | `/register/check` | 등록 상태 확인 | Device → DCM |  
| POST | `/validate` | 등록 확정 (URI 검증) | Device → DCM |  
| POST | `/register/cancel` | 가등록 취소 | App → DCM |  
| POST | `/deregister` | MQTT 디바이스 삭제 | App → DCM |  
| POST | `/http/deregister` | HTTP 디바이스 삭제 | App → DCM |  
| POST | `/execute` | 디바이스 제어 명령 | App → DCM → Device |  
| POST | `/info/search` | 디바이스 목록 조회 | App → DCM |  
| POST | `/info/search/detail` | 디바이스 상세 조회 | App → DCM |  
| POST | `/info/type` | 디바이스 타입 목록 | App → DCM |  
| POST | `/info/model` | 디바이스 모델 목록 | App → DCM |  
| POST | `/modify` | 디바이스 이름 수정 | App → DCM |  
| POST | `/chart/day` | 시계열 센서 데이터 | App → DCM |  
| GET  | `/getDeviceModel` | 블루투스 모델 목록 | App → DCM |  
  
### OTA API (`/api/1.0/device/ota`)  
  
| Method | Endpoint | 설명 | 방향 |  
|--------|----------|------|------|  
| POST | `/check` | MCU/ESP32 업데이트 여부 확인 | App → DCM |  
| POST | `/start` | 펌웨어 업데이트 시작 | App → DCM → Device |  
  
**OTA Check 응답 형태**  
```json  
{  
  "mcu": true,  "esp32": false}  
```  
  
### Portal API (`/api/1.0/device/thing`)  
  
| Method | Endpoint | 설명 |  
|--------|----------|------|  
| POST | `/type/put` | Thing Type 생성 |  
| POST | `/type/delete` | Thing Type 삭제 |  
| POST | `/group/put` | Thing Group 생성 |  
| POST | `/group/delete` | Thing Group 삭제 |  
| POST | `/group/rename` | Thing Group 수정 |  
  
---  
  
## AWS IoT Core 연동 상세  
  
### MQTT 통신 방식  
- 프로토콜: MQTT over SSL/TLS (포트 8883)  
- 클라이언트: Eclipse Paho (일회성 연결 — 연결 → 작업 → 해제)  
- QoS Level: 1 (최소 한 번 전달 보장)  
- 응답 대기: `CountDownLatch` (Timeout 10초)  
  
### 토픽 구조  
```  
디바이스 → 서버:  {DeviceType}/sync/{DeviceId}/iot-server/{MessageType}  
서버 → 디바이스:  {DeviceType}/sync/iot-server/{ClientId}/write/json  
```  
  
### 인증 방식  
- X.509 클라이언트 인증서 기반 상호 TLS 인증  
- Bouncy Castle으로 PEM 형식 인증서 및 RSA 키 파싱  
- 디바이스 등록 시 Certificate + Private Key 발급하여 디바이스에 전달  
  
### 다중 환경 AWS 계정 분리  
  
| 환경 | AWS 리전 | IoT 엔드포인트 |  
|------|----------|--------------|  
| dev  | ap-northeast-2 | a3rjr8miulwgyq-ats.iot.ap-northeast-2.amazonaws.com |  
| qa   | ap-northeast-2 | a3au0rqquakjfz-ats.iot.ap-northeast-2.amazonaws.com |  
| prod | ap-northeast-2 | a1ftgront5tqt3-ats.iot.ap-northeast-2.amazonaws.com |  
  
---  
  
## 주요 설계 포인트  
  
### 보상 트랜잭션 (Saga 패턴)  
AWS IoT Core는 JPA 트랜잭션 범위 밖이므로, 등록 중 실패 시 수동 롤백 처리  
```  
SVC DB 저장 → AWS Thing 생성 → AWS 인증서 발급 → IoT DB 저장  
                    ↓ 실패  
       rollbackRegistration() — Thing/Certificate/Policy 삭제 (Retry 3회)  
```  
  
### LWM2M 프로토콜 기반 리소스 모델  
디바이스 제어 및 센서 데이터를 LWM2M Object/Instance/Resource URI 체계로 관리  
```  
/3/0/0  → Manufacturer  
/3/0/1  → Model Number  
/3/0/3  → Firmware Version  
/3/0/N  → 커스텀 센서값 (온도, 습도 등)  
```  
  
### 디바이스 ID 구조  
```  
{Manufacturer}-{Model}-{Serial}  
예: LG-FREEZER-SN001234  
```  
  
---  
  
## 운영 환경  
  
- **서버 포트**: 9100  
- **배포 경로**: `/app/DCM/`  
- **로그 경로**: `/app/DCM/logs/` (로컬: `./logs/`)  
- **로그 롤링**: 50MB, gzip 압축  
- **DB**: AWS RDS MariaDB (ap-northeast-2)  
- **프로파일**: `local` / `dev` / `qa` / `prod`