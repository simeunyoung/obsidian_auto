---
Class: 업무일지
Project: DAMDA2.0
---
---
AWS
1. Lambda
	- [ ] 자격 증명 env 저장
- [x] dcm dsim 로그정리 ✅ 2026-03-03
- [x] 내부 인원 외 iam 계정 삭제 ✅ 2026-03-03
1.  LB
	- [ ] EM API 호출 내부로 변경 (NLB)
보안 점검
- [ ] 클라우드 보안 취약 사항 SDS 정책과 검토 후 조치 예정
- [ ] 앱/웹 보안 취약 사항 검토 및 수정 (~25.12)
	- [ ] 3-2-6. Firebase API 키 재발급

서버 (⭐⭐개발, 수요, 운영 확인 필수⭐⭐)
1. DCM
	- [x] OTA  /check IN_PROGRESS -> QUEUED로 변경 ✅ 2026-03-17
	- [x] /validate Json Null error 내용 로그 찍고 해결 ✅ 2026-03-17
	- [x] .idea, .gradle git 추적 충돌 해결 ✅ 2026-03-10
	- [x] 서버에서 git pull 충돌 해결 ✅ 2026-03-10
	- [ ] 등록 상태 저장 로직 추가
		- [ ] 가등록 완료 시 상태 변경 (P)
		- [ ] 등록 완료 시 상태 변경 (R)
		- [ ] /info/search 상태 R일 경우만 반환
	- [ ]  내부 서비스 호출 구조 Public → Internal 전환
2. DSIM(수요기업)
	- [x] 브랜치가 아닌 커밋을 가져와서 빌드중 > 브랜치로 변경해야함 ✅ 2026-03-18
수요 기업

---

정리할 내용
- API Gateway에 {proxy+}로 경로 설정
- OAuth2.0 서버 구축 내용(JWT Key관리)
- 로그 찍는 방법 리팩토링
- git worktree
- 서버에 서비스 실행할 때 systemctl에서 관리하게 하기
- git ssh key

사용해보기
- 오픈클로
- 클로드코드