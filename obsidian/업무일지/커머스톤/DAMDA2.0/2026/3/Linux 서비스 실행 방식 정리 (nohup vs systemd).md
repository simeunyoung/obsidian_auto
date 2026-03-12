
Linux 서버에서 애플리케이션을 실행하는 방법은 여러 가지가 있지만, 대표적으로 다음 두 가지가 있다.

- **nohup 방식**
    
- **systemd(systemctl) 방식**
    

두 방식 모두 **프로세스를 백그라운드에서 실행**한다는 공통점이 있지만,  
**프로세스를 누가 관리하느냐**에서 큰 차이가 있다.

|방식|관리 주체|
|---|---|
|nohup|Shell (사용자 스크립트)|
|systemd|OS (Linux 서비스 관리자)|

운영 서버에서는 일반적으로 **systemd 방식이 표준에 가깝다.**

# 2. nohup

## 개념

`nohup`은 **터미널 세션이 종료되어도 프로세스가 종료되지 않도록 실행하는 명령어**이다.

즉 SSH 세션이 끊겨도 프로그램이 계속 실행되도록 한다.

### 예시

nohup java -jar app.jar &

의미

- `nohup` → 터미널 종료 시에도 프로세스 유지
    
- `&` → 백그라운드 실행
    

실행 구조

shell  
 └─ nohup  
      └─ java

---

## 특징

- 백그라운드 실행 가능
    
- 터미널 종료 시에도 프로세스 유지
    
- 간단하게 사용할 수 있음
    

하지만 **프로세스 관리 기능은 거의 없음**

---

## 프로세스 관리 방식

### 실행

nohup java -jar app.jar &  
echo $! > app.pid

### 상태 확인

ps -ef | grep java

### 종료

kill $(cat app.pid)

또는

kill -9 $(cat app.pid)

---

## 단점

1️⃣ 프로세스 상태 관리 어려움  
2️⃣ 자동 재시작 없음  
3️⃣ 서버 reboot 시 자동 실행 없음  
4️⃣ 로그 관리 직접 필요  
5️⃣ 종료 시 graceful shutdown 보장 어려움

그래서 **운영 환경에서는 관리성이 떨어진다.**

---

# 3. systemd

## 개념

`systemd`는 **Linux의 서비스 관리 시스템**이다.

OS가 직접 서비스 프로세스를 관리한다.

서비스는 보통 다음 경로에 정의된다.

/etc/systemd/system/*.service

예

/etc/systemd/system/em.service

---

## service 파일 예시

[Unit]  
Description=EM Spring Boot Service  
After=network.target  
  
[Service]  
User=damda  
WorkingDirectory=/app/em  
ExecStart=/usr/bin/java -jar /app/em/EM.jar --spring.profiles.active=qa  
Restart=always  
RestartSec=5  
SuccessExitStatus=143  
  
[Install]  
WantedBy=multi-user.target

---

## 주요 항목 설명

|항목|의미|
|---|---|
|Description|서비스 설명|
|After|어떤 서비스 이후 실행할지|
|User|실행 사용자|
|WorkingDirectory|실행 디렉토리|
|ExecStart|서비스 실행 명령|
|Restart|프로세스 종료 시 재시작 여부|
|RestartSec|재시작 대기 시간|
|WantedBy|부팅 시 자동 실행 대상|

---

## 실행 구조

systemd  
 └─ java

systemd가 **프로세스를 직접 관리한다.**

---

## 주요 명령어

### 서비스 시작

systemctl start em

### 서비스 종료

systemctl stop em

### 서비스 재시작

systemctl restart em

### 상태 확인

systemctl status em

### 서버 부팅 시 자동 실행

systemctl enable em

### 로그 확인

journalctl -u em

---

# 4. SIGTERM / SIGKILL

Linux에서 프로세스를 종료할 때는 **signal**을 보낸다.

|signal|의미|
|---|---|
|SIGTERM|정상 종료 요청|
|SIGKILL|강제 종료|

### SIGTERM

kill PID

또는

systemctl stop service

프로그램에게

> "정상적으로 종료해라"

라고 요청한다.

Spring Boot에서는 다음 과정이 실행된다.

SIGTERM  
 ↓  
새 요청 차단  
 ↓  
현재 요청 처리 완료  
 ↓  
DB connection 정리  
 ↓  
Spring context 종료  
 ↓  
JVM 종료

이것을 **graceful shutdown**이라고 한다.

---

### SIGKILL

kill -9 PID

즉시 프로세스를 종료한다.

특징

- JVM shutdown 실행 안됨
    
- connection pool close 안됨
    
- shutdown hook 실행 안됨
    

그래서 운영에서는 **가능하면 사용하지 않는다.**

---

# 5. Graceful Shutdown

Graceful shutdown은 **프로그램이 안전하게 종료하도록 시간을 주는 방식**이다.

예

SIGTERM  
 ↓  
새 요청 차단  
 ↓  
현재 요청 처리 완료  
 ↓  
리소스 정리  
 ↓  
프로세스 종료

반대로 `kill -9`는

프로세스 즉시 종료

라서 다음 문제가 발생할 수 있다.

- 요청 처리 중단
    
- 로그 flush 안됨
    
- connection pool 정리 안됨
    

---

# 6. nohup vs systemd 비교

|항목|nohup|systemd|
|---|---|---|
|실행 방식|shell 실행|OS 서비스|
|프로세스 관리|직접 관리|systemd 관리|
|상태 확인|ps 명령|systemctl status|
|종료|kill|systemctl stop|
|자동 재시작|없음|가능|
|서버 reboot 자동 실행|없음|가능|
|graceful shutdown|직접 처리|기본 지원|
|운영 환경 적합성|낮음|높음|
