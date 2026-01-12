
### User API 항목

- [x] **1. Join** (회원 가입) ✅ 2025-12-02
    
    - 인증 필요 여부: **X**
        
- [x] **2. Withdrawal** (회원 탈퇴) ✅ 2026-01-12
    
    - 인증 필요 여부: **O**
        
- [x] **3. Request TempKey** (임시 인증 번호 요청) ✅ 2025-12-02
    
    - 인증 필요 여부: **X**
        
- [x] **4. Verify TempKey** (임시인증번호 유효성 체크) ✅ 2025-12-02
    
    - 인증 필요 여부: **X**
        
- [x] **5. Find Id** (아이디 찾기) ✅ 2025-12-02
    
    - 인증 필요 여부: **X**
        
- [x] **6. Reser Password** (비밀번호 초기화) ✅ 2025-12-02
    
    - 인증 필요 여부: **X**
        
- [x] **7. Duplicate Check** (id, email, tel 중복확인) ✅ 2025-12-02
    
    - 인증 필요 여부: **X**
        
- [x] **8. Search User Info** (사용자 정보 조회) ✅ 2025-12-02
    
    - 인증 필요 여부: **O**
        
- [x] **9. Modify User Info** (사용자 정보 편집) ✅ 2025-12-03
    
    - 인증 필요 여부: **O**
        
- [x] **10. Search Event History** (이벤트 이력 조회) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **11. Update Event History** (이벤트 이력 업데이트) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        

---

### Group API 항목

- [x] **12. Update Group NickName** (그룹 닉네임 편집) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **13. Search GroupId** (그룹 아이디 조회) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **14. Send Group Invitation** (그룹 초대) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **15. Search Group Invitation** (그룹초대 조회 - 수신) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **16. Search Send Group Invitation** (그룹초대 발송 조회) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **17. Delete Group Invitation List** (그룹초대 리스트 삭제) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **18. Result Group Invitation** (그룹초대 결과 입력) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **19. Search Group Member** (그룹 구성원 조회) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **20. Withdraw Group** (그룹 탈퇴) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**
        
- [x] **21. Delete Group Member** (그룹 구성원 삭제 - 추방) ✅ 2026-01-08
    
    - 인증 필요 여부: **O**


---

### ACTION API

- [x] **1. Execute Event Action** (이벤트 액션 실행) ✅ 2026-01-09
    
    - URI: `POST /api/1.0/event/action/execute`
        
    - 인증 필요 여부: **X**
        
- [x] **2. Delete Event Action** (이벤트 액션 삭제) ✅ 2026-01-09
    
    - URI: `POST /api/1.0/event/action/delete`
        
    - 인증 필요 여부: **X**
        

---

### SCHEDULE

- [x] **3. Search Schedule Event** (예약제어 검색) ✅ 2026-01-12
    
    - URI: `POST /api/1.0/event/schedule/search`
        
    - 인증 필요 여부: UserId가 필요없음 
        
- [x] **4. Add Schedule Event** (예약제어 추가) ✅ 2026-01-12
    
    - URI: `POST /api/1.0/event/schedule/add`
        
    - 인증 필요 여부: UserId가 필요없음 
        
- [x] **5. Update Schedule Event** (예약제어 수정) ✅ 2026-01-12
    
    - URI: `POST /api/1.0/event/schedule/update`
        
    - 인증 필요 여부: UserId가 필요없음 
        
- [x] **6. Delete Schedule Event** (예약제어 삭제) ✅ 2026-01-12
    
    - URI: `POST /api/1.0/event/schedule/delete`
        
    - 인증 필요 여부: UserId가 필요없음 
        

---

### TRIGGER (RECIPE)

- [ ]  **7. Search Recipe** (자동제어 레시피 검색)
    
    - URI: `POST /api/1.0/event/recipe/search`
        
    - 인증 필요 여부: **O**
        
- [ ]  **8. Add Recipe** (자동제어 레시피 추가)
    
    - URI: `POST /api/1.0/event/recipe/add`
        
    - 인증 필요 여부: **O**
        
- [ ]  **9. Delete Recipe** (자동제어 레시피 삭제)
    
    - URI: `POST /api/1.0/event/recipe/delete`
        
    - 인증 필요 여부: **O**
        
- [ ]  **10. Update Recipe** (자동제어 레시피 수정)
    
    - URI: `POST /api/1.0/event/recipe/update`
        
    - 인증 필요 여부: **O**
        
- [ ]  **11. Search Recipe List** (자동제어 레시피 리스트 검색)
    
    - URI: `POST /api/1.0/event/recipe/search/list`
        
    - 인증 필요 여부: **O**
    
- [x] **17. Trigger Event** (자동제어 트리거 처리) ✅ 2026-01-09
	
	  - URI: `POST /api/1.0/event/trigger`
	
	  - 인증 필요 여부: X


---

### BATCHCONTROL

- [ ]  **12. Search Batch Control** (원버튼 검색)
    
    - URI: `POST /api/1.0/event/batchcontrol/search`
        
    - 인증 필요 여부: **O**
        
- [ ]  **13. Add Batch Control** (원버튼 추가)
    
    - URI: `POST /api/1.0/event/batchcontrol/add`
        
    - 인증 필요 여부: **O**
        
- [ ]  **14. Delete Batch Control** (원버튼 삭제)
    
    - URI: `POST /api/1.0/event/batchcontrol/delete`
        
    - 인증 필요 여부: **O**
        
- [ ]  **15. Update Batch Control** (원버튼 수정)
    
    - URI: `POST /api/1.0/event/batchcontrol/update`
        
    - 인증 필요 여부: **O**
        
- [ ]  **16. Search Batch Control List** (원버튼 생성 화면 리스트 조회)
    
    - URI: `POST /api/1.0/event/batchcontrol/search/list`
        
    - 인증 필요 여부: **O**