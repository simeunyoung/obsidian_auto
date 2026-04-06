### TIER 1 — 반드시 해야 하는 것 (없으면 크롤링 자체가 안 됨)

|#|항목|이유|
|---|---|---|
|1|**`<html lang="xx">`**|검색엔진이 어떤 언어 사용자에게 노출할지 판단. 없으면 언어 타겟팅 불가|
|2|**`<meta charset="UTF-8">`**|한글 등 멀티바이트 문자 깨짐 방지. 없으면 콘텐츠 오염|
|3|**`<meta name="viewport">`**|모바일 크롤링 기준. Google은 모바일 우선 인덱싱 — 없으면 모바일 점수 0|
|4|**`<title>` 태그**|SERP에 표시되는 제목. 검색결과 클릭률(CTR) 직접 결정|
|5|**`<meta name="description">`**|SERP 설명 텍스트. CTR에 영향, 없으면 Google이 임의로 잘라서 표시|
|6|**`robots.txt`**|크롤러에게 어디를 허용/차단할지 알림. 없으면 민감한 경로도 인덱싱될 수 있음|
|7|**Sitemap (sitemap.xml)**|크롤러가 모든 페이지를 빠짐없이 발견하게 함. 없으면 신규 페이지 인덱싱 지연|
|8|**Sitemap을 robots.txt에 선언**|크롤러가 sitemap 위치를 즉시 알 수 있게 함|
|9|**HTTPS**|Google은 HTTP 사이트 랭킹 감점. 보안 신호로도 사용|
|10|**canonical URL**|중복 URL(www vs non-www, 트레일링 슬래시 등) 발생 시 대표 URL 지정. 없으면 중복 콘텐츠 패널티|

---

### TIER 2 — 해야 하는 것 (검색 결과 품질 + 소셜 공유)

|#|항목|이유|
|---|---|---|
|11|**Open Graph 태그** (og:title, og:description, og:image, og:url, og:type)|카카오톡/페이스북/슬랙 등 공유 시 미리보기 생성. 없으면 링크가 텍스트로만 표시|
|12|**Twitter Card 태그**|X(트위터) 공유 미리보기. og와 별도로 필요|
|13|**페이지별 고유 title + description**|같은 title 반복 시 Google이 중복 콘텐츠로 판단. 각 페이지마다 다른 설명 필수|
|14|**H1 태그 (페이지당 1개)**|검색엔진이 페이지의 핵심 주제를 파악하는 기준. 2개 이상이면 신호가 희석됨|
|15|**이미지 alt 텍스트**|이미지 크롤링 기준. 스크린리더 접근성 + Google 이미지 검색 노출|
|16|**favicon**|북마크, 탭, 검색결과(일부) 브랜드 인식. 없으면 기본 아이콘 표시|
|17|**Google Search Console 연동**|인덱싱 오류, 검색 키워드, 크롤링 상태 모니터링. 없으면 문제 발생해도 모름|
|18|**Analytics 설치** (GA4 등)|실제 유입 데이터 없이는 SEO 효과 측정 불가|

---

### TIER 3 — 하면 확실히 좋은 것 (SERP 향상 + 신뢰도)

|#|항목|이유|
|---|---|---|
|19|**JSON-LD 구조화 데이터 — Organization**|Google이 브랜드 정보를 Knowledge Panel에 표시. 신뢰도 신호|
|20|**JSON-LD — WebSite + SearchAction**|검색결과에 사이트 내 검색창 표시 가능 (Sitelinks Searchbox)|
|21|**JSON-LD — BreadcrumbList**|SERP에 경로 표시 (`사이트 > 블로그 > 포스트명`). CTR 향상|
|22|**JSON-LD — BlogPosting / Article**|블로그 있다면 필수. 구글 뉴스·Discover 노출 + 날짜 표시|
|23|**JSON-LD — LocalBusiness**|지역 기반 서비스라면 지도·로컬 검색에 노출. 주소, 영업시간 포함|
|24|**페이지 로딩 속도 (Core Web Vitals)**|LCP, CLS, INP 점수가 랭킹 신호. Google PageSpeed 80점 이상 목표|
|25|**이미지 최적화** (WebP/AVIF, lazy loading, 크기 명시)|LCP 점수 직접 영향. 대용량 이미지는 순위 하락 원인|
|26|**Preconnect / DNS-prefetch**|외부 리소스(GA, 폰트 등) 연결 시간 단축. 초기 렌더링 속도 향상|
|27|**Naver Search Advisor 연동**|한국 서비스라면 필수. 네이버 인덱싱은 Google과 별개|
|28|**heading 계층 구조** (H1 → H2 → H3)|콘텐츠 구조를 크롤러에게 전달. 뒤죽박죽이면 주제 파악 어려움|

---

### TIER 4 — 중장기 SEO 강화 (사이트 성장 후)

| #   | 항목                            | 이유                                              |
| --- | ----------------------------- | ----------------------------------------------- |
| 29  | **hreflang**                  | 다국어 서비스 시 국가/언어별 페이지 지정. 없으면 중복 콘텐츠 처리됨         |
| 30  | **noindex / nofollow 페이지 관리** | 검색에 노출되면 안 되는 페이지(관리자, 중복 필터 등) 명시적 제외          |
| 31  | **내부 링크 전략**                  | PageRank를 중요한 페이지에 집중. 고아 페이지(링크 없는 페이지) 제거     |
| 32  | **404 페이지 커스텀**               | 크롤러가 끊긴 링크 발견 시 사용자 이탈 방지 + 재방문 유도              |
| 33  | **OG 이미지 규격 통일**              | 1200×630px, 100KB 이하. 소셜 공유 미리보기 최적화            |
| 34  | **콘텐츠 업데이트 주기**               | Google은 최신 콘텐츠 선호. dateModified 구조화 데이터 함께 업데이트 |
| 35  | **외부 백링크 전략**                 | 외부에서 링크가 들어올수록 도메인 신뢰도(DA) 상승 — SEO 최대 변수 중 하나  |


# 이미지 형식
### 한번에 변환하는 방법

Node.js `sharp` 패키지로 스크립트 하나 만들면 폴더 전체 한번에 됩니다.

Bash Check if sharp is already installed

IN

cd s:/01.project/seo/gangalnam-web && cat package.json | grep -E '"sharp|"imagemin'

OUT

(Bash completed with no output)

`sharp` 없으니 설치 후 스크립트 실행하면 됩니다.

```bash
# 1. sharp 설치 (개발 의존성)
npm install --save-dev sharp

# 2. 변환 스크립트 실행 (아래 파일 만든 후)
node scripts/convert-images.mjs
```

스크립트 만들어드릴까요? 실행하면:

- `src/assets/images/` 안의 **PNG/JPEG/JPG 전부** 스캔
- 각각 **WebP로 변환** (품질 85)
- 원본은 건드리지 않고 **같은 폴더에 `.webp` 파일 추가**
- 변환 전후 파일 크기 콘솔 출력

변환 후에는 코드에서 import 경로만 `.webp`로 바꿔주면 됩니다. 진행할까요?