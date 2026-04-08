
## 문제 원인

전 담당자에게서 인계 받은 후 모든 파일에 실행 권한 부여한 history를 발견했다.

`chmod -R 755 ./*` 실행 시 git이 파일 퍼미션 변경을 코드 수정으로 인식하여  
원격 브랜치와 충돌 발생 → `git pull` 실패

```
error: Your local changes to the following files would be overwritten by merge:
    src/main/java/.../MqttMessageHandler.java
Please commit your changes or stash them before you merge.
Aborting
```

---

## 해결 방법 비교

|방법|동작|장점|단점|
|---|---|---|---|
|`git restore .`|git 기록 기준으로 퍼미션+내용 복원|git 본연의 방법, 실행파일 퍼미션도 정확히 복원|-|
|`find . -type f -exec chmod 644 {} \;`|모든 파일을 644로 직접 변경|-|`gradlew` 등 실행파일도 644로 변경되어 `Permission denied` 오류 발생|
|`git config core.fileMode false`|git이 퍼미션 변경 자체를 무시|반복 문제 예방|근본 해결이 아닌 눈감기|

---

## ✅ 정석 해결법

```bash
git restore .   # git 기록 기준으로 퍼미션+내용 복원
git pull        # pull 진행
```

---

## `find + chmod 644` 방법의 문제점

```bash
find . -type f -exec chmod 644 {} \;
# gradlew, gradlew.bat 같은 실행파일도 644로 변경됨
# → ./gradlew build 실행 시 Permission denied 오류 발생
```

---

## `git checkout -- .` vs `git restore .`

두 명령어는 **동작이 완전히 동일**하며, `git checkout -- .`은 구버전 명령어다.

|명령어|버전|비고|
|---|---|---|
|`git checkout -- .`|git 2.23 이전|구버전|
|`git restore .`|git 2.23 이후 (2019~)|현재 권장|

### `--` 가 필요한 이유

`git checkout`이 **브랜치 전환**과 **파일 복원** 두 가지 역할을 했기 때문에 구분자 필요

```bash
git checkout develop      # 브랜치 전환
git checkout -- develop   # develop이라는 "파일"을 복원
```

### git 2.23에서 역할 분리

```bash
git switch    # 브랜치 전환 전담
git restore   # 파일 복원 전담
```

---

## 실제 코드 수정사항도 있을 경우

```bash
git stash        # 변경사항 임시 저장
git pull         # pull 진행
git stash pop    # 변경사항 다시 적용 (충돌 시 수동 해결 필요)
```

---

## 주의사항

> `chmod -R 755 ./*` 는 프로젝트 디렉토리에 절대 사용 금지  
> 필요하다면 특정 파일에만 권한 부여할 것

```bash
# ❌ 잘못된 방법
chmod -R 755 ./*

# ✅ 올바른 방법
chmod +x gradlew
```