
AWS S3 + CloudFront 환경에서 타임스탬프 기반 파일명 전략 적용

## 1. 문제 상황

AWS S3에 퍼블릭 액세스 차단 정책을 적용하고 CloudFront 배포를 통해 이미지를 서빙하는 구조에서, **같은 파일명으로 이미지를 교체 업로드해도 변경된 이미지가 반영되지 않는 현상** 발생.

## 2. 해결 전략 비교

| 전략                   | 장점              | 단점                | 비용          |
| -------------------- | --------------- | ----------------- | ----------- |
| ① Cache Invalidation | 기존 URL 유지       | 수동 작업, 전파 1~5분 소요 | 월 1,000건 무료 |
| ② Cache-Control 헤더   | 자동 만료           | TTL 내 즉시 반영 불가    | 무료          |
| ③ 쿼리스트링 버전           | 즉시 반영, 구현 쉬움    | 캐시 정책 수정 필요       | 무료          |
| ④ **타임스탬프 파일명**      | 즉시 반영, 캐시 충돌 없음 | 파일명 관리 필요         | 무료          |
## 3. 채택 전략 — 타임스탬프 기반 파일명

### 선택 이유

- 기존 CloudFront 캐시 정책 변경 불필요
- Cache Invalidation 비용 및 전파 대기 시간 없음
- 새로운 URL로 요청되므로 CloudFront가 반드시 S3에서 최신 파일을 가져옴
- 기존 파일명 체계를 유지하면서 버전 구분 가능
- 서버 코드에서 타임스탬프 생성 및 관리가 단순함

### 파일명 구조

```
# 변경 전
icon/type/DRY/big_disconnect.png

# 변경 후
icon/type/DRY/big_disconnect_1709251200.png

# URL
https://dev.img.godamda.kr/icon/type/DRY/big_disconnect_1709251200.png
```

### 동작 원리

```
1. 새 이미지 준비
   └─ big_disconnect.png (신규)

2. 타임스탬프 생성 및 파일명 조합
   └─ timestamp = 1709251200
   └─ filename  = big_disconnect_1709251200.png

3. S3 업로드
   └─ s3://bucket/icon/type/DRY/big_disconnect_1709251200.png

4. DB 또는 설정에 새 파일명 업데이트
   └─ image_url = '.../big_disconnect_1709251200.png'

5. 서버 응답 → 클라이언트가 새 URL로 요청
   └─ CloudFront 캐시 미스 → S3에서 최신 이미지 반환
```

## 4. 최종 아키텍처

```
클라이언트
   │  GET .../icon/type/DRY/big_disconnect_1709251200.png
   ▼
CloudFront (Img-CachingOptimized)
   │  처음 요청 → 캐시 없음 (MISS)
   ▼
S3 (퍼블릭 액세스 차단)
   │  OAC로 CloudFront만 접근 허용
   │  파일 반환 → CloudFront 캐시 저장 (1년)
   ▼
이후 동일 URL 요청 → CloudFront 캐시 히트 ⚡

이미지 교체 시:
   타임스탬프 갱신 → 새 URL → 자동으로 새 캐시 생성
```