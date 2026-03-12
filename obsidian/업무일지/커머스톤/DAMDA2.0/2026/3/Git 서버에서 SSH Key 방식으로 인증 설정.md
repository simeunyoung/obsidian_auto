## 1. 개요

기존 `HTTPS + username + token` 방식 대신 `SSH Key` 방식으로 Git 저장소에 접근하면,  
`git pull`, `git clone`, `git push` 시 매번 계정과 토큰을 입력하지 않아도 된다.

운영 서버에서는 보통 이 방식을 사용한다.

---

## 2. 기본 개념

SSH 방식은 서버에서 **SSH 키 쌍**을 생성한 뒤,

- **개인키(private key)**: 서버에 보관
    
- **공개키(public key)**: GitHub에 등록
    

하는 구조이다.

이후 Git remote URL을 HTTPS가 아니라 SSH 주소로 변경하면 된다.

---

## 3. 우리 환경 기준 운영 방식

### 기준

- **SSH Key는 repo 기준이 아니라 서버 기준으로 생성**
    
- 각 서버가 필요한 repo만 접근 권한을 가지도록 설정
    

### 예시 구조

- repo: `em`, `mm`, `fpn`, `oauth`, `dcm`, `dsim`
    
- 서버:
    
    - `dcm` 서버 → `dcm` repo 사용
        
    - `dsim` 서버 → `dsim` repo 사용
        
    - `em` 서버 → `em`, `mm`, `fpn` repo 사용
        

### 권장 방식

- `dcm` 서버: SSH key 1개 생성 → `dcm` repo에 등록
    
- `dsim` 서버: SSH key 1개 생성 → `dsim` repo에 등록
    
- `em` 서버: SSH key 1개 생성 → `em`, `mm`, `fpn` repo에 등록
    

즉, **서버마다 키를 따로 만들고**, 그 서버가 접근할 repo에만 등록하는 것이 정석이다.

---

## 4. SSH Key 생성

서버에서 아래 명령 실행

ssh-keygen -t ed25519 -C "damda-msm02"

### 설명

- `-t ed25519` : 키 타입
    
- `-C "damda-msm02"` : 키 식별용 이름(comment)
    

여기서 `"damda-msm02"` 는 **우리가 직접 정하는 이름**이다.  
보통 **서버명**으로 작성한다.

예:

- `damda-msm02`
    
- `dcm01`
    
- `dsim01`
    

### 생성 결과

기본 경로에 아래 파일이 생성된다.

~/.ssh/id_ed25519  
~/.ssh/id_ed25519.pub

- `id_ed25519` : 개인키
    
- `id_ed25519.pub` : 공개키
    

---

## 5. 공개키 확인

cat ~/.ssh/id_ed25519.pub

출력된 전체 문자열을 복사한다.

예:

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA.... damda-msm02

---

## 6. GitHub에 공개키 등록

GitHub에서 해당 repo 또는 계정에 공개키를 등록한다.

### 등록 시 넣는 값

- **Title**: 서버 식별용 이름
    
- **Key**: `id_ed25519.pub` 내용 전체
    

예:

- Title: `damda-msm02`
    
- Key: `ssh-ed25519 AAAA...`
    

### 주의

우리가 이름을 직접 정하는 부분은 보통 2개다.

1. `ssh-keygen -C` 뒤의 comment
    
2. GitHub 등록 시 Title
    

둘 다 보통 서버명으로 맞춘다.

---

## 7. SSH 연결 테스트

ssh -T git@github.com

정상 시 예시 메시지:

Hi <계정명>! You've successfully authenticated

---

## 8. Git remote URL을 SSH로 변경

현재 remote 확인

git remote -v

기존이 HTTPS일 경우 예시:

origin  https://github.com/DAMDA-KEA/EM.git

SSH 방식으로 변경:

git remote set-url origin git@github.com:DAMDA-KEA/EM.git

확인:

git remote -v

결과 예시:

origin  git@github.com:DAMDA-KEA/EM.git

---

## 9. Git 명령 실행

이후부터는 username / token 없이 사용 가능

git pull origin main

또는

git clone git@github.com:DAMDA-KEA/EM.git