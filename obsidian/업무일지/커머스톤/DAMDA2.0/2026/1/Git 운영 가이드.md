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


> 각 서버는 **자기 브랜치만** pull 한다.

---

## 4. 기본 개발 흐름 (정석)

### 4.1 기능 개발

`git checkout develop
git fetch origin 
git pull 
git checkout -b feature/기능명`

### 4.2 develop 브랜치에 머지

- **`--no-ff` 방식으로 merge**
    
- merge commit으로 기능 단위 기록을 남긴다
    

### 4.3 feature 브랜치 삭제

- GitHub: merge 후 `Delete branch`
    
- 로컬:  `git branch -d feature/기능명 git fetch -p`

#### 4.4. 부분 반영이 필요한 경우 (cherry-pick)
---
## 8. 명령어 사용 기준

| 상황          | 사용                      |
| ----------- | ----------------------- |
| 원격 상태 갱신    | `git fetch origin`      |
| 기능 반영       | `merge --no-ff`         |
| 최신화(기능 브랜치) | `rebase origin/develop` |
| 부분/긴급 반영    | `cherry-pick`           |

---
## 10. 팀 규칙 요약

> **feature 브랜치는 develop에서 생성한다.  
> `--no-ff` merge로 기록을 남기고, merge 후 삭제한다.  
> 각 서버는 자기 브랜치만 single-branch로 pull한다.**
> commit mesage 규칙은 Verb-based Commit Message Rules을 따른다.


#### Verb-based Commit Message Rules

| Verb         | 의미            | 사용 시점            |
| ------------ | ------------- | ---------------- |
| **add**      | 신규 기능 / 신규 코드 | 기능 처음 구현         |
| **update**   | 기능 변경 / 개선    | 기존 로직 수정         |
| **delete**   | 기능/코드 제거      | 더 이상 안 쓰는 코드     |
| **fix**      | 버그 수정         | 의도와 다른 동작        |
| **refactor** | 구조 개선         | 동작 변화 없음         |
| **config**   | 설정 변경         | yml, gradle, env |
| **docs**     | 문서            | README, 주석       |
| **test**     | 테스트           | 테스트 코드           |
