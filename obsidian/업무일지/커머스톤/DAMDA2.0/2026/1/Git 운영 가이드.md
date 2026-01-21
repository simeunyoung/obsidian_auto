## 1. 목적

본 문서는 우리 팀의 **Git 브랜치 전략, 개발–배포 흐름, 작업 규칙**을 정의한다.  
모든 팀원은 본 문서를 기준으로 개발 및 배포를 수행한다.

---

## 2. 기본 전제

- **로컬 개발**: 개인 GitHub 계정 사용
    
- **서버 배포**: 회사 GitHub 계정 + Token 사용
    
- **서버는 single-branch만 pull**
    
- main에 Merge는 PR 사용

---

## 3. 브랜치 ↔ 배포 환경 매핑

| 브랜치         | 배포 대상   | 설명             |
| ----------- | ------- | -------------- |
| `develop`   | 개발 서버   | 내부 개발 및 통합 테스트 |
| `release`   | 수요기업 서버 | 고객사 검증         |
| `main`      | 운영 서버   | 실제 운영 서비스      |
| `feature/*` | 배포 ❌    | 기능 개발용 임시 브랜치  |

> 각 서버는 **자기 브랜치만** pull 한다.

---

## 4. 기본 개발 흐름 (정석)

### 4.1 기능 개발 시작

`git checkout develop
git fetch origin 
git pull 
git checkout -b feature/기능명`

### 4.2 개발 및 커밋

`git add . git commit -m "feat: 기능 설명"`

### 4.3 원격 저장소에 push

`git push -u origin feature/기능명`

### 4.4 Pull Request 생성

- 대상: `feature/*` → `develop`
    
- 코드 리뷰 및 CI 통과 필수
    

### 4.5 develop 브랜치에 머지

- **`--no-ff` 방식으로 merge**
    
- merge commit으로 기능 단위 기록을 남긴다
    

### 4.6 feature 브랜치 삭제

- GitHub: PR merge 후 `Delete branch`
    
- 로컬:
    

`git branch -d feature/기능명 git fetch -p`

---

## 5. 기록 관리 원칙

- **기능 기록**: Pull Request
    
- **반영 기록**: merge commit (`--no-ff`)
    
- feature 브랜치를 삭제해도:
    
    - 어떤 기능이
        
    - 언제
        
    - 누가
        
    - 왜 들어갔는지  
        → **100% 추적 가능**
        

---

## 6. 여러 기능 동시 개발

### 6.1 기능이 서로 독립적인 경우

- 모두 `develop`에서 각각 feature 브랜치 생성
    
- PR도 각각 생성
    

### 6.2 기능 간 의존성이 있는 경우

- 선행 기능을 먼저 `develop`에 merge
    
- 후행 feature 브랜치는 develop 최신 반영
    

`git checkout feature/후행기능 git fetch origin git rebase origin/develop git push --force-with-lease`

> rebase는 **혼자 사용하는 feature 브랜치에서만 허용**

---

## 7. 배포 흐름

### 7.1 개발 서버 배포

- `develop` 브랜치 기준
    
- 서버는 `develop`만 pull
    

---

### 7.2 수요기업 서버 배포 (릴리즈)

`git checkout release git fetch origin git merge --no-ff origin/develop git push origin release`

- 서버는 `release`만 pull
    

#### 부분 반영이 필요한 경우 (cherry-pick)

`git checkout release git cherry-pick <commit_sha> git push origin release`

> cherry-pick도 PR/merge commit으로 기록을 남긴다

---

### 7.3 운영 서버 배포

`git checkout main git fetch origin git merge --no-ff origin/release git push origin main`

- 서버는 `main`만 pull
    

---

## 8. 명령어 사용 기준

|상황|사용|
|---|---|
|원격 상태 갱신|`git fetch origin`|
|기능 반영|`merge --no-ff`|
|최신화(기능 브랜치)|`rebase origin/develop`|
|부분/긴급 반영|`cherry-pick`|
|서버 배포|merge만 사용|

---

## 9. 금지 사항

❌ feature 브랜치를 서버에서 pull  
❌ develop → 운영 서버 직접 배포  
❌ 서버에서 커밋/브랜치 생성  
❌ main / release 브랜치 직접 push  
❌ 공용 브랜치에서 rebase

---

## 10. 팀 규칙 요약

> **feature 브랜치는 develop에서 생성한다.  
> PR + `--no-ff` merge로 기록을 남기고, merge 후 삭제한다.  
> develop → release → main 순으로 단계별 배포한다.  
> 각 서버는 자기 브랜치만 single-branch로 pull한다.**