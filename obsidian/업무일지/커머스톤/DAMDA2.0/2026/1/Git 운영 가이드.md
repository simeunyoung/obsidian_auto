## 1. 목적

본 문서는 우리 팀의 **Git 브랜치 전략, 개발–배포 흐름, 작업 규칙**을 정의한다.  
모든 팀원은 본 문서를 기준으로 개발 및 배포를 수행한다.

---

## 2. 기본 전제

- **로컬 개발**: 개인 GitHub 계정 사용
- **서버 배포**: 회사 GitHub 계정 + Token 사용
- **서버는 single-branch만 pull**
- PR은 `develop` → `main`에만 생성

---

## 3. 브랜치 ↔ 배포 환경 매핑

| 브랜치       | 배포 대상   | 설명             |
| --------- | ------- | -------------- |
| `develop` | 개발 서버   | 내부 개발 및 통합 테스트 |
| `release` | 수요기업 서버 | 고객사 검증         |
| `main`    | 운영 서버   | 실제 운영 서비스      |


> 각 서버는 **자기 브랜치만** pull 한다.

---

## 4. 기본 개발 흐름

### 4.1 기능 개발

```
git checkout develop 
git fetch origin 
git pull 
git checkout -b feature/브랜치명
```

### 4.2 develop 브랜치에 머지

1. **`--no-ff` 방식으로 merge**
2. git merge --squash feature/xxx


### 4.3 작업 브랜치 삭제

- GitHub: merge 후 `Delete branch`
- 로컬:  `git branch -d feature/기능명 git fetch -p`

#### 4.4. 부분 반영이 필요한 경우 (cherry-pick)

---
## 5. 브랜치 네이밍 규칙

- **`feature/*`** : 기능 개발
- **`chore/*`** : 설정 / 정책 / 빌드 등

#### 5.1 네이밍 형식

`<type>/<short-description>`

- `type` : `feature` 또는 `chore`
- `short-description` :
    - 소문자 사용
    - 단어 구분은 하이픈(`-`)
    - 작업 내용을 한눈에 알 수 있게 작성
---
## 6. 명령어 사용 기준

| 상황          | 사용                      |
| ----------- | ----------------------- |
| 원격 상태 갱신    | `git fetch origin`      |
| 기능 반영       | `merge --no-ff`         |
| 최신화(기능 브랜치) | `rebase origin/develop` |
| 부분/긴급 반영    | `cherry-pick`           |

---
## 7. 팀 규칙 요약

> **작업 브랜치는 develop에서 생성한다.  
> `--no-ff`/squash merge로 기록을 남기고, merge 후 삭제한다.  
> 각 서버는 자기 브랜치만 single-branch로 pull한다.**
> commit mesage 규칙은 Conventional Commits Rules을 따른다.



## Commit Type Rules (Conventional)

| type         | 의미       | 사용 시점               |
| ------------ | -------- | ------------------- |
| **feat**     | Feature  | 사용자 기능 추가 / 변경      |
| **fix**      | Bug Fix  | 버그 수정               |
| **refactor** | Refactor | 동작 변화 없는 구조 개선      |
| **chore**    | Chore    | 설정 / 정책 / 빌드 / 리팩토링 |
| **docs**     | Docs     | 문서 작성 / 수정          |
| **test**     | Test     | 테스트 코드 추가 / 수정      |
