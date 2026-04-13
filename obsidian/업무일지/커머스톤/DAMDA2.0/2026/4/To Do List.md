---
Class: 업무일지
Project: DAMDA2.0
---
---
AWS
1. Lambda
	- [ ] 자격 증명 env 저장
	- [x] router code 전달하기 ✅ 2026-04-08

2.  LB
	- [ ] EM API 호출 내부로 변경 (NLB)
3. OTA
	- [ ] mcu 문서 업데이트
4. S3
	- [ ] S3 Lifecycle 정책(cloudfront 캐시 정책 uuid 수정 후 삭제 정책 필요)

보안 점검
- [ ] 클라우드 보안 취약 사항 SDS 정책과 검토 후 조치 예정
- [ ] 앱/웹 보안 취약 사항 검토 및 수정 (~25.12)
	- [ ] 3-2-6. Firebase API 키 재발급
1. IAM
	- [x] 개발 계정 IAM 발급해서 전달하기 ✅ 2026-04-08
2. IoTCore
	- [x] STSC 정책 발급 ✅ 2026-04-08

서버 (⭐⭐개발, 수요, 운영 확인 필수⭐⭐)
3. DCM
	- [ ] 등록 상태 저장 로직 추가
		- [ ] 가등록 완료 시 상태 변경 (P)
		- [ ] 등록 완료 시 상태 변경 (R)
		- [ ] /info/search 상태 R일 경우만 반환
	- [ ]  내부 서비스 호출 구조 Public → Internal 전환
	- [x] OTA check MCU/ESP32 구분 반환 ✅ 2026-04-08
		- [x] 개발 ✅ 2026-03-30
		- [x] 수요 ✅ 2026-04-08
		- [x] 운영 ✅ 2026-04-08

	- [ ] 민감 정보 AWS secret maneger 또는 최소 환경 변수 설정
수요 기업

---

정리할 내용
- API Gateway에 {proxy+}로 경로 설정
- 로그 찍는 방법 리팩토링
- git worktree
