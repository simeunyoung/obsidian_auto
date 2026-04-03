## 환경

- OS: Amazon Linux 2023
- Redis 버전: redis6

---

## 1. 사전 작업 (AWS VPC 설정)

### 1-1. S3 VPC 엔드포인트 추가

AL2023 패키지 저장소가 AWS S3에 있어서 내부망 연결을 위해 필요했음.

AWS 콘솔 → VPC → Endpoints → Create endpoint

- Service: `com.amazonaws.ap-northeast-2.s3` (Gateway 타입)
- VPC: 해당 VPC 선택
- Route tables: 해당 라우팅 테이블 선택

### 1-2. DNF 리전 설정 변경

기본값이 `default`로 잘못 설정되어 있어서 서울 리전으로 변경

bash

```bash
echo "ap-northeast-2" | sudo tee /etc/dnf/vars/awsregion
```

---

## 2. Redis 설치

bash

```bash
sudo dnf install -y redis6
```

---

## 3. Redis 시작 및 자동시작 등록

bash

```bash
# 시작
sudo systemctl start redis6

# 서버 재부팅 시 자동 시작
sudo systemctl enable redis6

# 상태 확인
sudo systemctl status redis6
# Active: active (running) 확인
```

---

## 4. 비밀번호 설정

bash

```bash
# 설정 파일 열기
sudo vi /etc/redis6/redis6.conf
```

아래 항목 찾아서 수정

bash

```bash
# 수정 전
# requirepass foobared

# 수정 후
requirepass 비밀번호
```

bash

```bash
# 재시작
sudo systemctl restart redis6
```

---

## 5. 설치 확인

bash

```bash
# 접속 테스트 (AL2023은 redis6-cli 사용)
redis6-cli -a 비밀번호 ping
# PONG 나오면 성공
```

---

## 참고 - 서비스 관리 명령어

bash

```bash
# 시작
sudo systemctl start redis6

# 중지
sudo systemctl stop redis6

# 재시작
sudo systemctl restart redis6

# 상태 확인
sudo systemctl status redis6

# 로그 확인
sudo journalctl -u redis6 -f
```