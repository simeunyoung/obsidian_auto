
### 1. 목표
- OAuth2 Authorization Server(토큰 발급/ 재발급)와 App/Web(Resource/API)를 SecurityFilterChain 기준으로 역할 분리
- `/oauth2/**` 엔드포인트를 App 체인에서 임의로 permitAll 처리하지 않도록 정리
- UserInfo를 커스텀 컨트롤러가 아닌 OIDC 표준(UserInfo endpoint)로 제공
- JWT 기반 구조에서 logout의 의미를 구분(세션 UX or 토큰 무효화)

---
### 2. 역할 정의
#### Authorization Server(어스 서버)
- 인증(로그인) 후 토큰 발급
- Authorization Code 발급
- Access Token(JWT) 발급
- Refeash Token 발급/ 저장/ 재발급 처리
- OIDC 기능(Discovery/Userinfo 등) 제공

#### Resource Server(리소스 서버)
- Access Token(JWT) 검증만 수행
- 토큰 발급/ 재발급 없음
- 일반 API(/api/**) 보호 용도

---
### 3. SecurityFilterChain 구성 전략
#### 3.1 Chain 1: Authorization Server 전용 (@Order(1))
- `OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);`
- `oidc(Customizer.withDefaults())` 활성화 -> OIDC UserInfo 제공
- 인증이 없으면 /login 으로 보내도록 EntryPoint 설정 가능

#### 3.2 Chain 2: App/Web/API 전용(@Order(2))
- `/`, `/login`, 정적 리소스(`/css/**`, `/images/**`)만 permitAll
- 그 외는 authenticated 
- `/oauth2/**` 관련 permitAll 절대 포함하지 않음
	- 이유
		- app 체인에서 `"/oauth2/**"`를 permitAll로 풀어두면,
			- 체인 매칭이 예상과 다르게 동작할 때
			- `/oauth2/token`, `/oauth2/jwks` 같은 핵심 엔드포인트가 공개되는 위험 발생
	- 결론
		- `/oauth2/**`는 Authorization Server 체인이 전담하게 하고, app 체인에서는 제거한다.
 
---

### 4. Logout 정리 (JWT vs 세션)
#### 4.1 "로그아웃 UX"가 필요한 경우
- 브라우저 기반 로그인(formLogin) 사용
- 관리자 화면/로그인 페이지가 있고, 사용자가 로그아웃 버튼을 눌러 
	- 세션을 종료하고 
	- UI가 즉시 로그아웃 상태로 바뀌는 경험이 필요할 때
#### 4.2 JWT 관점에서 logout이 의미 없는 이유
- JWT는 서버가 상태를 저장하지 않으면 만료 전까지 유효
- `/logout` 호출해도 JWT 자체는 자동으로 무효화되지 않음
#### 4.3 진짜 로그아웃(재발급 차단)이 필요하면
- Authorization Server에서 Refresh Token을 DB에 저장하고
- logout 시 refresh token 폐기 정책 등을 적용해야 함
#### 결론
- API만이면 logout은 필수 아님(클라이언트가 토큰 삭제)

---

### 5. UserInfo 리팩토링: 커스텀 컨트롤러 제거 -> OIDC 표준으로
#### 5.1 기존 구현(커스텀 `/oauth2/userinfo`)의 한계
- OIDC 표준 UserInfo가 아니라 서비스 임의 API가 됨
- 체인/ 필터 충돌 위험
- 표준 scope/claim 처리와 일관성 유지가 어려움
- 매서드 POST 사용(호환성 측면에서 GET이 일반적)
#### 5.2 리팩토링 목표
- Spring Authorization Server의 OIDC userinfo endpoint 사용
- 사용자 정보는 토큰 발급 시점에 claims 주입 방식으로 통일
#### 5.3`OAuth2TokenCustomizer<JwtEncodingContext>`로 claims 주입
- access_token/ id_token에 사용자 claims 추가
- 표준 클레임(호환성) + 필요한 커스텀 클레임 혼합 가능
예시
- `sub`: 사용자 고유 식별자(문자열 통일 권장)
- `preferred_username`, `email`, `phone_number`: 표준 클레임
- `group_id`, `withdrawal_status`: 커스텀 클레임

---

### 6. 최종 리팩토링 결정
1. app 체인에서 "`/oauth2/**`" permitAll 제거
2. `/oauth2/userinfo` 커스텀 컨트롤러 삭제
3. OIDC 활성화(`oidc()`)유지
4. TokenCustomizer로 access_token/ id_token에 필요한 claims 주입
5. logout은 refresh token revoke 및 필요 데이터 삭제 확인 후 로직 /logout api 생성