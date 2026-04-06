2026.04.06
seo최적화 1회 직후

**전체 점수: 8.2 / 10**

---

## 잘 된 것들 (완료)

### 기술 SEO

- **사이트 URL 설정** — `astro.config.mjs`에 `https://www.gangalnam.com` 명시
- **Sitemap 통합** — `@astrojs/sitemap` 설치 및 활성화
- **robots.txt** — `Allow: /`, Sitemap 경로 정상 선언
- **Google / Naver Search Console 인증 태그** — BaseLayout에 삽입 완료
- **lang="ko"** — HTML 언어 속성 정상 설정

### 메타 태그

- **title, description** — 모든 페이지 개별 설정됨
- **og:title, og:description, og:image, og:url** — Open Graph 완전 구현
- **twitter:card, twitter:title, twitter:image** — Twitter Card 완전 구현
- **canonical URL** — `SEO.astro` 컴포넌트에서 동적 생성

### 구조화 데이터 (JSON-LD)

- **Organization 스키마** — BaseLayout에 선언
- **WebSite 스키마** — BaseLayout에 선언
- **BlogPosting 스키마** — 블로그 개별 포스트마다 동적 생성 (headline, author, publisher, datePublished, image 포함)

### 콘텐츠 & 키워드

- **H1 태그** — 모든 페이지에 단 하나, 키워드 포함
- **H2/H3 계층 구조** — 논리적으로 구성됨
- **이미지 alt 텍스트** — 전부 한국어 설명 적용 (예: "해피해피 업소 내부 전경 1")
- **키워드 자연 배치** — 제목, 설명, 본문에 자연스럽게 삽입
- **블로그 포스트** — 장문 콘텐츠, 전문성 표현, 최신 날짜

### 모바일 / UX

- **반응형 디자인** — Tailwind 브레이크포인트 적용 (sm/md/lg)
- **스티키 헤더** — 모바일에서 항상 노출
- **뷰포트 메타** — 정상 설정
- **Astro Image 컴포넌트** — WebP 포맷, 품질 설정 적용

---

## 안 된 것들 (개선 필요)

### 즉시 수정 필요 (Critical)

| #   | 문제                 | 내용                                                     | 영향                                     |
| --- | ------------------ | ------------------------------------------------------ | -------------------------------------- |
| 1   | **이미지 파일 크기**      | 갤러리 이미지가 PNG/JPEG 기준 **2~2.7MB**씩 됨. OG 이미지도 **2.9MB** | Core Web Vitals (LCP 점수) 직격 → 검색 순위 하락 |
| 2   | **EmailJS 설정 미완료** | 댓글 폼의 EmailJS 키가 주석처리된 채로 있음                           | 폼 미작동 + 보안 이슈                          |

### 높은 우선순위 (1~2주 내)

|#|문제|내용|
|---|---|---|
|3|**하위 페이지 canonical 미설정**|`/info/recruit/`, `/info/gangnam-system-guide/` 페이지에서 `canonical` props를 명시적으로 넘기지 않아 BaseLayout 기본값 사용 중|
|4|**BreadcrumbList 스키마 없음**|블로그 포스트, 서브 페이지에 Breadcrumb 구조화 데이터 없음 → 검색 결과에 경로 표시 안 됨|
|5|**푸터 내부 링크 없음**|현재 푸터는 저작권만 있음. 중요 페이지 링크, 사이트맵 링크 추가 필요|

### 중간 우선순위 (1개월 내)

|#|문제|내용|
|---|---|---|
|6|**Person / LocalBusiness 스키마 없음**|"진부장" Person 스키마 + 로컬 비즈니스 스키마 추가하면 로컬 SEO 강화|
|7|**Organization 스키마에 sameAs 없음**|카카오, 텔레그램 등 소셜 계정을 `sameAs` 필드로 추가 가능|
|8|**favicon 명시적 선언 미확인**|Astro가 자동으로 처리할 수 있지만, `<link rel="icon">` 명시 권장|
|9|**preconnect 없음**|GA, KakaoTalk API 등 외부 도메인 preconnect 미적용 → 초기 로딩 미세 지연|

### 낮은 우선순위 (장기)

|#|문제|내용|
|---|---|---|
|10|**og:type 미세 조정**|블로그 포스트에서 `og:type="article"` 미적용 (현재 전부 `website`)|
|11|**twitter:site / twitter:creator 없음**|X(트위터) 계정이 없으면 불필요하지만, 있다면 추가 권장|
|12|**GA_ID 환경변수 확인**|실제 배포 환경에서 `GA_ID`가 설정되어 있는지 확인 필요|

---

## 세부 점수

|영역|점수|비고|
|---|---|---|
|기술 SEO|9/10|설정·구조화 데이터 우수|
|콘텐츠 SEO|9/10|전문성 있는 장문 콘텐츠|
|On-Page SEO|8.5/10|메타태그 우수, 일부 누락|
|모바일/UX|9/10|완전 반응형|
|성능 (Core Web Vitals)|6.5/10|이미지 최적화 미흡|
|사용자 신호|8/10|CTA 좋음, 복수 연락 수단|

---

## 우선 해결 추천 순서

1. **갤러리 이미지 & OG 이미지 압축** → WebP 변환, 500KB 이하로
2. **canonical 명시** → recruit, system-guide 페이지에 props 추가
3. **푸터 내부 링크** → 주요 페이지 링크 추가
4. **BreadcrumbList JSON-LD** → 블로그 포스트에 추가

어떤 항목부터 작업할까요?